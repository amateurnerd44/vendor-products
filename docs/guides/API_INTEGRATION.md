# API Integration Guide
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026

---

## Overview

This guide covers setup and configuration for all external APIs used in the Vendor Product Data Collection System.

### APIs Covered
1. Google Drive API (Tier 1)
2. Dropbox API (Tier 1)
3. Google Sheets API (Tier 1)
4. Shopify API (Tier 2)
5. Zyte API (Tier 3)
6. Google AI Studio - Gemini API (Tier 3)

---

## Google Drive API Setup

### Purpose
Access vendor product data files stored in Google Drive folders.

### Prerequisites
- Google Cloud Platform account
- Project with Drive API enabled
- OAuth 2.0 credentials

### Setup Steps

**1. Create GCP Project**
```
1. Go to https://console.cloud.google.com
2. Click "New Project"
3. Name: "Vendor Product Sync"
4. Click "Create"
```

**2. Enable Drive API**
```
1. Navigate to "APIs & Services" → "Library"
2. Search for "Google Drive API"
3. Click "Enable"
```

**3. Create OAuth 2.0 Credentials**
```
1. Go to "APIs & Services" → "Credentials"
2. Click "Create Credentials" → "OAuth client ID"
3. Application type: "Web application"
4. Authorized redirect URIs: https://[n8n-instance].n8n.cloud/rest/oauth2-credential/callback
5. Click "Create"
6. Save Client ID and Client Secret
```

**4. Configure in n8n**
```
1. Open n8n Cloud dashboard
2. Go to "Credentials" → "New"
3. Select "Google Drive OAuth2 API"
4. Enter Client ID and Client Secret
5. Click "Connect my account"
6. Authorize access
7. Save credential as "Google Drive - Vendor Sync"
```

### Usage in Workflows

**Tier 1 Scraper Node:**
```json
{
  "node": "Google Drive",
  "operation": "Download",
  "resource": "File",
  "fileId": "{{$json.drive_folder_id}}",
  "credential": "Google Drive - Vendor Sync"
}
```

### Rate Limits
- **Queries per day:** 1,000,000,000
- **Queries per 100 seconds per user:** 1,000
- **Recommendation:** No throttling needed for 70 vendors

### Troubleshooting

**Error: "Invalid credentials"**
- Solution: Re-authenticate OAuth connection in n8n

**Error: "File not found"**
- Solution: Verify folder ID in vendor_config.source_url

**Error: "Insufficient permissions"**
- Solution: Ensure Drive API has "drive.readonly" scope

---

## Dropbox API Setup

### Purpose
Access vendor product data files stored in Dropbox folders.

### Prerequisites
- Dropbox account
- Dropbox App created
- Access token

### Setup Steps

**1. Create Dropbox App**
```
1. Go to https://www.dropbox.com/developers/apps
2. Click "Create app"
3. Choose "Scoped access"
4. Choose "Full Dropbox" access
5. Name: "Vendor Product Sync"
6. Click "Create app"
```

**2. Configure Permissions**
```
1. Go to "Permissions" tab
2. Enable:
   - files.metadata.read
   - files.content.read
   - sharing.read
3. Click "Submit"
```

**3. Generate Access Token**
```
1. Go to "Settings" tab
2. Scroll to "OAuth 2"
3. Click "Generate access token"
4. Copy token (starts with "sl.")
5. Store securely
```

**4. Configure in n8n**
```
1. Open n8n Cloud dashboard
2. Go to "Credentials" → "New"
3. Select "Dropbox OAuth2 API"
4. Enter Access Token
5. Save credential as "Dropbox - Vendor Sync"
```

### Usage in Workflows

**Tier 1 Scraper Node:**
```json
{
  "node": "Dropbox",
  "operation": "Download",
  "resource": "File",
  "path": "{{$json.source_url}}",
  "credential": "Dropbox - Vendor Sync"
}
```

### Rate Limits
- **Requests per app:** Unlimited (with throttling)
- **Recommendation:** Add 100ms delay between requests

### Troubleshooting

**Error: "Invalid access token"**
- Solution: Regenerate access token in Dropbox App Console

**Error: "Path not found"**
- Solution: Verify path in vendor_config.source_url

---

## Google Sheets API Setup

### Purpose
Read vendor product data from Google Sheets.

### Prerequisites
- Google Cloud Platform account
- Project with Sheets API enabled
- OAuth 2.0 credentials (same as Drive API)

### Setup Steps

**1. Enable Sheets API**
```
1. Go to https://console.cloud.google.com
2. Select your project (or use same as Drive API)
3. Navigate to "APIs & Services" → "Library"
4. Search for "Google Sheets API"
5. Click "Enable"
```

**2. Use Existing OAuth Credentials**
```
- If you already set up Drive API, use the same OAuth credentials
- Sheets API shares the same OAuth scope
```

**3. Configure in n8n**
```
1. Open n8n Cloud dashboard
2. Go to "Credentials" → "New"
3. Select "Google Sheets OAuth2 API"
4. Enter Client ID and Client Secret (same as Drive)
5. Click "Connect my account"
6. Authorize access
7. Save credential as "Google Sheets - Vendor Sync"
```

