# Vendor Onboarding Guide
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026

---

## Overview

This guide provides step-by-step instructions for onboarding new vendors to the Vendor Product Data Collection System.

### Target Audience
- Operations team
- Vendor management team
- System administrators

### Onboarding Timeline
- **Tier 1:** 1-2 hours per vendor
- **Tier 2:** 30 minutes per vendor
- **Tier 3:** 2-3 hours per vendor

---

## Vendor Information Collection Template

### Basic Information

```
Vendor Name: ___________________________
Vendor ID (slug): ___________________________
Contact Name: ___________________________
Contact Email: ___________________________
Contact Phone: ___________________________
Website: ___________________________
Estimated Product Count: ___________________________
```

### Data Source Information

```
Data Source Type: [ ] Drive  [ ] Dropbox  [ ] Sheets  [ ] Shopify  [ ] Website
Source URL: ___________________________
Update Frequency: [ ] Daily  [ ] Weekly  [ ] Monthly  [ ] On-Demand
Data Format: ___________________________
Special Requirements: ___________________________
```

### Credentials (if applicable)

```
[ ] OAuth Access Granted
[ ] API Key Provided
[ ] Folder/Sheet Shared
[ ] Credentials Stored Securely
```

---

## Tier Selection Decision Tree

### Step 1: Identify Data Source

**Question 1: Does the vendor provide structured data exports?**
- YES → Go to Question 2
- NO → Go to Question 3

**Question 2: What format is the data export?**
- Google Drive folder (CSV/Excel) → **TIER 1**
- Dropbox folder (CSV/Excel) → **TIER 1**
- Google Sheets → **TIER 1**
- Other → Go to Question 3

**Question 3: Does the vendor have an e-commerce platform?**
- Shopify store → **TIER 2**
- WooCommerce store → **TIER 2**
- Other/Custom website → **TIER 3**

### Step 2: Verify Tier Assignment

**Tier 1 Checklist:**
```
□ Vendor willing to share data
□ Data is structured (CSV/Excel/Sheets)
□ Data includes required fields (SKU, name, images, pricing)
□ Data is updated regularly
□ Access can be granted via OAuth or sharing
```

**Tier 2 Checklist:**
```
□ Vendor has Shopify or WooCommerce store
□ Store is publicly accessible
□ Products are not password-protected
□ Product data is complete
□ Store has reasonable product count (<1000 products)
```

**Tier 3 Checklist:**
```
□ Vendor has public website with product pages
□ Website allows scraping (check robots.txt)
□ Products are accessible without login
□ Website structure is scrapable
□ Cost is acceptable (~$4.26/week per vendor)
```

### Step 3: Document Tier Assignment

```
Vendor: ___________________________
Tier Assigned: ___________________________
Reason: ___________________________
Fallback Tier (if applicable): ___________________________
Estimated Cost: ___________________________
```

---

## Tier 1: Drive/Dropbox/Sheets Onboarding

### Prerequisites

- Vendor has Google Drive, Dropbox, or Google Sheets account
- Vendor is willing to share data
- Data is in structured format (CSV, Excel, or Sheets)

### Step 1: Request Data Access

**Email Template:**
```
Subject: Product Data Access Request - [Your Company]

Hi [Vendor Contact],

We're setting up an automated system to sync your product catalog for faster order fulfillment. To do this, we need access to your product data.

Could you please share your product catalog with us via one of these methods:

Option 1: Google Drive
- Share a folder containing your product CSV/Excel files
- Share with: [your-email@domain.com]

Option 2: Dropbox
- Share a folder containing your product CSV/Excel files
- Share with: [your-email@domain.com]

Option 3: Google Sheets
- Share your product spreadsheet
- Share with: [your-email@domain.com]
- Grant "View" access

Required Data Fields:
- SKU (product identifier)
- Product Name
- Description
- Images (URLs or files)
- Pricing (MSRP, MAP, Cost)
- Dimensions (Weight, Length, Width, Height)
- Any additional product details

Please let me know if you have any questions!

Best regards,
[Your Name]
```

### Step 2: Verify Data Format

