# Tier 3: Zyte API + Gemini AI Scraper

## Overview

The Tier 3 scraper is designed for vendors with custom websites that don't have public APIs or structured data sources. It combines **Zyte API** for JavaScript-enabled web rendering with **Google Gemini AI** for intelligent data extraction.

**Target Cost:** ~$0.50/vendor/week (~$222/year for 20 vendors)

---

## Architecture

### Components

1. **Zyte Fetcher** - Renders JavaScript-heavy pages and returns clean HTML
2. **Gemini Extractor** - Uses AI to extract structured product data from HTML
3. **Error Handler** - Manages retries, fallbacks, and error logging
4. **Cost Tracker** - Monitors API usage and alerts on budget overruns
5. **AI Prompts Config** - Version-controlled extraction templates

### Workflow Flow

```
Input (URL) 
  → Zyte Fetcher (render HTML)
    → Gemini Extractor (extract data)
      → Success? → Return product data
      → Failure? → Error Handler
        → Retry? → Back to Zyte Fetcher
        → Fallback? → Try simpler prompt
        → Log & Alert → Admin notification
  → Cost Tracker (log all API calls)
```

---

## Workflows

### 1. Zyte Fetcher (`zyte_fetcher.json`)

**Purpose:** Fetch and render web pages using Zyte API with JavaScript execution enabled.

**Input:**
```json
{
  "url": "https://vendor-website.com/product/abc-123",
  "maxRetries": 3
}
```

**Output (Success):**
```json
{
  "success": true,
  "html": "<html>...</html>",
  "url": "https://vendor-website.com/product/abc-123",
  "fetchedAt": "2026-03-19T16:10:00.000Z",
  "cost": 0.00013
}
```

**Output (Failure):**
```json
{
  "success": false,
  "error": "Zyte API request failed after 3 retries",
  "url": "https://vendor-website.com/product/abc-123",
  "retriesAttempted": 3
}
```

**Features:**
- JavaScript rendering enabled
- Automatic retry with exponential backoff (1s, 2s, 4s)
- Max 3 retry attempts
- Cost tracking ($0.00013 per request)

**Configuration:**
- **Zyte API Credentials:** Set up in n8n credentials manager
- **Timeout:** 60 seconds per request
- **Retry Strategy:** Exponential backoff

---

### 2. Gemini Extractor (`gemini_extractor.json`)

**Purpose:** Extract structured product data from HTML using Google Gemini AI.

**Input:**
```json
{
  "html": "<html>...</html>",
  "url": "https://vendor-website.com/product/abc-123",
  "pageType": "product"
}
```

**Output (Success):**
```json
{
  "success": true,
  "productData": {
    "sku": "ABC-123",
    "name": "Premium Widget",
    "images": ["https://cdn.example.com/image1.jpg"],
    "videos": ["https://cdn.example.com/video1.mp4"],
    "description": "High quality widget for all your needs",
    "metadata": {
      "price": "$99.99",
      "dimensions": "10x10x10 cm",
      "materials": "Aluminum"
    }
  },
  "url": "https://vendor-website.com/product/abc-123",
  "cost": 0.002,
  "extractedAt": "2026-03-19T16:10:00.000Z"
}
```

**Output (Failure):**
```json
{
  "success": false,
  "error": "Missing required fields: sku or name",
  "url": "https://vendor-website.com/product/abc-123"
}
```

**Features:**
- Uses Gemini 1.5 Flash (cost-effective)
- Supports multiple page types: `product`, `catalog`, `paginated`
- Automatic JSON validation
- Handles markdown code blocks in responses
- HTML truncation to 50k chars (Gemini limit)

**Page Types:**

1. **Product Page** - Single product detail page
   - Extracts: SKU, name, images, videos, description, metadata
   
2. **Catalog Page** - Multiple products on one page
   - Extracts: Array of products with SKU, name, images, product URLs
   
3. **Paginated Page** - Pagination detection
   - Extracts: Next page URL, all page URLs, page numbers

**Configuration:**
- **Gemini API Key:** Set up in n8n credentials manager
- **Model:** `gemini-1.5-flash`
- **Temperature:** 0.1 (low for consistent extraction)
- **Max Output Tokens:** 8192
- **Response Format:** JSON

---

### 3. Error Handler (`error_handler.json`)

**Purpose:** Intelligent error handling with retry logic, fallbacks, and admin alerts.

**Input:**
```json
{
  "error": "Timeout error",
  "url": "https://vendor-website.com/product/abc-123",
  "vendorId": "vendor-001",
  "attemptCount": 1
}
```

**Output (Retry):**
```json
{
  "action": "retry",
  "url": "https://vendor-website.com/product/abc-123",
  "vendorId": "vendor-001",
  "attemptCount": 2,
  "waitedSeconds": 10
}
```

**Output (Fallback):**
```json
{
  "action": "fallback",
  "fallbackMethod": "simple_prompt",
  "url": "https://vendor-website.com/product/abc-123",
  "vendorId": "vendor-001",
  "originalError": "Missing required fields: sku"
}
```

