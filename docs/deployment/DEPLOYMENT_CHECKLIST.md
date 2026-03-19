# Deployment Checklist
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026

---

## Overview

This checklist guides the deployment of the Vendor Product Data Collection System to production.

### Prerequisites
- n8n Cloud account with active subscription
- All API credentials obtained (Drive, Dropbox, Sheets, Zyte, Gemini)
- 70 vendors identified and categorized by tier
- MarketTime email access configured
- Shopify webhook access (if applicable)

### Estimated Time
- **Pre-Deployment:** 2-3 hours
- **Deployment:** 3-4 hours
- **Post-Deployment Validation:** 2-3 hours
- **Total:** 7-10 hours

---

## Phase 1: Pre-Deployment Validation

### 1.1 Environment Setup

**n8n Cloud Instance:**
```
□ n8n Cloud subscription active
□ Sufficient execution quota available
□ Data Tables feature enabled
□ Webhook endpoints accessible
□ Email/Slack notifications configured
```

**API Credentials:**
```
□ Google Drive OAuth credentials obtained
□ Dropbox OAuth credentials obtained
□ Google Sheets OAuth credentials obtained
□ Zyte API key obtained
□ Gemini API key obtained
□ All credentials tested and working
```

**Access & Permissions:**
```
□ Admin access to n8n Cloud dashboard
□ Access to MarketTime email inbox
□ Access to Shopify admin (if applicable)
□ Access to vendor data sources
□ Access to monitoring tools (Slack/Email)
```

### 1.2 Documentation Review

```
□ Read USER_GUIDE.md
□ Read ARCHITECTURE.md
□ Read API_INTEGRATION.md
□ Read RUNBOOK.md
□ Read VENDOR_ONBOARDING_GUIDE.md
□ Understand 35-field hybrid schema
□ Understand 3-tier approach
□ Understand Flow 1 and Flow 2
```

### 1.3 Vendor Data Collection

```
□ List of 70 vendors compiled
□ Vendor tier assignments determined
□ Vendor data sources identified
□ Vendor credentials collected
□ Vendor contact information documented
□ Estimated product counts per vendor
```

**Tier Distribution Target:**
- Tier 1: ~30 vendors (Drive/Dropbox/Sheets)
- Tier 2: ~20 vendors (Shopify)
- Tier 3: ~20 vendors (Web scraping)

---

## Phase 2: n8n Configuration

### 2.1 Create Data Tables