**Once access is granted:**
```
□ Download sample data file
□ Verify required fields are present
□ Check data quality (no missing SKUs, valid URLs)
□ Verify images are accessible
□ Confirm pricing data is accurate
□ Document any data quality issues
```

**Required Fields:**
- SKU (must be unique)
- Product Name
- Description
- Images (URLs or file paths)
- MSRP
- At least 3 other fields from 35-field schema

### Step 3: Configure in System

**Add to vendor_config table:**
```sql
INSERT INTO vendor_config (
  vendor_id,
  vendor_name,
  data_source_type,
  source_url,
  credentials,
  tier_used,
  sync_status
) VALUES (
  '[vendor-slug]',
  '[Vendor Name]',
  'drive', -- or 'dropbox' or 'sheets'
  '[folder_id or sheet_id]',
  '{"drive_folder_id": "[id]"}', -- or dropbox/sheets equivalent
  1,
  'pending'
);
```

### Step 4: Test Sync

```
□ Run manual sync test
□ Verify products are inserted into product_data
□ Check all 35 fields are mapped correctly
□ Verify images are accessible
□ Confirm pricing data is accurate
□ Document any issues
```

### Step 5: Production Deployment

```
□ Update sync_status to 'pending'
□ Wait for next weekly sync
□ Monitor first sync closely
□ Verify success
□ Notify vendor of successful onboarding
```

---

## Tier 2: Shopify Onboarding

### Prerequisites

- Vendor has Shopify store
- Store is publicly accessible
- Products are not password-protected

### Step 1: Verify Shopify Store

**Test Shopify JSON endpoint:**
```bash
curl https://[vendor-store].myshopify.com/products.json
```

**Expected Response:**
```json
{
  "products": [
    {
      "id": 123456789,
      "title": "Product Name",
      "handle": "product-name",
      "images": [...],
      "variants": [...]
    }
  ]
}
```

### Step 2: Check Product Count

```bash
# Count products
curl https://[vendor-store].myshopify.com/products.json?limit=250 | jq '.products | length'
```

**If product count > 250:**
- Pagination is required
- System handles this automatically

### Step 3: Configure in System

**Add to vendor_config table:**
```sql
INSERT INTO vendor_config (
  vendor_id,
  vendor_name,
  data_source_type,
  source_url,
  credentials,
  tier_used,
  fallback_to_tier3,
  sync_status
) VALUES (
  '[vendor-slug]',
  '[Vendor Name]',
  'shopify',
  'https://[vendor-store].myshopify.com',
  '{"shopify_domain": "[vendor-store].myshopify.com"}',
  2,
  true, -- Enable fallback to Tier 3 if Shopify fails
  'pending'
);
```

### Step 4: Test Sync

```
□ Run manual sync test
□ Verify products are inserted into product_data
□ Check pagination works (if >250 products)
□ Verify all 35 fields are mapped correctly
□ Verify images are accessible
□ Confirm pricing data is accurate
□ Document any issues
```

### Step 5: Production Deployment

```
□ Update sync_status to 'pending'
□ Wait for next weekly sync
□ Monitor first sync closely
□ Verify success
□ No need to notify vendor (no cooperation required)
```

---

## Tier 3: Website Scraping Onboarding

### Prerequisites

- Vendor has public website with product pages
- Website allows scraping (check robots.txt)
- Products are accessible without login
- Budget approved for scraping costs (~$4.26/week per vendor)

### Step 1: Analyze Website Structure

**Manual Inspection:**
```
□ Visit vendor website
□ Navigate to product pages
□ Identify product list page
□ Identify individual product pages
□ Check if JavaScript is required
□ Check robots.txt for scraping rules
□ Document website structure
```

**Key Information to Collect:**
- Product list URL: ___________________________
- Product page URL pattern: ___________________________
- JavaScript required: [ ] Yes  [ ] No
- Pagination: [ ] Yes  [ ] No
- Anti-bot measures: [ ] Yes  [ ] No

### Step 2: Test Zyte Rendering

**Test with Zyte API:**
```bash
curl -X POST https://api.zyte.com/v1/extract \
  -H "Authorization: Basic [base64(API_KEY:)]" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "[vendor-website-url]",
    "browserHtml": true,
    "javascript": true
  }'
```