### Usage in Workflows

**Tier 1 Scraper Node:**
```json
{
  "node": "Google Sheets",
  "operation": "Read",
  "resource": "Sheet",
  "sheetId": "{{$json.source_url}}",
  "range": "A1:Z1000",
  "credential": "Google Sheets - Vendor Sync"
}
```

### Rate Limits
- **Read requests per minute per user:** 60
- **Recommendation:** Add 1-second delay between sheet reads

### Troubleshooting

**Error: "Unable to parse range"**
- Solution: Verify sheet name and range format

**Error: "Requested entity was not found"**
- Solution: Ensure sheet is shared with OAuth account

---

## Shopify API Setup

### Purpose
Fetch product data from Shopify stores via public JSON endpoints.

### Prerequisites
- None (uses public endpoints)
- No authentication required for public product data

### Setup Steps

**No setup required!** Shopify provides public JSON endpoints for all stores.

### Usage in Workflows

**Tier 2 Scraper Node:**
```json
{
  "node": "HTTP Request",
  "method": "GET",
  "url": "https://{{$json.shopify_domain}}/products.json",
  "options": {
    "pagination": {
      "paginationMode": "responseContainsNextURL",
      "nextURL": "={{$response.body.next_page}}"
    }
  }
}
```

### Endpoint Format
```
Base URL: https://{store-name}.myshopify.com/products.json
Pagination: ?page=1&limit=250
Product Detail: /products/{product-handle}.json
```

### Rate Limits
- **Requests per second:** 2 (public endpoints)
- **Recommendation:** Add 500ms delay between requests

### Best Practices

**1. Pagination**
```javascript
// Handle pagination in n8n
let allProducts = [];
let page = 1;
let hasMore = true;

while (hasMore) {
  const response = await fetch(
    `https://${domain}/products.json?page=${page}&limit=250`
  );
  const data = await response.json();
  allProducts = allProducts.concat(data.products);
  hasMore = data.products.length === 250;
  page++;
}
```

**2. Error Handling**
```javascript
// Check for rate limiting
if (response.status === 429) {
  const retryAfter = response.headers.get('Retry-After');
  await sleep(retryAfter * 1000);
  // Retry request
}
```

### Troubleshooting

**Error: "404 Not Found"**
- Solution: Verify Shopify domain is correct
- Check if store has password protection enabled

**Error: "429 Too Many Requests"**
- Solution: Implement exponential backoff
- Reduce concurrent requests

---

## Zyte API Setup

### Purpose
Render JavaScript-heavy websites and extract HTML for AI processing.

### Prerequisites
- Zyte account (https://www.zyte.com)
- API key

### Setup Steps

**1. Create Zyte Account**
```
1. Go to https://www.zyte.com/zyte-api/
2. Click "Start Free Trial"
3. Complete registration
4. Verify email
```

**2. Get API Key**
```
1. Log in to Zyte dashboard
2. Go to "API Access"
3. Copy API key
4. Store securely
```

**3. Configure in n8n**
```
1. Open n8n Cloud dashboard
2. Go to "Credentials" → "New"
3. Select "Header Auth"
4. Header Name: "Authorization"
5. Header Value: "Basic [base64(API_KEY:)]"
6. Save credential as "Zyte API"
```

### Usage in Workflows

**Tier 3 Scraper Node:**
```json
{
  "node": "HTTP Request",
  "method": "POST",
  "url": "https://api.zyte.com/v1/extract",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "zyte",
  "body": {
    "url": "{{$json.source_url}}",
    "browserHtml": true,
    "javascript": true
  }
}
```

### Pricing
- **Cost per request:** $0.00013
- **Estimated monthly cost:** ~$18.50 for 20 vendors (50 products each, weekly sync)

### Rate Limits
- **Concurrent requests:** 100 (default)
- **Recommendation:** Limit to 5 concurrent for cost control

### Best Practices

**1. Request Optimization**
```json
{
  "url": "https://vendor-website.com/products",
  "browserHtml": true,
  "javascript": true,
  "geolocation": "US",
  "actions": [
    {"action": "waitForSelector", "selector": ".product-list", "timeout": 10}
  ]
}
```

**2. Error Handling**
```javascript
// Retry logic for Zyte failures
const maxRetries = 3;
let attempt = 0;
let success = false;