**vendor_config Table:**
```sql
CREATE TABLE vendor_config (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  vendor_id VARCHAR(100) UNIQUE NOT NULL,
  vendor_name VARCHAR(255) NOT NULL,
  data_source_type ENUM('drive', 'dropbox', 'sheets', 'shopify', 'website'),
  source_url TEXT,
  credentials JSON,
  last_sync_date TIMESTAMP,
  sync_status ENUM('success', 'failed', 'pending', 'disabled') DEFAULT 'pending',
  tier_used INTEGER CHECK (tier_used IN (1, 2, 3)),
  fallback_to_tier3 BOOLEAN DEFAULT FALSE,
  products_synced INTEGER DEFAULT 0,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**product_data Table (35 Fields):**
```
See docs/schemas/product_data.md for complete schema
```

**sync_log Table:**
```sql
CREATE TABLE sync_log (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  vendor_id VARCHAR(100) NOT NULL,
  sync_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  sync_status ENUM('success', 'failed', 'partial'),
  products_synced INTEGER DEFAULT 0,
  error_message TEXT,
  processing_time_seconds INTEGER,
  tier_used INTEGER
);
```

**order_fulfillment_log Table:**
```sql
CREATE TABLE order_fulfillment_log (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  order_id VARCHAR(100),
  customer_email VARCHAR(255),
  sku_count INTEGER,
  fulfillment_status ENUM('success', 'failed', 'partial'),
  delivery_method ENUM('email', 'drive', 'dropbox', 'download_link'),
  processing_time_seconds INTEGER,
  error_message TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Verification:**
```
□ All tables created successfully
□ Table schemas match documentation
□ Indexes created on key fields (SKU, vendor_id)
□ Test insert/update/select operations
```

### 2.2 Configure Credentials

**Google Drive:**
```
□ Create credential: "Google Drive - Vendor Sync"
□ Type: Google Drive OAuth2 API
□ Enter Client ID and Client Secret
□ Authorize access
□ Test connection
```

**Dropbox:**
```
□ Create credential: "Dropbox - Vendor Sync"
□ Type: Dropbox OAuth2 API
□ Enter Access Token
□ Test connection
```

**Google Sheets:**
```
□ Create credential: "Google Sheets - Vendor Sync"
□ Type: Google Sheets OAuth2 API
□ Enter Client ID and Client Secret
□ Authorize access
□ Test connection
```

**Zyte API:**
```
□ Create credential: "Zyte API"
□ Type: Header Auth
□ Header Name: "Authorization"
□ Header Value: "Basic [base64(API_KEY:)]"
□ Test connection
```

**Gemini API:**
```
□ Create credential: "Gemini API"
□ Type: Header Auth
□ Header Name: "x-goog-api-key"
□ Header Value: [API_KEY]
□ Test connection
```

**Email (SMTP):**
```
□ Create credential: "Email - Customer Delivery"
□ Type: SMTP
□ Configure SMTP settings
□ Test email sending
```

**Email (IMAP):**
```
□ Create credential: "Email - MarketTime Monitor"
□ Type: IMAP
□ Configure IMAP settings
□ Test email reading
```

### 2.3 Import Workflows

**Flow 1: Weekly Catalog Sync:**
```
□ Import workflow from workflows/flow1_weekly_sync.json
□ Verify all nodes are connected
□ Update credential references
□ Configure cron trigger (Sunday 2 AM)
□ Test workflow with single vendor
□ Verify data is written to product_data table
```

**Flow 2: Order Fulfillment:**
```
□ Import workflow from workflows/flow2_order_fulfillment.json
□ Verify all nodes are connected
□ Update credential references
□ Configure IMAP trigger (MarketTime email)
□ Configure webhook trigger (Shopify)
□ Test workflow with sample order
□ Verify asset delivery works
```

**Tier 1 Scraper (Drive/Dropbox/Sheets):**
```
□ Import workflow from workflows/tier1_scraper.json
□ Verify all nodes are connected
□ Update credential references
□ Test with sample vendor data
□ Verify 35-field mapping
```

**Tier 2 Scraper (Shopify):**
```
□ Import workflow from workflows/tier2_scraper.json
□ Verify all nodes are connected
□ Test with sample Shopify store
□ Verify pagination works
□ Verify 35-field mapping
```

**Tier 3 Scraper (Zyte + AI):**
```
□ Import workflow from workflows/tier3_scraper.json
□ Verify all nodes are connected
□ Update credential references
□ Test with sample website
□ Verify Zyte rendering works
□ Verify Gemini extraction works
□ Verify 35-field mapping
```

---

## Phase 3: Vendor Configuration

### 3.1 Populate vendor_config Table

**Tier 1 Vendors (30 vendors):**
```
□ Insert all Tier 1 vendor records
□ Verify data_source_type = 'drive'|'dropbox'|'sheets'
□ Verify source_url is correct
□ Verify credentials are configured
□ Set tier_used = 1
□ Set sync_status = 'pending'
```

**Tier 2 Vendors (20 vendors):**
```
□ Insert all Tier 2 vendor records
□ Verify data_source_type = 'shopify'
□ Verify source_url (Shopify domain)
□ Set tier_used = 2
□ Set fallback_to_tier3 = true (recommended)
□ Set sync_status = 'pending'
```

**Tier 3 Vendors (20 vendors):**
```
□ Insert all Tier 3 vendor records
□ Verify data_source_type = 'website'
□ Verify source_url (website URL)
□ Set tier_used = 3
□ Set sync_status = 'pending'
```

**Verification:**
```sql
-- Verify vendor count by tier
SELECT tier_used, COUNT(*) as vendor_count
FROM vendor_config
GROUP BY tier_used;

-- Expected:
-- Tier 1: ~30
-- Tier 2: ~20
-- Tier 3: ~20
```

### 3.2 Test Individual Vendors

**Test 3 vendors per tier (9 total):**
```
□ Select 3 Tier 1 vendors for testing
□ Select 3 Tier 2 vendors for testing
□ Select 3 Tier 3 vendors for testing
□ Run manual sync for each test vendor
□ Verify products are inserted into product_data
□ Verify all 35 fields are populated
□ Verify images/videos are accessible
□ Document any issues
```

---

## Phase 4: Initial Sync

### 4.1 Pre-Sync Preparation

```
□ Verify all workflows are active
□ Verify all credentials are valid
□ Verify sufficient API quotas
□ Set up monitoring dashboard
□ Configure Slack/Email notifications
□ Ensure on-call engineer is available
□ Schedule sync for off-peak hours
```

### 4.2 Execute Initial Sync

**Trigger Flow 1: Weekly Catalog Sync:**
```
□ Open Flow 1 workflow
□ Click "Execute Workflow"
□ Monitor execution in real-time
□ Watch for error notifications
□ Log any issues immediately
□ Estimated duration: 6-8 hours
```

**Real-Time Monitoring:**
```
□ Check sync progress every 30 minutes
□ Monitor API usage (Zyte, Gemini)
□ Monitor n8n execution logs
□ Monitor Slack/Email alerts
□ Document any vendor failures
```

### 4.3 Post-Sync Validation

**Success Rate Check:**
```sql
SELECT 
  COUNT(*) as total_vendors,
  SUM(CASE WHEN sync_status = 'success' THEN 1 ELSE 0 END) as successful,
  ROUND(100.0 * SUM(CASE WHEN sync_status = 'success' THEN 1 ELSE 0 END) / COUNT(*), 2) as success_rate
FROM vendor_config
WHERE last_sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR);
```

**Expected:** Success rate ≥95% (≤3-4 vendors failing)

**Product Count Validation:**
```sql
SELECT 
  COUNT(DISTINCT vendor_id) as vendors_with_products,
  COUNT(*) as total_products,
  AVG(products_per_vendor) as avg_products_per_vendor
FROM (
  SELECT vendor_id, COUNT(*) as products_per_vendor
  FROM product_data
  GROUP BY vendor_id
) subquery;
```

**Expected:** 
- Vendors with products: ≥67 (95%+)
- Total products: ~3,500 (50 per vendor average)

**Error Analysis:**
```sql
SELECT 
  vendor_id,
  error_message,
  tier_used
FROM sync_log
WHERE sync_status = 'failed'
AND sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR);
```

**Action:** Investigate and fix failed vendors

---

## Phase 5: Production Deployment

### 5.1 Enable Triggers

**Flow 1: Weekly Catalog Sync:**
```
□ Verify cron trigger is configured (Sunday 2 AM)
□ Activate workflow
□ Verify trigger is enabled
□ Test trigger manually (if possible)
```

**Flow 2: Order Fulfillment - MarketTime Email:**
```
□ Configure IMAP trigger
□ Set email filter (Subject: "Order Confirmation")
□ Activate workflow
□ Verify trigger is enabled
□ Send test email to verify trigger
```

**Flow 2: Order Fulfillment - Shopify Webhook:**
```
□ Register webhook endpoint in Shopify admin
□ Event: orders/create
□ URL: https://[n8n-instance].n8n.cloud/webhook/shopify-order
□ Activate workflow
□ Verify webhook is registered
□ Test webhook with sample order
```

### 5.2 Configure Notifications

**Slack Notifications:**
```
□ Create Slack channel: #vendor-sync-alerts
□ Add team members to channel
□ Configure Slack webhook in n8n
□ Test notification sending
□ Verify notifications are received
```

**Email Notifications:**
```
□ Configure email distribution list
□ Add operations team to list
□ Configure email alerts in n8n
□ Test email sending
□ Verify emails are received
```

### 5.3 Set Up Monitoring

**n8n Dashboard:**
```
□ Create monitoring dashboard
□ Add workflow execution metrics
□ Add error rate metrics
□ Add performance metrics
□ Set up alerts for failures
```

**API Usage Monitoring:**
```
□ Set up Zyte API usage tracking
□ Set up Gemini API usage tracking
□ Configure cost alerts
□ Set budget thresholds
```

---

## Phase 6: Final Validation

### 6.1 Test Order Fulfillment

**Test Scenario 1: Cached SKU (Fast Path)**
```
□ Create test order with known SKU
□ Trigger order fulfillment
□ Verify processing time <10 seconds
□ Verify asset package is complete
□ Verify delivery method works
□ Verify customer receives assets
```

**Test Scenario 2: Missing SKU (On-Demand Scraping)**
```
□ Create test order with unknown SKU
□ Trigger order fulfillment
□ Verify on-demand scraping executes
□ Verify processing time <60 seconds
□ Verify SKU is added to product_data
□ Verify asset package is complete
□ Verify delivery method works
```

**Test Scenario 3: Multiple SKUs**
```
□ Create test order with 5-10 SKUs
□ Mix of cached and uncached SKUs
□ Trigger order fulfillment
□ Verify all SKUs are processed
□ Verify asset package includes all products
□ Verify delivery method works
```

**Test Scenario 4: All Delivery Methods**
```
□ Test email delivery (ZIP attachment)
□ Test Google Drive delivery (shared folder)
□ Test Dropbox delivery (shared folder)
□ Test download link delivery (temporary link)
□ Verify all methods work correctly
```

### 6.2 Verify System Health

**Workflow Status:**
```
□ All workflows are active
□ All triggers are enabled
□ No failed executions in last 24 hours
□ No error notifications
```

**Data Integrity:**
```
□ vendor_config table populated (70 vendors)
□ product_data table populated (~3,500 products)
□ sync_log table has initial sync records
□ No data corruption or missing fields
```

**API Connectivity:**
```
□ Google Drive API working
□ Dropbox API working
□ Google Sheets API working
□ Shopify API working
□ Zyte API working
□ Gemini API working
```

**Monitoring:**
```
□ Slack notifications working
□ Email notifications working
□ Monitoring dashboard accessible
□ API usage tracking working
□ Cost alerts configured
```

### 6.3 Performance Validation

**Sync Performance:**
```
□ Initial sync completed in <8 hours
□ Success rate ≥95%
□ No performance degradation
□ API costs within budget
```

**Order Fulfillment Performance:**
```
□ Cached SKU fulfillment <10 seconds
□ On-demand scraping <60 seconds
□ Success rate >98%
□ All delivery methods functional
```

---

## Phase 7: Go-Live

### 7.1 Final Checklist

```
□ All pre-deployment tasks completed
□ All configuration tasks completed
□ All vendor configuration tasks completed
□ Initial sync successful (≥95%)
□ All triggers enabled
□ All notifications configured
□ All monitoring set up
□ All validation tests passed
□ Team trained on system
□ Documentation complete and accessible
□ On-call rotation established
□ Rollback plan documented
```

### 7.2 Go-Live Announcement

**Send announcement to stakeholders:**
```
Subject: Vendor Product Data Collection System - Now Live

Team,

The Vendor Product Data Collection System is now live and operational!

System Capabilities:
- 70 vendors configured and syncing weekly
- ~3,500 products in catalog
- Order fulfillment <10 seconds (cached)
- On-demand scraping <60 seconds
- 4 delivery methods available

Key Information:
- Weekly sync: Sunday 2:00 AM
- Monitoring: #vendor-sync-alerts Slack channel
- Documentation: [Link to docs]
- Support: vendor-ops@[domain].com

Next Steps:
- Monitor first week closely
- Review weekly sync reports
- Provide feedback on system performance

Thank you for your support during deployment!
```

### 7.3 Post-Go-Live Monitoring

**First 24 Hours:**
```
□ Monitor all order fulfillment
□ Check for any errors
□ Verify notifications working
□ Respond to any issues immediately
□ Document any problems
```

**First Week:**
```
□ Monitor daily order fulfillment
□ Check API usage and costs
□ Review vendor sync status
□ Gather user feedback
□ Make minor adjustments as needed
```

**First Month:**
```
□ Review weekly sync reports
□ Analyze performance metrics
□ Identify optimization opportunities
□ Plan improvements
□ Update documentation based on learnings
```

---

## Rollback Procedures

### When to Rollback

**Trigger rollback if:**
- System completely non-functional
- Data corruption detected
- Security breach identified
- Cost exceeding budget by >100%
- Success rate <80% for >24 hours

### Rollback Steps

**1. Stop All Workflows:**
```
□ Deactivate Flow 1: Weekly Catalog Sync
□ Deactivate Flow 2: Order Fulfillment
□ Deactivate all tier scrapers
□ Disable all triggers
```

**2. Notify Stakeholders:**
```
□ Send rollback notification
□ Explain reason for rollback
□ Provide estimated recovery time
□ Set up status updates
```

**3. Restore from Backup:**
```
□ Restore n8n workflows from backup
□ Restore database from backup
□ Restore credentials
□ Verify data integrity
```

**4. Investigate Issue:**
```
□ Identify root cause
□ Document findings
□ Develop fix
□ Test fix thoroughly
```

**5. Re-Deploy:**
```
□ Apply fix
□ Test in staging (if available)
□ Re-deploy to production
□ Monitor closely
□ Verify issue is resolved
```

---

## Post-Deployment Tasks

### Week 1

```
□ Monitor system health daily
□ Review order fulfillment metrics
□ Check API usage and costs
□ Respond to user feedback
□ Document any issues
□ Make minor adjustments
```

### Week 2-4

```
□ Review weekly sync reports
□ Analyze performance trends
□ Identify optimization opportunities
□ Plan improvements
□ Update documentation
□ Train additional team members
```

### Month 2-3

```
□ Conduct performance review
□ Optimize tier assignments
□ Negotiate with vendors for better data access
□ Implement cost optimizations
□ Plan feature enhancements
□ Update runbook based on learnings
```

---

## Success Criteria

### System is considered successfully deployed when:

**Functional Requirements:**
- ✅ All 70 vendors configured and syncing
- ✅ Weekly sync success rate ≥95%
- ✅ Order fulfillment success rate >98%
- ✅ Cached SKU fulfillment <10 seconds
- ✅ On-demand scraping <60 seconds
- ✅ All delivery methods functional

**Operational Requirements:**
- ✅ Monitoring and alerting operational
- ✅ Team trained on system
- ✅ Documentation complete
- ✅ On-call rotation established
- ✅ Backup and recovery procedures tested

**Cost Requirements:**
- ✅ Monthly cost <$25 (~$300/year)
- ✅ Cost tracking and alerts configured
- ✅ Optimization plan in place

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Operations Team  
**Last Updated:** March 19, 2026