**Verify:**
```
□ HTML is returned successfully
□ JavaScript is rendered
□ Product data is visible in HTML
□ Images are loaded
□ No blocking detected
```

### Step 3: Test AI Extraction

**Test with Gemini API:**
```bash
curl -X POST https://generativelanguage.googleapis.com/v1/models/gemini-pro:generateContent \
  -H "x-goog-api-key: [API_KEY]" \
  -H "Content-Type: application/json" \
  -d '{
    "contents": [{
      "parts": [{
        "text": "Extract product data from this HTML: [html_content]"
      }]
    }]
  }'
```

**Verify:**
```
□ Product data is extracted
□ JSON format is returned
□ All required fields are present
□ Data quality is acceptable
```

### Step 4: Estimate Costs

**Cost Calculation:**
```
Estimated Products: ___
Zyte Cost: ___ products × $0.00013 = $___
Gemini Cost: ___ products × $0.001 = $___
Weekly Cost: $___ × 4 weeks = $___/month
Annual Cost: $___ × 52 weeks = $___/year
```

**Budget Approval:**
```
□ Cost estimate documented
□ Budget approved by manager
□ Cost tracking configured
```

### Step 5: Configure in System

**Add to vendor_config table:**
```sql
INSERT INTO vendor_config (
  vendor_id,
  vendor_name,
  data_source_type,
  source_url,
  credentials,
  tier_used,
  sync_status
) VALUES (
  '[vendor-slug]',
  '[Vendor Name]',
  'website',
  '[vendor-website-url]',
  '{"website_url": "[url]", "zyte_api_key": "[stored in n8n]", "gemini_api_key": "[stored in n8n]"}',
  3,
  'pending'
);
```

### Step 6: Test Sync

```
□ Run manual sync test
□ Verify Zyte rendering works
□ Verify Gemini extraction works
□ Verify products are inserted into product_data
□ Check all 35 fields are mapped correctly
□ Verify images are accessible
□ Confirm pricing data is accurate
□ Monitor API costs
□ Document any issues
```

### Step 7: Production Deployment

```
□ Update sync_status to 'pending'
□ Wait for next weekly sync
□ Monitor first sync closely
□ Monitor API costs
□ Verify success
□ No need to notify vendor (no cooperation required)
```

---

## Tier Fallback Configuration

### When to Enable Fallback

**Enable Tier 2 → Tier 3 fallback when:**
- Vendor's Shopify store has frequent downtime
- Shopify API rate limits are hit regularly
- Product data in Shopify is incomplete
- Vendor is critical and must always sync

### How to Enable Fallback

**Update vendor_config:**
```sql
UPDATE vendor_config
SET fallback_to_tier3 = true
WHERE vendor_id = '[vendor-slug]';
```

**Cost Impact:**
- Only charged when fallback is used
- Typical usage: 1-2 fallbacks per month per vendor
- Additional cost: ~$0.50-$1.00/month per vendor

---

## Vendor Testing Procedures

### Pre-Production Testing

**Test Checklist:**
```
□ Manual sync test successful
□ Products inserted into product_data
□ All 35 fields populated (or null if not available)
□ Images accessible
□ Videos accessible (if applicable)
□ Pricing data accurate
□ No errors in sync_log
□ Processing time acceptable
```

### Production Monitoring

**First Sync Monitoring:**
```
□ Monitor sync execution in real-time
□ Check for errors
□ Verify product count matches expectations
□ Verify data quality
□ Document any issues
□ Fix issues before next sync
```

**Ongoing Monitoring:**
```
□ Review weekly sync reports
□ Check for consistent failures
□ Monitor data quality
□ Monitor API costs (Tier 3 only)
□ Respond to vendor issues promptly
```

---

## Vendor Communication Templates

### Onboarding Confirmation Email

```
Subject: Product Data Sync - Successfully Configured

Hi [Vendor Contact],

Great news! We've successfully configured your product data sync in our system.

Details:
- Vendor: [Vendor Name]
- Data Source: [Tier 1/2/3 description]
- Sync Schedule: Weekly (Sunday 2:00 AM)
- Product Count: ~[estimated count]

What happens next:
- Your products will be synced automatically every week
- We'll monitor the sync and alert you if any issues arise
- You don't need to do anything unless we contact you

If you have any questions or need to update your product data, please let us know!

Best regards,
[Your Name]
```

