# Vendor Product Data Collection System - User Guide
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026  
**Status:** ⚠️ Template documentation - to be verified post-deployment

---

## 📖 Table of Contents
1. [System Overview](#system-overview)
2. [Getting Started](#getting-started)
3. [Adding a New Vendor](#adding-a-new-vendor)
4. [Monitoring Weekly Syncs](#monitoring-weekly-syncs)
5. [Understanding Order Fulfillment](#understanding-order-fulfillment)
6. [Troubleshooting Common Issues](#troubleshooting-common-issues)
7. [Understanding Tier Selection](#understanding-tier-selection)
8. [FAQ](#faq)

---

## System Overview

### What Does This System Do?

The Vendor Product Data Collection System automatically:
- **Collects product data** from 70+ vendors weekly (images, videos, descriptions, pricing, metadata)
- **Stores product catalogs** in a centralized database with 35 standardized fields
- **Fulfills customer orders** by delivering digital asset packages within seconds
- **Handles multiple data sources** (Google Drive, Dropbox, Sheets, Shopify stores, custom websites)

### Key Capabilities

✅ **Automated Weekly Sync** - Every Sunday at 2:00 AM, the system syncs all vendor catalogs  
✅ **Fast Order Fulfillment** - Cached products deliver in <10 seconds  
✅ **On-Demand Scraping** - Missing SKUs are scraped in real-time (<60 seconds)  
✅ **Multi-Tier Approach** - Optimizes cost by using free methods first, paid scraping as fallback  
✅ **Board Game/Toy Focused** - Captures industry-specific data (player counts, ages, BGG ratings, mechanics)

### System Architecture (Simplified)

```
┌─────────────────────────────────────────────────────────────┐
│                    VENDOR DATA SOURCES                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Drive   │  │ Dropbox  │  │ Shopify  │  │ Websites │   │
│  │  (Tier 1)│  │ (Tier 1) │  │ (Tier 2) │  │ (Tier 3) │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              FLOW 1: WEEKLY CATALOG SYNC                     │
│         (Runs Sunday 2 AM, processes all vendors)            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│              PRODUCT DATABASE (35 Fields)                    │
│    SKU | Images | Videos | Pricing | Metadata | BGG Data    │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│         FLOW 2: ORDER FULFILLMENT (On-Demand)                │
│  Triggered by: MarketTime Email | Shopify Webhook            │
│  Delivers: Email | Google Drive | Dropbox | Download Link    │
└─────────────────────────────────────────────────────────────┘
```

---

## Getting Started

### Prerequisites

Before using the system, ensure you have:
- ✅ Access to n8n Cloud dashboard (contact admin for credentials)
- ✅ Permissions to view/edit vendor configurations
- ✅ Access to monitoring dashboards (Slack channel or email notifications)
- ✅ Understanding of your vendor's data source type

### First-Time Setup Checklist

1. **Review Vendor List** - Check which vendors are already configured
2. **Verify Your Access** - Ensure you can view the n8n workflows
3. **Join Notification Channels** - Get added to Slack/email alerts
4. **Review Tier Assignments** - Understand which tier each vendor uses
5. **Test Order Fulfillment** - Place a test order to see the system in action

### Accessing the System

**n8n Cloud Dashboard:**
- URL: `https://[your-instance].n8n.cloud` *(verify with admin)*
- Navigate to: **Workflows** → **Vendor Product Sync**
- Key workflows:
  - `Flow 1: Weekly Catalog Sync`
  - `Flow 2: Order Fulfillment`

**Database Access:**
- n8n Data Tables: **Workflows** → **Data** → **Tables**
- Key tables:
  - `vendor_config` - Vendor settings and credentials
  - `product_data` - Product catalog (35 fields)
  - `sync_log` - Sync history and errors

---

## Adding a New Vendor

### Step-by-Step Process

#### Step 1: Gather Vendor Information

Before adding a vendor, collect:
- ✅ **Vendor Name** (official business name)
- ✅ **Data Source Type** (Drive, Dropbox, Sheets, Shopify, Website)
- ✅ **Source URL** (where product data is located)
- ✅ **Credentials** (API keys, OAuth tokens, folder IDs)
- ✅ **Product Count** (estimated number of SKUs)
- ✅ **Update Frequency** (how often vendor updates their catalog)

#### Step 2: Determine Tier Assignment

Use the **Tier Selection Decision Tree** (see [Understanding Tier Selection](#understanding-tier-selection)):

**Tier 1 (FREE)** - Vendor provides structured data via:
- Google Drive folder with CSV/Excel files
- Dropbox folder with product sheets
- Google Sheets with product catalog

**Tier 2 (FREE)** - Vendor has:
- Shopify store with public product JSON endpoints
- WooCommerce store with REST API

**Tier 3 (PAID ~$4.26/week)** - Vendor has:
- Custom website requiring web scraping
- No structured data export
- Complex JavaScript-rendered pages

#### Step 3: Add Vendor to Configuration

**Option A: Via n8n Dashboard (Recommended)**
1. Open n8n Cloud dashboard
2. Navigate to **Data** → **Tables** → `vendor_config`
3. Click **Add Row**
4. Fill in vendor details:
   ```
   vendor_id: [unique-vendor-slug]
   vendor_name: [Official Vendor Name]
   data_source_type: [drive|dropbox|sheets|shopify|website]
   source_url: [URL to data source]
   credentials: [JSON with API keys - see examples below]
   tier_used: [1|2|3]
   fallback_to_tier3: [true|false]
   sync_status: pending
   ```
5. Click **Save**

**Option B: Via SQL Insert (Advanced)**
```sql
INSERT INTO vendor_config (
  vendor_id, vendor_name, data_source_type, source_url, 
  credentials, tier_used, fallback_to_tier3, sync_status
) VALUES (
  'acme-games',
  'Acme Board Games',
  'shopify',
  'https://acmegames.com',
  '{"shopify_domain": "acmegames.myshopify.com"}',
  2,
  true,
  'pending'
);
```

#### Step 4: Configure Credentials

**Tier 1 (Drive/Dropbox/Sheets) Credentials:**
```json
{
  "drive_folder_id": "1a2b3c4d5e6f7g8h9i0j",
  "oauth_token": "[stored in n8n credentials]"
}
```

**Tier 2 (Shopify) Credentials:**
```json
{
  "shopify_domain": "vendor-store.myshopify.com",
  "api_version": "2024-01"
}
```

**Tier 3 (Website Scraping) Credentials:**
```json
{
  "website_url": "https://vendor-website.com/products",
  "zyte_api_key": "[stored in n8n credentials]",
  "gemini_api_key": "[stored in n8n credentials]"
}
```

⚠️ **Security Note:** Never store API keys in plain text. Use n8n's credential management system.

#### Step 5: Test Vendor Configuration

1. **Manual Test Sync:**
   - Open `Flow 1: Weekly Catalog Sync` workflow
   - Click **Execute Workflow**
   - Select **Test Mode** with single vendor
   - Enter vendor_id: `[your-new-vendor]`
   - Click **Execute**

2. **Verify Results:**
   - Check `sync_log` table for success/failure
   - Review `product_data` table for new products
   - Confirm product count matches expectations

3. **Troubleshoot if Needed:**
   - See [Troubleshooting Common Issues](#troubleshooting-common-issues)
   - Check error logs in n8n workflow execution history
   - Verify credentials are correct

#### Step 6: Enable for Production

Once testing is successful:
1. Update vendor status: `sync_status = 'success'`
2. Vendor will be included in next weekly sync (Sunday 2 AM)
3. Monitor first production sync for any issues

---

## Monitoring Weekly Syncs

### Sync Schedule

**When:** Every Sunday at 2:00 AM (system timezone)  
**Duration:** 6-8 hours for all 70 vendors  
**Process:**
1. Fetch MarketTime items list from Google Drive
2. Loop through all vendors in `vendor_config`
3. Route each vendor to appropriate tier scraper
4. Upsert products to `product_data` table
5. Log results to `sync_log` table
6. Send notifications for errors

### Checking Sync Status

**Real-Time Monitoring:**
- Open n8n workflow: `Flow 1: Weekly Catalog Sync`
- View **Executions** tab to see current/recent runs
- Green = Success, Red = Failed, Yellow = Running

**Post-Sync Review:**
```sql
-- Check latest sync results
SELECT 
  vendor_id, 
  vendor_name, 
  sync_status, 
  last_sync_date,
  products_synced
FROM vendor_config
ORDER BY last_sync_date DESC;

-- View sync errors
SELECT * FROM sync_log
WHERE sync_status = 'failed'
AND sync_date >= NOW() - INTERVAL '7 days'
ORDER BY sync_date DESC;
```

### Success Criteria

✅ **Healthy Sync:**
- ≥95% vendor success rate (≤3-4 vendors failing)
- All Tier 1 vendors succeed (should be 100%)
- Tier 2 vendors ≥98% success (Shopify is reliable)
- Tier 3 vendors ≥90% success (web scraping can be flaky)

⚠️ **Warning Signs:**
- >5 vendors failing consistently
- Tier 1 vendors failing (indicates credential issues)
- Sync duration >10 hours (performance degradation)
- Product counts dropping significantly

### Notification Channels

**Slack Notifications:**
- Channel: `#vendor-sync-alerts` *(verify with admin)*
- Receives: Sync start, completion, errors, warnings

**Email Notifications:**
- Sent to: Operations team distribution list
- Includes: Error summary, failed vendor list, recommended actions

---

## Understanding Order Fulfillment

### How Order Fulfillment Works

**Trigger Sources:**
1. **MarketTime Email** - IMAP monitor detects order confirmation emails
2. **Shopify Webhook** - Real-time webhook when order is placed

**Fulfillment Process:**
```
Order Received → Parse SKUs → Check Cache (product_data)
                                    ↓
                          ┌─────────┴─────────┐
                          │                   │
                    Found in Cache      Not in Cache
                          │                   │
                    <10 seconds         On-Demand Scrape
                                             <60 seconds
                          │                   │
                          └─────────┬─────────┘
                                    ↓
                    Compile Asset Package
                    (images, videos, descriptions)
                                    ↓
                    Deliver to Customer
                    (Email/Drive/Dropbox/Link)
```

### Delivery Methods

**1. Email Delivery**
- Sends ZIP file with all assets
- Includes product descriptions in email body
- Best for: Small orders (<10 products)

**2. Google Drive Delivery**
- Creates shared folder with assets
- Organizes by SKU
- Best for: Large orders, repeat customers

**3. Dropbox Delivery**
- Uploads to customer's Dropbox folder
- Requires customer Dropbox account
- Best for: Customers with Dropbox integration

**4. Download Link**
- Generates temporary download link (expires in 7 days)
- Hosted on cloud storage
- Best for: One-time downloads

### Monitoring Order Fulfillment

**Check Recent Orders:**
```sql
SELECT 
  order_id,
  customer_email,
  sku_count,
  fulfillment_status,
  delivery_method,
  processing_time_seconds,
  created_at
FROM order_fulfillment_log
ORDER BY created_at DESC
LIMIT 20;
```

**Performance Metrics:**
- **Cached SKU Fulfillment:** <10 seconds (target)
- **On-Demand Scraping:** <60 seconds (target)
- **Success Rate:** >98% (target)

---

## Troubleshooting Common Issues

### Issue 1: Vendor Sync Failing

**Symptoms:**
- Vendor shows `sync_status = 'failed'` in `vendor_config`
- Error in `sync_log` table
- No products updated for vendor

**Common Causes & Solutions:**

**A. Credential Issues**
```
Error: "Authentication failed" or "Access denied"
Solution:
1. Verify credentials in n8n credential manager
2. Check if OAuth tokens need refresh
3. Confirm API keys are still valid
4. Test credentials manually (e.g., access Drive folder directly)
```

**B. Data Source Changed**
```
Error: "File not found" or "URL not accessible"
Solution:
1. Verify source_url is still correct
2. Check if vendor moved their data location
3. Contact vendor to confirm data source
4. Update source_url in vendor_config
```

**C. Rate Limiting**
```
Error: "Too many requests" or "Rate limit exceeded"
Solution:
1. Check if multiple syncs running simultaneously
2. Add delay between requests in workflow
3. Verify API rate limits haven't changed
4. Consider spreading sync across multiple days
```

**D. Data Format Changed**
```
Error: "Parsing failed" or "Invalid data format"
Solution:
1. Download sample data from vendor source
2. Compare with expected format
3. Update parsing logic in workflow
4. Test with new format before production
```

### Issue 2: Order Fulfillment Slow

**Symptoms:**
- Orders taking >60 seconds to fulfill
- Customer complaints about delays
- High on-demand scraping rate

**Solutions:**

**A. Improve Cache Hit Rate**
```
Problem: Too many SKUs not in cache
Solution:
1. Check if weekly sync is running successfully
2. Verify all vendors are syncing properly
3. Consider running sync more frequently (e.g., twice weekly)
4. Add popular SKUs to priority sync list
```

**B. Optimize On-Demand Scraping**
```
Problem: Scraping taking too long
Solution:
1. Review Zyte API performance metrics
2. Check if websites have anti-bot measures
3. Consider pre-caching frequently ordered SKUs
4. Optimize Gemini AI extraction prompts
```

### Issue 3: Missing Product Data

**Symptoms:**
- Products in database but missing images/videos
- Incomplete product descriptions
- Null values in critical fields

**Solutions:**

**A. Tier 1 (Drive/Dropbox/Sheets)**
```
Problem: Vendor file incomplete
Solution:
1. Contact vendor about missing data
2. Check if vendor has multiple files
3. Verify all columns are being parsed
4. Add fallback to Tier 3 if data consistently incomplete
```

**B. Tier 2 (Shopify)**
```
Problem: Shopify JSON missing fields
Solution:
1. Check if vendor has product variants
2. Verify metafields are being extracted
3. Confirm image URLs are accessible
4. Enable Tier 3 fallback for missing data
```

**C. Tier 3 (Web Scraping)**
```
Problem: AI extraction incomplete
Solution:
1. Review Gemini extraction prompts
2. Check if website structure changed
3. Verify Zyte is rendering JavaScript properly
4. Adjust extraction rules for specific vendor
```

### Issue 4: High Costs

**Symptoms:**
- Monthly costs exceeding $25 ($300/year target)
- Zyte API usage higher than expected
- Gemini API token usage high

**Solutions:**

**A. Audit Tier 3 Usage**
```sql
-- Check Tier 3 vendor usage
SELECT 
  vendor_id,
  COUNT(*) as scrape_count,
  AVG(processing_time_seconds) as avg_time
FROM sync_log
WHERE tier_used = 3
AND sync_date >= NOW() - INTERVAL '30 days'
GROUP BY vendor_id
ORDER BY scrape_count DESC;
```

**B. Optimize Tier Assignments**
```
Problem: Vendors in Tier 3 that could be Tier 1/2
Solution:
1. Review each Tier 3 vendor
2. Ask vendors if they can provide structured data
3. Check if vendor has Shopify store (move to Tier 2)
4. Negotiate data export access (move to Tier 1)
```

**C. Reduce Scraping Frequency**
```
Problem: Scraping too often
Solution:
1. Analyze product change frequency per vendor
2. Adjust sync schedule (e.g., bi-weekly for stable catalogs)
3. Implement change detection (only scrape if products changed)
4. Cache aggressively for rarely-changing products
```

---

## Understanding Tier Selection

### Tier Selection Decision Tree

```
START: New Vendor
    ↓
Does vendor provide structured data export?
    ├─ YES → Does vendor use Google Drive/Dropbox/Sheets?
    │         ├─ YES → TIER 1 (FREE)
    │         └─ NO → Continue below
    │
    └─ NO → Does vendor have Shopify/WooCommerce store?
              ├─ YES → TIER 2 (FREE)
              └─ NO → TIER 3 (PAID ~$4.26/week)
```

### Tier Characteristics

| Feature | Tier 1 | Tier 2 | Tier 3 |
|---------|--------|--------|--------|
| **Cost** | FREE | FREE | ~$4.26/week |
| **Data Source** | Drive/Dropbox/Sheets | Shopify/WooCommerce | Any Website |
| **Reliability** | 99%+ | 98%+ | 90%+ |
| **Setup Time** | 15 mins | 30 mins | 2 hours |
| **Maintenance** | Low | Low | Medium |
| **Data Quality** | Excellent | Good | Variable |
| **Vendor Cooperation** | Required | Not Required | Not Required |

### Tier 1: Direct Integrations (FREE)

**Best For:**
- Vendors who already maintain product spreadsheets
- Vendors willing to share data via cloud storage
- Vendors with structured data exports

**Requirements:**
- Vendor provides Google Drive folder, Dropbox folder, or Google Sheet
- Data includes: SKU, name, images, pricing, descriptions
- Vendor updates data regularly (weekly/monthly)

**Advantages:**
- ✅ Zero cost
- ✅ Highest reliability
- ✅ Best data quality (vendor-provided)
- ✅ Fast processing
- ✅ Easy to maintain

**Disadvantages:**
- ❌ Requires vendor cooperation
- ❌ Vendor must maintain data
- ❌ Limited to vendors willing to share

**Example Vendors:**
- Major distributors with data feeds
- Vendors using inventory management systems
- Vendors with existing data export processes

### Tier 2: Shopify Public API (FREE)

**Best For:**
- Vendors with Shopify stores
- Vendors with WooCommerce stores
- Vendors with public product APIs

**Requirements:**
- Vendor has Shopify store with public product JSON
- Products are publicly accessible
- No authentication required for product data

**Advantages:**
- ✅ Zero cost
- ✅ High reliability (Shopify uptime 99.9%+)
- ✅ Standardized data format
- ✅ No vendor cooperation needed
- ✅ Real-time data access

**Disadvantages:**
- ❌ Limited to Shopify/WooCommerce vendors
- ❌ May miss some product details
- ❌ Rate limits apply (though generous)

**Example Vendors:**
- Small/medium e-commerce businesses
- Direct-to-consumer brands
- Vendors using Shopify POS

### Tier 3: Web Scraping with AI (PAID)

**Best For:**
- Vendors with custom websites
- Vendors without structured data
- Vendors unwilling to share data

**Requirements:**
- Vendor has public website with product pages
- Products are accessible without login
- Website allows scraping (check robots.txt)

**Advantages:**
- ✅ Works with any website
- ✅ No vendor cooperation needed
- ✅ AI extracts complex data
- ✅ Handles JavaScript-rendered pages

**Disadvantages:**
- ❌ Costs ~$4.26/week per vendor (~$222/year)
- ❌ Lower reliability (90%+)
- ❌ Slower processing
- ❌ Requires maintenance if website changes
- ❌ May miss some data

**Cost Breakdown:**
- Zyte API: $0.00013 per request
- Gemini AI: ~$0.001 per product extraction
- Estimated: ~50 products/vendor × 52 weeks = $4.26/week

**Example Vendors:**
- Small businesses with custom websites
- Vendors with complex product catalogs
- Vendors without e-commerce platforms

### Tier Fallback Strategy

**Tier 2 → Tier 3 Fallback:**
- Enable with `fallback_to_tier3 = true` in vendor_config
- If Shopify sync fails, automatically try web scraping
- Useful for vendors with unreliable Shopify access
- Adds cost but improves reliability

**When to Enable Fallback:**
- Vendor's Shopify store has frequent downtime
- Shopify API rate limits are hit regularly
- Product data in Shopify is incomplete
- Critical vendor that must always sync

**Cost Impact:**
- Only charged when fallback is used
- Typical usage: 1-2 fallbacks per month per vendor
- Additional cost: ~$0.50-$1.00/month per vendor

---

## FAQ

### General Questions

**Q: How often does the system sync vendor data?**  
A: Weekly, every Sunday at 2:00 AM. The sync takes 6-8 hours to process all 70 vendors.

**Q: What happens if a vendor's website is down during sync?**  
A: The system logs the failure and sends a notification. The vendor will be retried in the next weekly sync. For critical vendors, enable Tier 3 fallback.

**Q: Can I trigger a manual sync for a specific vendor?**  
A: Yes, open the `Flow 1: Weekly Catalog Sync` workflow in n8n, enable test mode, and execute with the specific vendor_id.

**Q: How long are product images/videos stored?**  
A: Product data is stored indefinitely in the database. Images/videos are cached and refreshed weekly during syncs.

### Order Fulfillment Questions

**Q: What happens if a SKU is not in the database?**  
A: The system performs on-demand scraping to fetch the product data in real-time (<60 seconds). The product is then added to the database for future orders.

**Q: Can customers choose their delivery method?**  
A: Yes, delivery method is specified in the order (MarketTime email or Shopify checkout). Default is email delivery.

**Q: What if order fulfillment fails?**  
A: The system sends an alert to the operations team and notifies the customer of the delay. Manual intervention may be required.

### Technical Questions

**Q: What is the 35-field hybrid schema?**  
A: A standardized product data model that combines board game/toy-specific fields (player count, ages, BGG data) with generic e-commerce fields (images, videos, pricing). See `docs/schemas/product_data.md` for details.

**Q: How are API credentials stored securely?**  
A: All credentials are stored in n8n's encrypted credential manager. Never store credentials in plain text in the database.

**Q: Can I add custom fields to the product schema?**  
A: Yes, but requires updating the database schema and all workflows that read/write product data. Consult with the technical team before making changes.

### Cost Questions

**Q: What is the total annual cost?**  
A: Target is <$300/year. Breakdown:
- Tier 1 vendors: $0 (30 vendors)
- Tier 2 vendors: $0 (20 vendors)
- Tier 3 vendors: ~$222/year (20 vendors × $4.26/week × 52 weeks)
- n8n Cloud: Included in existing subscription

**Q: How can I reduce costs?**  
A: 
1. Move vendors from Tier 3 to Tier 1/2 when possible
2. Negotiate data exports with vendors
3. Reduce sync frequency for stable catalogs
4. Optimize Tier 3 scraping efficiency

**Q: What if costs exceed budget?**  
A: Review Tier 3 vendor usage, identify high-cost vendors, and work with them to provide structured data (move to Tier 1).

---

## Getting Help

### Support Channels

**Technical Issues:**
- Email: `tech-support@[your-domain].com` *(verify with admin)*
- Slack: `#vendor-sync-support`
- Response Time: <4 hours during business hours

**Vendor Onboarding:**
- Email: `vendor-ops@[your-domain].com` *(verify with admin)*
- Slack: `#vendor-onboarding`
- Response Time: <24 hours

**Emergency Issues:**
- Phone: `[Emergency Hotline]` *(verify with admin)*
- Use for: System down, critical vendor failures, data loss
- Available: 24/7

### Additional Resources

- **Technical Documentation:** `docs/technical/ARCHITECTURE.md`
- **API Integration Guide:** `docs/guides/API_INTEGRATION.md`
- **Operational Runbook:** `docs/guides/RUNBOOK.md`
- **Deployment Guide:** `docs/deployment/DEPLOYMENT_CHECKLIST.md`

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Operations Team  
**Last Updated:** March 19, 2026

