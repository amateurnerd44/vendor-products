# Setup Guide - Vendor Product Data Collection System

This guide walks you through setting up the complete system from scratch.

---

## Prerequisites

Before you begin, ensure you have:

- [ ] n8n Cloud account (https://n8n.io)
- [ ] Google Cloud Console account (for Drive, Sheets APIs)
- [ ] Dropbox Developer account (if using Dropbox vendors)
- [ ] Zyte API account (https://www.zyte.com/zyte-api/)
- [ ] Google AI Studio account (https://aistudio.google.com/)
- [ ] Email account for receiving MarketTime confirmations
- [ ] Shopify store (if using Shopify order webhooks)

---

## Phase 1: n8n Data Tables Setup

### Step 1: Create Vendor Config Table

1. Log in to n8n Cloud
2. Navigate to **Data** → **Tables**
3. Click **Create Table**
4. Name: `vendor_config`
5. Add columns:

| Column Name | Type | Constraints |
|------------|------|-------------|
| vendor_id | Text | Primary Key |
| vendor_name | Text | Required |
| data_source_type | Text | Required |
| source_url | Text | Optional |
| credentials | JSON | Optional, Encrypted |
| last_sync_date | DateTime | Optional |
| sync_status | Text | Default: 'pending' |
| tier_used | Number | Required |
| notes | Long Text | Optional |

6. Click **Create Table**

### Step 2: Create Product Data Table

1. Click **Create Table**
2. Name: `product_data`
3. Add columns:

| Column Name | Type | Constraints |
|------------|------|-------------|
| sku | Text | Primary Key |
| vendor_id | Text | Required |
| product_name | Text | Required |
| image_urls | JSON | Optional |
| video_urls | JSON | Optional |
| marketing_copy | Long Text | Optional |
| metadata | JSON | Optional |
| last_updated | DateTime | Required |
| source_tier | Number | Required |

4. Click **Create Table**

### Step 3: Create Sync Log Table (Optional)

1. Click **Create Table**
2. Name: `sync_log`
3. Add columns:

| Column Name | Type | Constraints |
|------------|------|-------------|
| log_id | Number | Primary Key, Auto-increment |
| sync_date | DateTime | Required |
| vendor_id | Text | Required |
| status | Text | Required |
| products_updated | Number | Default: 0 |
| products_added | Number | Default: 0 |
| errors | JSON | Optional |
| duration_seconds | Number | Optional |

4. Click **Create Table**

---

## Phase 2: API Credentials Setup

### Zyte API

1. Sign up at https://www.zyte.com/zyte-api/
2. Navigate to **API Access**
3. Copy your API key
4. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **HTTP Header Auth**
   - Name: `Zyte API`
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_ZYTE_API_KEY`

### Google AI Studio (Gemini)

1. Go to https://aistudio.google.com/app/apikey
2. Click **Create API Key**
3. Copy the API key
4. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **HTTP Header Auth**
   - Name: `Google AI Studio`
   - Header Name: `x-goog-api-key`
   - Header Value: `YOUR_GOOGLE_AI_API_KEY`

### Google Drive

1. Go to Google Cloud Console (https://console.cloud.google.com)
2. Create a new project or select existing
3. Enable **Google Drive API**
4. Create OAuth 2.0 credentials:
   - Application type: Web application
   - Authorized redirect URIs: `https://your-instance.n8n.cloud/oauth/callback`
5. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **Google Drive OAuth2 API**
   - Enter Client ID and Client Secret
   - Click **Connect** and authorize

### Dropbox

1. Go to https://www.dropbox.com/developers/apps
2. Click **Create app**
3. Choose **Scoped access** → **Full Dropbox** → Name your app
4. In **Settings**, copy App key and App secret
5. Generate an access token (for testing) or set up OAuth
6. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **Dropbox OAuth2 API**
   - Enter App Key and App Secret
   - Click **Connect** and authorize

### Google Sheets

1. In Google Cloud Console, enable **Google Sheets API**
2. Create a **Service Account**:
   - Go to **IAM & Admin** → **Service Accounts**
   - Click **Create Service Account**
   - Name it (e.g., "n8n-sheets-access")
   - Grant role: **Editor**
3. Create a JSON key:
   - Click on the service account
   - Go to **Keys** → **Add Key** → **Create new key** → **JSON**
   - Download the JSON file
4. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **Google Sheets Service Account**
   - Upload the JSON key file

---

## Phase 3: Email Configuration (MarketTime Triggers)

### Gmail IMAP Setup

1. Enable 2-factor authentication on your Gmail account
2. Generate an App Password:
   - Go to Google Account → Security → 2-Step Verification → App passwords
   - Select **Mail** and **Other (Custom name)**
   - Name it "n8n MarketTime"
   - Copy the 16-character password
3. In n8n:
   - Go to **Credentials** → **Add Credential**
   - Select **IMAP**
   - Host: `imap.gmail.com`
   - Port: `993`
   - User: `your_email@gmail.com`
   - Password: `your_app_password`
   - Enable SSL: Yes

---

## Phase 4: Shopify Webhook Setup (Optional)

### Create Webhook in Shopify

1. Log in to Shopify Admin
2. Go to **Settings** → **Notifications** → **Webhooks**
3. Click **Create webhook**
4. Event: **Order creation**
5. Format: **JSON**
6. URL: `https://your-instance.n8n.cloud/webhook/shopify-orders`
7. Copy the webhook secret
8. In n8n:
   - Create a webhook trigger node
   - Path: `/shopify-orders`
   - HTTP Method: POST
   - Copy the webhook URL to Shopify

---

## Phase 5: Populate Vendor Configuration

### Import Sample Vendors

1. Open `templates/vendor-config-template.csv`
2. Fill in your actual vendor data:
   - Replace `FOLDER_ID_HERE` with actual Google Drive folder IDs
   - Replace `SHEET_ID_HERE` with actual Google Sheets IDs
   - Update vendor names and URLs
3. Import to n8n:
   - Option A: Use n8n's **Import CSV** feature in Data Tables
   - Option B: Create an n8n workflow to read CSV and insert rows

### Example Import Workflow (n8n)

```
[Manual Trigger]
  ↓
[Read CSV File] (templates/vendor-config-template.csv)
  ↓
[Split In Batches] (batch size: 10)
  ↓
[n8n Data Table Insert] (table: vendor_config)
```

---

## Phase 6: Test Connectivity

### Test Each Tier

**Tier 1 - Google Drive:**
1. Create a test workflow:
   ```
   [Manual Trigger]
     ↓
   [Google Drive - List Files]
     - Folder ID: (from vendor_config)
     ↓
   [Display Results]
   ```
2. Execute and verify files are listed

**Tier 2 - Shopify:**
1. Create a test workflow:
   ```
   [Manual Trigger]
     ↓
   [HTTP Request]
     - URL: https://store.myshopify.com/products.json
     - Method: GET
     ↓
   [Display Results]
   ```
2. Execute and verify products are returned

**Tier 3 - Zyte + Gemini:**
1. Create a test workflow:
   ```
   [Manual Trigger]
     ↓
   [HTTP Request - Zyte API]
     - URL: (test website)
     - Headers: Authorization with Zyte credentials
     ↓
   [HTTP Request - Gemini API]
     - Extract product data from HTML
     ↓
   [Display Results]
   ```
2. Execute and verify extraction works

---

## Phase 7: Environment Variables

### Set n8n Environment Variables

1. In n8n Cloud, go to **Settings** → **Environment Variables**
2. Add the following variables (from `.env.example`):

```
SYNC_SCHEDULE=0 2 * * 0
MAX_RETRIES=3
RETRY_DELAY_SECONDS=5
SHOPIFY_RATE_LIMIT_PER_SECOND=2
GEMINI_RATE_LIMIT_PER_MINUTE=60
LOG_LEVEL=info
MONTHLY_COST_ALERT_THRESHOLD=50
API_TIMEOUT=30
```

---

## Phase 8: Security Checklist

- [ ] All API keys stored in n8n Credentials (not hardcoded)
- [ ] Vendor credentials encrypted in `vendor_config` table
- [ ] Webhook endpoints use HTTPS only
- [ ] Email app passwords used (not main password)
- [ ] Service account keys have minimal required permissions
- [ ] n8n workflows set to private (not public)
- [ ] Regular credential rotation scheduled (quarterly)

---

## Phase 9: Monitoring Setup

### Cost Tracking

1. Create a monitoring workflow:
   ```
   [Schedule Trigger] (daily)
     ↓
   [Query sync_log table]
     - Count Tier 3 scrapes in last 24 hours
     ↓
   [Calculate Costs]
     - Zyte: count × $0.00013
     - Gemini: count × $0.002
     ↓
   [Check Threshold]
     - If > daily budget, send alert
     ↓
   [Send Notification] (email/Slack)
   ```

### Sync Monitoring

1. Create a monitoring workflow:
   ```
   [Schedule Trigger] (after weekly sync)
     ↓
   [Query sync_log table]
     - Get latest sync results
     ↓
   [Check for Failures]
     - Filter status = 'failed'
     ↓
   [Send Alert] (if failures found)
   ```

---

## Phase 10: Backup & Recovery

### Enable Automatic Backups

1. In n8n Cloud, go to **Settings** → **Backups**
2. Enable automatic daily backups
3. Set retention period: 30 days
4. Test restore procedure with sample data

### Export Workflows

1. Export all workflows as JSON files
2. Store in version control (this repository)
3. Update exports after each workflow modification

---

## Troubleshooting

### Common Issues

**Issue:** Google Drive authentication fails
- **Solution:** Check OAuth redirect URI matches exactly
- **Solution:** Ensure Drive API is enabled in Google Cloud Console

**Issue:** Shopify rate limiting errors
- **Solution:** Add delay nodes between requests (500ms)
- **Solution:** Reduce `SHOPIFY_RATE_LIMIT_PER_SECOND`

**Issue:** Zyte API returns errors
- **Solution:** Verify API key is correct
- **Solution:** Check account balance and usage limits
- **Solution:** Review Zyte API documentation for error codes

**Issue:** Gemini API quota exceeded
- **Solution:** Reduce request frequency
- **Solution:** Upgrade to paid tier if needed
- **Solution:** Implement caching to reduce duplicate requests

**Issue:** n8n Data Table insert fails
- **Solution:** Check data types match schema
- **Solution:** Verify required fields are populated
- **Solution:** Check for duplicate primary keys

---

## Next Steps

After completing setup:

1. ✅ Proceed to **Phase 2** of implementation (Tier 1 Scrapers)
2. ✅ Test each scraper with sample vendors
3. ✅ Build **Phase 5** (Weekly Sync Workflow)
4. ✅ Build **Phase 6** (Order Fulfillment Workflow)
5. ✅ Run end-to-end tests
6. ✅ Deploy to production

---

## Support

For issues or questions:
- Review `docs/guides/` for additional documentation
- Check `docs/workflows/` for workflow examples
- Refer to `IMPLEMENTATION_PLAN.md` for architecture details

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-19  
**Author:** Codegen AI