### Sync Failure Notification

```
Subject: Product Data Sync Issue - Action Required

Hi [Vendor Contact],

We encountered an issue syncing your product data this week.

Error: [error message]
Vendor: [Vendor Name]
Sync Date: [date]

Possible causes:
- [List possible causes based on error]

Action needed:
- [List specific actions vendor should take]

Please let us know once you've addressed this issue, and we'll re-run the sync.

Best regards,
[Your Name]
```

### Data Quality Issue Notification

```
Subject: Product Data Quality Issue

Hi [Vendor Contact],

We've noticed some data quality issues with your product sync:

Issues:
- [List specific issues, e.g., missing images, incomplete descriptions]

Impact:
- [Explain impact on order fulfillment]

Recommendation:
- [Suggest improvements to data quality]

Can you help us improve the data quality? This will ensure faster and more accurate order fulfillment.

Best regards,
[Your Name]
```

---

## Troubleshooting Common Issues

### Issue: Vendor Data Format Changed

**Symptoms:**
- Sync fails with parsing error
- Products not inserted into database
- Missing fields in product_data

**Resolution:**
1. Download latest data file from vendor
2. Compare with previous format
3. Update parsing logic in workflow
4. Test with new format
5. Deploy updated workflow
6. Re-run sync

### Issue: Vendor Credentials Expired

**Symptoms:**
- Sync fails with authentication error
- "Invalid credentials" or "Access denied"

**Resolution:**
1. Contact vendor for new credentials
2. Update credentials in n8n
3. Test authentication
4. Re-run sync
5. Monitor next sync

### Issue: Vendor Website Changed (Tier 3)

**Symptoms:**
- Zyte rendering fails
- AI extraction returns incomplete data
- Products not inserted correctly

**Resolution:**
1. Inspect new website structure
2. Update Zyte selectors if needed
3. Update Gemini extraction prompts
4. Test with new structure
5. Deploy updated workflow
6. Re-run sync

### Issue: High API Costs (Tier 3)

**Symptoms:**
- Monthly cost exceeds budget
- Zyte/Gemini usage higher than expected

**Resolution:**
1. Analyze vendor scraping frequency
2. Check if vendor has Shopify store (move to Tier 2)
3. Negotiate data export with vendor (move to Tier 1)
4. Reduce sync frequency if acceptable
5. Optimize AI extraction prompts
6. Monitor costs closely

---

## Vendor Offboarding

### When to Offboard

- Vendor relationship ended
- Vendor no longer carries relevant products
- Vendor data quality consistently poor
- Cost exceeds value

### Offboarding Steps

**1. Disable Vendor:**
```sql
UPDATE vendor_config
SET sync_status = 'disabled'
WHERE vendor_id = '[vendor-slug]';
```

**2. Archive Vendor Data:**
```sql
-- Move products to archive table
INSERT INTO product_data_archive
SELECT * FROM product_data
WHERE vendor_id = '[vendor-slug]';

-- Delete from active table
DELETE FROM product_data
WHERE vendor_id = '[vendor-slug]';
```

**3. Notify Stakeholders:**
```
□ Notify operations team
□ Notify sales team
□ Update vendor list
□ Document reason for offboarding
```

**4. Clean Up:**
```
□ Remove credentials from n8n
□ Revoke OAuth access
□ Remove from monitoring dashboards
□ Update documentation
```

---

## Vendor Onboarding Metrics

### Track These Metrics

**Onboarding Efficiency:**
- Time to onboard per tier
- Success rate on first sync
- Issues encountered per vendor
- Time to resolve issues

**Vendor Performance:**
- Sync success rate per vendor
- Data quality score per vendor
- API costs per vendor (Tier 3)
- Product count per vendor

**System Performance:**
- Total vendors onboarded
- Vendors by tier
- Overall sync success rate
- Total API costs

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Operations Team  
**Last Updated:** March 19, 2026