**Output (Log & Alert):**
```json
{
  "action": "log_and_alert",
  "errorType": "auth",
  "errorMessage": "Authentication failed",
  "url": "https://vendor-website.com/product/abc-123",
  "vendorId": "vendor-001",
  "attemptsMade": 3
}
```

**Error Classification:**

| Error Type | Should Retry? | Use Fallback? | Alert Admin? |
|------------|---------------|---------------|--------------|
| `timeout` | ✅ Yes | ❌ No | ❌ No |
| `rate_limit` | ✅ Yes | ❌ No | ❌ No |
| `auth` | ❌ No | ❌ No | ✅ Yes |
| `not_found` | ❌ No | ❌ No | ❌ No |
| `parse_error` | ❌ No | ✅ Yes | ❌ No |
| `extraction_error` | ❌ No | ✅ Yes | ❌ No |

**Retry Strategy:**
- Max 3 attempts
- Exponential backoff: 5s, 10s, 20s

**Fallback Strategy:**
- Use simpler AI prompt (`simple_fallback`)
- Try regex extraction (future enhancement)

**Alert Conditions:**
- Authentication errors
- Critical failures after max retries

---

### 4. Cost Tracker (`cost_tracker.json`)

**Purpose:** Track API usage costs and alert when budget thresholds are exceeded.

**Input:**
```json
{
  "service": "zyte",
  "cost": 0.00013,
  "vendorId": "vendor-001",
  "url": "https://vendor-website.com/product/abc-123",
  "operation": "scrape"
}
```

**Output:**
```json
{
  "success": true,
  "costStats": {
    "period": "last_7_days",
    "zyteCost": "0.2600",
    "geminiCost": "4.0000",
    "totalCost": "4.2600",
    "totalRequests": 2000,
    "weeklyBudget": "4.26",
    "weeklyBudgetUsed": "100.0%",
    "annualProjection": "221.52",
    "annualBudget": "222.00",
    "annualBudgetProjection": "99.8%",
    "alertLevel": "normal",
    "averageCostPerRequest": "0.002130",
    "timestamp": "2026-03-19T16:10:00.000Z"
  }
}
```

**Features:**
- Logs every API call to `sync_log` table
- Calculates weekly and annual cost projections
- Compares against budget thresholds
- Sends email alerts when over budget

**Budget Thresholds:**
- **Weekly Budget:** $4.26
- **Annual Budget:** $222
- **Warning Level:** 80% of budget
- **Critical Level:** 100% of budget

**Alert Levels:**
- `normal` - Under 80% of budget
- `warning` - 80-100% of budget
- `critical` - Over 100% of budget

---

## AI Prompts Configuration

**File:** `config/ai_prompts.json`

### Structure

```json
{
  "version": "1.0.0",
  "prompts": {
    "product": { ... },
    "catalog": { ... },
    "paginated": { ... },
    "simple_fallback": { ... }
  },
  "settings": {
    "maxHtmlLength": 50000,
    "temperature": 0.1,
    "model": "gemini-1.5-flash",
    "fallbackChain": ["product", "simple_fallback"]
  }
}
```

### Prompt Types

#### 1. Product Page Prompt
- **Use Case:** Single product detail pages
- **Extracts:** SKU, name, images, videos, description, metadata
- **Output Format:** JSON object

#### 2. Catalog Page Prompt
- **Use Case:** Product listing/catalog pages
- **Extracts:** Array of products
- **Output Format:** JSON array

#### 3. Paginated Page Prompt
- **Use Case:** Detecting pagination links
- **Extracts:** Next page URL, all page URLs
- **Output Format:** JSON object

#### 4. Simple Fallback Prompt
- **Use Case:** When main prompts fail
- **Extracts:** Basic product info (name, images)
- **Output Format:** JSON object

### Prompt Engineering Best Practices

1. **Be Specific:** Clearly define what to extract
2. **Provide Examples:** Include expected output format
3. **Handle Edge Cases:** Account for missing data
4. **Request JSON Only:** Avoid markdown formatting
5. **Version Control:** Update version number when changing prompts

---

## Testing

### Test Workflow

1. **Simple Product Page**
   ```json
   {
     "url": "https://simple-vendor.com/product/test-123",
     "pageType": "product"
   }
   ```
   - Expected: Clean extraction with all fields

2. **Complex Product Page**
   ```json
   {
     "url": "https://complex-vendor.com/product/test-456",
     "pageType": "product"
   }
   ```
   - Expected: Successful extraction despite complex HTML

3. **Catalog Page**
   ```json
   {
     "url": "https://vendor.com/catalog",
     "pageType": "catalog"
   }
   ```
   - Expected: Array of products

4. **Error Scenarios**
   - 404 page → Should fail gracefully
   - Timeout → Should retry 3 times
   - Invalid HTML → Should use fallback prompt

### Validation Checklist

- [ ] Zyte API credentials configured
- [ ] Gemini API credentials configured
- [ ] Email alerts configured (admin email)
- [ ] Sync log table exists
- [ ] AI prompts file exists
- [ ] Test with 3-5 real vendor websites
- [ ] Verify extraction accuracy >90%
- [ ] Confirm costs are within budget
- [ ] Test error handling (timeouts, failures)
- [ ] Verify cost tracking and alerts