while (attempt < maxRetries && !success) {
  try {
    const response = await zyteRequest(url);
    success = true;
  } catch (error) {
    attempt++;
    await sleep(2000 * attempt); // Exponential backoff
  }
}
```

### Troubleshooting

**Error: "Request timeout"**
- Solution: Increase timeout in request options
- Check if website is accessible

**Error: "Extraction failed"**
- Solution: Verify website structure hasn't changed
- Check if website blocks scrapers

---

## Google AI Studio (Gemini) API Setup

### Purpose
Extract structured product data from HTML using AI.

### Prerequisites
- Google Cloud Platform account
- AI Studio API key

### Setup Steps

**1. Get API Key**
```
1. Go to https://aistudio.google.com/app/apikey
2. Click "Create API Key"
3. Select project (or create new)
4. Copy API key
5. Store securely
```

**2. Configure in n8n**
```
1. Open n8n Cloud dashboard
2. Go to "Credentials" → "New"
3. Select "Header Auth"
4. Header Name: "x-goog-api-key"
5. Header Value: "[YOUR_API_KEY]"
6. Save credential as "Gemini API"
```

### Usage in Workflows

**Tier 3 AI Extraction Node:**
```json
{
  "node": "HTTP Request",
  "method": "POST",
  "url": "https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "gemini",
  "body": {
    "contents": [{
      "parts": [{
        "text": "Extract product data from this HTML: {{$json.html}}"
      }]
    }]
  }
}
```

### Extraction Prompt Template

```
Extract the following product information from the HTML:
- SKU (product identifier)
- Title (product name)
- Description (marketing copy)
- Price (MSRP)
- Images (all image URLs)
- Videos (all video URLs)
- Category
- Tags/Keywords

Return as JSON with these exact field names.
If a field is not found, use null.

HTML:
{{html_content}}
```

### Pricing
- **Input tokens:** $0.00025 per 1K tokens
- **Output tokens:** $0.00075 per 1K tokens
- **Estimated cost:** ~$0.001 per product extraction

### Rate Limits
- **Requests per minute:** 60
- **Recommendation:** Add 1-second delay between requests

### Best Practices

**1. Optimize Prompts**
```
- Be specific about field names
- Request JSON output format
- Include examples for complex fields
- Limit HTML input to relevant sections
```

**2. Token Management**
```javascript
// Truncate HTML to reduce tokens
const relevantHTML = extractProductSection(fullHTML);
const maxTokens = 8000; // Leave room for response
const truncatedHTML = truncateToTokens(relevantHTML, maxTokens);
```

### Troubleshooting

**Error: "Invalid API key"**
- Solution: Verify API key in n8n credentials

**Error: "Quota exceeded"**
- Solution: Check usage in Google Cloud Console
- Upgrade quota if needed

**Error: "Invalid JSON response"**
- Solution: Improve prompt to enforce JSON format
- Add JSON validation step

---

## Security Best Practices

### Credential Storage
- ✅ Store all API keys in n8n credential manager (encrypted)
- ❌ Never hardcode credentials in workflows
- ✅ Use environment variables for sensitive data
- ✅ Rotate credentials quarterly

### Access Control
- ✅ Limit credential access to specific workflows
- ✅ Use least-privilege principle
- ✅ Audit credential usage regularly
- ✅ Revoke unused credentials

### Monitoring
- ✅ Monitor API usage and costs
- ✅ Set up alerts for unusual activity
- ✅ Log all API errors
- ✅ Review logs weekly

---

## Cost Monitoring

### Monthly Cost Breakdown

| API | Tier | Cost | Usage |
|-----|------|------|-------|
| Google Drive | 1 | $0 | 30 vendors |
| Dropbox | 1 | $0 | Included in Tier 1 |
| Google Sheets | 1 | $0 | Included in Tier 1 |
| Shopify | 2 | $0 | 20 vendors |
| Zyte | 3 | ~$18.50 | 20 vendors × 50 products × 4 weeks |
| Gemini | 3 | ~$0.50 | 20 vendors × 50 products × 4 weeks |
| **Total** | | **~$19/month** | **~$228/year** |

### Cost Optimization Tips

**1. Reduce Tier 3 Usage**
- Move vendors to Tier 1/2 when possible
- Negotiate data exports with vendors

**2. Optimize Scraping**
- Cache aggressively
- Reduce sync frequency for stable catalogs
- Batch requests efficiently

**3. Monitor Usage**
```sql
-- Track Tier 3 usage
SELECT 
  vendor_id,
  COUNT(*) as scrapes,
  SUM(processing_time_seconds) as total_time
FROM sync_log
WHERE tier_used = 3
AND sync_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY vendor_id
ORDER BY scrapes DESC;
```

---

## Testing API Integrations

### Test Checklist

**Google Drive:**
- [ ] Can authenticate successfully
- [ ] Can list files in folder
- [ ] Can download CSV/Excel files
- [ ] Can parse file contents

**Dropbox:**
- [ ] Can authenticate successfully
- [ ] Can access shared folders
- [ ] Can download files
- [ ] Can handle large files

**Google Sheets:**
- [ ] Can authenticate successfully
- [ ] Can read sheet data
- [ ] Can handle pagination
- [ ] Can parse different formats

**Shopify:**
- [ ] Can fetch products.json
- [ ] Can handle pagination
- [ ] Can parse product data
- [ ] Can handle rate limits

**Zyte:**
- [ ] Can render JavaScript sites
- [ ] Can extract HTML
- [ ] Can handle timeouts
- [ ] Can retry failures

**Gemini:**
- [ ] Can extract product data
- [ ] Can return valid JSON
- [ ] Can handle large HTML
- [ ] Can process multiple products

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Technical Team  
**Last Updated:** March 19, 2026