---

## Cost Optimization

### Current Costs

- **Zyte API:** $0.00013 per request
- **Gemini 1.5 Flash:** ~$0.002 per request
- **Total per page:** ~$0.00213

### Weekly Projection (20 vendors, 100 products each)

- **Total Requests:** 2,000
- **Zyte Cost:** 2,000 × $0.00013 = $0.26
- **Gemini Cost:** 2,000 × $0.002 = $4.00
- **Weekly Total:** $4.26
- **Annual Total:** $221.52

### Optimization Strategies

1. **Cache Results:** Store extracted data for 7 days
2. **Batch Processing:** Process multiple products in one catalog page
3. **Selective Scraping:** Only scrape changed products
4. **Prompt Optimization:** Reduce HTML length sent to Gemini
5. **Fallback to Tier 2:** Try Shopify JSON first if applicable

---

## Troubleshooting

### Common Issues

#### 1. Zyte API Timeouts
**Symptom:** Requests timing out after 60s  
**Solution:** 
- Check vendor website speed
- Increase timeout to 90s if needed
- Verify Zyte API status

#### 2. Gemini Extraction Failures
**Symptom:** Missing required fields (SKU, name)  
**Solution:**
- Review AI prompt for clarity
- Check HTML structure of vendor site
- Use fallback prompt
- Manually inspect HTML for data location

#### 3. Cost Overruns
**Symptom:** Budget alerts triggered  
**Solution:**
- Review vendor list (reduce Tier 3 vendors)
- Increase sync interval (bi-weekly instead of weekly)
- Optimize prompts to reduce token usage
- Cache results longer

#### 4. Authentication Errors
**Symptom:** 401/403 errors from Zyte or Gemini  
**Solution:**
- Verify API keys are correct
- Check API quota limits
- Regenerate API keys if needed

---

## Integration with Main Workflows

### Flow 1: Weekly Sync

```javascript
// In weekly sync workflow
if (vendor.tier_used === 3) {
  // Call Zyte Fetcher
  const html = await executeWorkflow('Tier 3: Zyte Fetcher', {
    url: vendor.source_url,
    maxRetries: 3
  });
  
  if (html.success) {
    // Call Gemini Extractor
    const data = await executeWorkflow('Tier 3: Gemini Extractor', {
      html: html.html,
      url: vendor.source_url,
      pageType: 'product'
    });
    
    if (data.success) {
      // Save to product data table
      // Track cost
      await executeWorkflow('Tier 3: Cost Tracker', {
        service: 'zyte',
        cost: 0.00013,
        vendorId: vendor.vendor_id
      });
      await executeWorkflow('Tier 3: Cost Tracker', {
        service: 'gemini',
        cost: 0.002,
        vendorId: vendor.vendor_id
      });
    } else {
      // Call Error Handler
      await executeWorkflow('Tier 3: Error Handler', {
        error: data.error,
        url: vendor.source_url,
        vendorId: vendor.vendor_id
      });
    }
  }
}
```

### Flow 2: Order Fulfillment (On-Demand)

```javascript
// When SKU not found in cache
if (!productData && vendor.tier_used === 3) {
  // Construct product URL from SKU
  const productUrl = `${vendor.source_url}/products/${sku}`;
  
  // Fetch and extract
  const html = await executeWorkflow('Tier 3: Zyte Fetcher', {
    url: productUrl
  });
  
  if (html.success) {
    const data = await executeWorkflow('Tier 3: Gemini Extractor', {
      html: html.html,
      url: productUrl,
      pageType: 'product'
    });
    
    // Save to cache for future orders
    // Track cost
  }
}
```

---

## Maintenance

### Weekly Tasks
- [ ] Review cost tracker reports
- [ ] Check error logs for patterns
- [ ] Verify extraction accuracy

### Monthly Tasks
- [ ] Update AI prompts if accuracy drops
- [ ] Review vendor list (move to Tier 2 if possible)
- [ ] Optimize slow scrapers

### Quarterly Tasks
- [ ] Audit all Tier 3 vendors
- [ ] Test with new vendor websites
- [ ] Update documentation

---

## Success Metrics

- **Extraction Accuracy:** >90%
- **Weekly Cost:** <$0.50/vendor
- **Annual Cost:** <$300 total
- **Error Rate:** <5%
- **Average Response Time:** <10 seconds

---

## Future Enhancements

1. **Regex Fallback:** Add regex-based extraction as final fallback
2. **Screenshot Analysis:** Use Gemini Vision for image-based extraction
3. **Incremental Scraping:** Only scrape changed products
4. **Multi-Model Support:** Add Claude or GPT-4 as alternatives
5. **Smart Caching:** ML-based cache invalidation

---

## Support

For issues or questions:
- Check error logs in `sync_log` table
- Review cost tracker reports
- Contact admin if critical errors occur
- Update AI prompts if extraction quality degrades

---

**Last Updated:** 2026-03-19  
**Version:** 1.0.0  
**Author:** Codegen AI

