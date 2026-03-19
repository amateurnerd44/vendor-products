# Vendor Product Data Collection & Order Fulfillment System
## Implementation Plan

---

## 🎯 Project Overview

**Goal:** Build an automated system that collects product data (images, videos, marketing copy, metadata) from 70+ vendors and delivers digital asset packages to customers when they place orders.

**Key Requirements:**
- Usage-based pricing (no monthly subscriptions)
- Handle varying vendor data sources (Google Drive, Dropbox, Sheets, Shopify sites, custom websites)
- Weekly automated catalog sync
- On-demand scraping fallback for missing SKUs
- Fast order fulfillment (5-10 seconds for cached products)

**Tech Stack:**
- **Orchestration:** n8n Cloud
- **Web Scraping:** Zyte API (pay-per-success, $0.00013/request)
- **AI Extraction:** Google AI Studio Gemini API (usage-based)
- **Data Storage:** n8n Data Tables
- **Triggers:** MarketTime email confirmations, Shopify order webhooks

---

## 📋 System Architecture

### **Flow 1: Weekly Product Catalog Sync**
- Runs weekly (e.g., Sunday 2 AM)
- Loops through all vendors
- Uses tiered data collection strategy
- Updates n8n data table with product information

### **Flow 2: Order Fulfillment**
- Triggered by MarketTime email or Shopify order
- Queries data table for SKUs
- Falls back to on-demand scraping if SKU not found
- Delivers digital assets to customer

### **Tiered Data Collection Strategy:**
1. **Tier 1:** Direct integrations (Google Drive/Dropbox/Sheets) - FREE
2. **Tier 2:** Shopify public JSON endpoints - FREE
3. **Tier 3:** Zyte API + Gemini AI - USAGE-BASED

---

## 🗂️ Data Models

### **Vendor Config Table**
```
- vendor_id (primary key, string)
- vendor_name (string)
- data_source_type (enum: 'drive', 'dropbox', 'sheets', 'shopify', 'website')
- source_url (string, nullable)
- credentials (JSON, encrypted, nullable)
- last_sync_date (timestamp)
- sync_status (enum: 'success', 'failed', 'pending')
- tier_used (integer: 1, 2, or 3)
- notes (text, nullable)
```

### **Product Data Table**
```
- sku (primary key, string)
- vendor_id (foreign key, string)
- product_name (string)
- image_urls (JSON array)
- video_urls (JSON array)
- marketing_copy (text)
- metadata (JSON)
- last_updated (timestamp)
- source_tier (integer: 1, 2, or 3)
```

### **Sync Log Table** (Optional but recommended)
```
- log_id (primary key, auto-increment)
- sync_date (timestamp)
- vendor_id (string)
- status (enum: 'success', 'partial', 'failed')
- products_updated (integer)
- products_added (integer)
- errors (JSON array)
- duration_seconds (integer)
```

---

## 🏗️ Implementation Phases

### **PHASE 1: Foundation & Infrastructure** 
**Status:** Can be developed FIRST (no dependencies)

**Deliverables:**
1. **Repository Setup**
   - Initialize project structure
   - Create documentation folders
   - Set up version control standards

2. **n8n Data Tables Setup**
   - Create Vendor Config table
   - Create Product Data table
   - Create Sync Log table (optional)
   - Document table schemas

3. **Vendor Configuration Template**
   - Create CSV/JSON template for vendor data
   - Document required fields
   - Create example vendor entries (5-10 samples)

4. **Environment Variables & Secrets**
   - Document required API keys (Zyte, Gemini, n8n)
   - Create `.env.example` file
   - Set up n8n credentials for:
     - Google Drive
     - Dropbox
     - Google Sheets
     - Zyte API
     - Google AI Studio

**Dependencies:** None  
**Estimated Time:** 2-4 hours  
**Can Run in Parallel With:** Phase 2, Phase 3

---

### **PHASE 2: Tier 1 - Direct Integration Scrapers**
**Status:** Can be developed SECOND (depends on Phase 1 data models)

**Deliverables:**
1. **Google Drive Integration**
   - n8n workflow to list files in vendor folders
   - Extract product data from structured folders
   - Parse filenames for SKUs
   - Download/reference image and video URLs

2. **Dropbox Integration**
   - Similar to Google Drive
   - Handle Dropbox-specific authentication
   - Extract product data from folder structure

3. **Google Sheets Integration**
   - Connect to vendor spreadsheets
   - Parse varying sheet formats
   - Extract SKU, name, images, videos, copy
   - Handle different column structures

4. **Data Normalization Module**
   - Standardize data from all Tier 1 sources
   - Map to Product Data table schema
   - Handle missing fields gracefully

**Dependencies:** Phase 1 (data models)  
**Estimated Time:** 6-8 hours  
**Can Run in Parallel With:** Phase 3, Phase 4

---

### **PHASE 3: Tier 2 - Shopify Public Endpoint Scraper**
**Status:** Can be developed SECOND (depends on Phase 1 data models)

**Deliverables:**
1. **Shopify Detection Logic**
   - Check if URL is a Shopify store
   - Test `/products.json` endpoint availability
   - Handle rate limiting

2. **Shopify Catalog Scraper**
   - Fetch paginated product list
   - Extract product data (title, images, variants, description)
   - Build SKU → product handle mapping
   - Handle Shopify-specific data structures

3. **SKU Mapping Database**
   - Store SKU → Shopify handle relationships
   - Enable fast lookups for order fulfillment
   - Update mappings on weekly sync

**Dependencies:** Phase 1 (data models)  
**Estimated Time:** 4-6 hours  
**Can Run in Parallel With:** Phase 2, Phase 4

---

### **PHASE 4: Tier 3 - Zyte + AI Scraper**
**Status:** Can be developed SECOND (depends on Phase 1 data models)

**Deliverables:**
1. **Zyte API Integration**
   - Set up Zyte API client in n8n
   - Configure HTTP request nodes
   - Handle authentication
   - Implement retry logic for failed requests

2. **Gemini AI Extraction Module**
   - Create prompts for product data extraction
   - Handle varying HTML structures
   - Extract: SKU, name, images, videos, marketing copy
   - Return structured JSON

3. **AI Prompt Templates**
   - Template for product page extraction
   - Template for catalog page extraction
   - Template for handling edge cases
   - Version control for prompts

4. **Error Handling & Fallbacks**
   - Retry failed Zyte requests
   - Log extraction failures
   - Fallback to simpler extraction if AI fails

**Dependencies:** Phase 1 (data models)  
**Estimated Time:** 8-10 hours  
**Can Run in Parallel With:** Phase 2, Phase 3

---

### **PHASE 5: Flow 1 - Weekly Catalog Sync Workflow**
**Status:** Must be developed THIRD (depends on Phases 2, 3, 4)

**Deliverables:**
1. **Main Sync Orchestrator**
   - n8n workflow with weekly cron trigger
   - Loop through all vendors in config table
   - Route to appropriate tier based on vendor config
   - Aggregate results

2. **Tier Router Logic**
   - Check vendor `data_source_type`
   - Call Tier 1, 2, or 3 scraper accordingly
   - Handle tier fallbacks (if Tier 2 fails, try Tier 3)

3. **Data Upsert Logic**
   - Check if SKU exists in Product Data table
   - Update existing records
   - Insert new records
   - Update `last_updated` timestamp

4. **Sync Logging & Monitoring**
   - Log sync start/end times
   - Track products added/updated per vendor
   - Log errors and failures
   - Send summary notification after sync

5. **Error Notifications**
   - Email/Slack alert for failed vendor syncs
   - Include vendor name, error details, tier used
   - Suggest manual intervention steps

**Dependencies:** Phases 2, 3, 4 (all scrapers must be complete)  
**Estimated Time:** 6-8 hours  
**Can Run in Parallel With:** Phase 6 (partially)

---

### **PHASE 6: Flow 2 - Order Fulfillment Workflow**
**Status:** Must be developed THIRD (depends on Phase 5 for data table population)

**Deliverables:**
1. **MarketTime Email Trigger**
   - n8n email trigger node
   - Parse MarketTime order confirmation emails
   - Extract: customer info, SKUs, vendor names

2. **Shopify Order Webhook Trigger**
   - Set up Shopify webhook in n8n
   - Parse order data
   - Extract: customer info, SKUs, vendor names

3. **SKU Lookup Logic**
   - Query Product Data table by SKU
   - Handle multiple SKUs per order
   - Aggregate product data

4. **On-Demand Scraping Fallback**
   - Detect missing SKUs
   - Get vendor config for that SKU's vendor
   - Run appropriate tier scraper (1, 2, or 3)
   - Save result to Product Data table
   - Continue with fulfillment

5. **Digital Asset Package Compiler**
   - Collect all images, videos, copy for ordered SKUs
   - Format data for customer delivery
   - Create download links or zip files

6. **Customer Delivery Module**
   - Email assets to customer
   - OR upload to shared Google Drive/Dropbox folder
   - OR provide download link
   - Log delivery confirmation

7. **Failure Handling**
   - If on-demand scrape fails, send alert to you
   - Include: customer name, SKU, vendor, error
   - Pause order fulfillment for manual intervention

**Dependencies:** Phase 5 (data must be populated)  
**Estimated Time:** 8-10 hours  
**Can Run in Parallel With:** Phase 7 (testing)

---

### **PHASE 7: Testing & Validation**
**Status:** Must be developed FOURTH (depends on Phases 5 & 6)

**Deliverables:**
1. **Unit Tests for Each Tier**
   - Test Tier 1 scrapers with sample Drive/Dropbox/Sheets
   - Test Tier 2 with known Shopify stores
   - Test Tier 3 with sample websites

2. **Integration Tests**
   - Test full Flow 1 with 5-10 sample vendors
   - Test full Flow 2 with sample orders
   - Test on-demand scraping fallback

3. **Edge Case Testing**
   - Missing SKUs
   - Failed scrapes
   - Invalid vendor configs
   - Rate limiting scenarios
   - Malformed data

4. **Performance Testing**
   - Measure sync time for 70 vendors
   - Measure order fulfillment time
   - Optimize slow scrapers

5. **Cost Validation**
   - Track Zyte API usage
   - Track Gemini API usage
   - Validate costs are within budget

**Dependencies:** Phases 5 & 6 (both flows must be complete)  
**Estimated Time:** 6-8 hours  
**Can Run in Parallel With:** Phase 8 (documentation)

---

### **PHASE 8: Documentation & Deployment**
**Status:** Must be developed LAST (depends on all previous phases)

**Deliverables:**
1. **User Documentation**
   - How to add a new vendor
   - How to update vendor configs
   - How to manually trigger syncs
   - How to troubleshoot common issues

2. **Technical Documentation**
   - Architecture diagrams
   - Data flow diagrams
   - API integration guides
   - n8n workflow export/import instructions

3. **Runbook**
   - Weekly sync monitoring checklist
   - Error response procedures
   - Vendor onboarding process
   - Cost monitoring guidelines

4. **Deployment Checklist**
   - Verify all API keys are set
   - Test both flows end-to-end
   - Set up monitoring/alerting
   - Schedule weekly sync
   - Enable order triggers

5. **Vendor Onboarding**
   - Populate vendor config table with all 70 vendors
   - Classify each vendor by tier
   - Test each vendor's data source
   - Run initial full sync

**Dependencies:** All previous phases  
**Estimated Time:** 4-6 hours  
**Can Run in Parallel With:** None (final phase)

---

## 📊 Dependency Graph

```
PHASE 1 (Foundation)
    ↓
    ├─→ PHASE 2 (Tier 1 Scrapers)
    ├─→ PHASE 3 (Tier 2 Scrapers)
    └─→ PHASE 4 (Tier 3 Scrapers)
            ↓
            └─→ PHASE 5 (Flow 1: Weekly Sync)
                    ↓
                    └─→ PHASE 6 (Flow 2: Order Fulfillment)
                            ↓
                            └─→ PHASE 7 (Testing)
                                    ↓
                                    └─→ PHASE 8 (Documentation & Deployment)
```

**Parallel Development Opportunities:**
- Phases 2, 3, 4 can all be developed simultaneously after Phase 1
- Phase 6 can start while Phase 5 is being finalized (but needs Phase 5 data)
- Phase 7 can start as soon as Phase 6 is functional
- Phase 8 documentation can be written alongside development

---

## ⏱️ Estimated Timeline

| Phase | Duration | Can Start After | Can Run in Parallel |
|-------|----------|-----------------|---------------------|
| Phase 1 | 2-4 hours | Immediately | Phases 2, 3, 4 |
| Phase 2 | 6-8 hours | Phase 1 | Phases 3, 4 |
| Phase 3 | 4-6 hours | Phase 1 | Phases 2, 4 |
| Phase 4 | 8-10 hours | Phase 1 | Phases 2, 3 |
| Phase 5 | 6-8 hours | Phases 2, 3, 4 | Phase 6 (partial) |
| Phase 6 | 8-10 hours | Phase 5 | Phase 7 |
| Phase 7 | 6-8 hours | Phases 5, 6 | Phase 8 (docs) |
| Phase 8 | 4-6 hours | Phase 7 | None |

**Total Sequential Time:** ~44-60 hours  
**With Parallel Development:** ~25-35 hours

---

## 💰 Cost Estimates

### **Weekly Sync (70 vendors, ~100 products each)**
- Tier 1 (30 vendors): FREE
- Tier 2 (20 vendors): FREE
- Tier 3 (20 vendors): 2,000 pages × $0.00013 = **$0.26/week**
- Gemini AI: 2,000 pages × $0.002 = **$4/week**
- **Weekly Total: ~$4.26**
- **Annual Total: ~$221**

### **Order Fulfillment (on-demand scraping)**
- Assume 10% of orders need on-demand scraping
- 100 orders/month × 10% × 2 products/order = 20 scrapes/month
- Zyte: 20 × $0.00013 = **$0.0026/month**
- Gemini: 20 × $0.002 = **$0.04/month**
- **Monthly Total: ~$0.04**
- **Annual Total: ~$0.50**

### **Grand Total: ~$222/year**

---

## 🚀 Success Metrics

1. **Sync Success Rate:** >95% of vendors sync successfully each week
2. **Order Fulfillment Speed:** <10 seconds for cached SKUs, <60 seconds with on-demand scraping
3. **Data Freshness:** All products updated within 7 days
4. **Cost Efficiency:** Stay under $300/year
5. **Manual Intervention Rate:** <5% of orders require manual handling

---

## 🔧 Technical Considerations

### **Rate Limiting**
- Shopify: Max 2 requests/second per store
- Zyte: Handled by their infrastructure
- Gemini: 60 requests/minute (free tier)

### **Error Handling**
- Retry failed requests 3 times with exponential backoff
- Log all errors to Sync Log table
- Send alerts for critical failures

### **Data Privacy**
- Encrypt vendor credentials in config table
- Don't store customer payment info
- Comply with vendor terms of service

### **Scalability**
- Current design handles 70 vendors easily
- Can scale to 200+ vendors with same architecture
- May need to optimize Gemini API usage at higher volumes

---

## 📝 Next Steps for Managing AI Agent

**Sequential Phases:**
1. Phase 1 → Phases 2, 3, 4 (parallel) → Phase 5 → Phase 6 → Phase 7 → Phase 8

**Recommended Development Order:**
1. Start with Phase 1 (foundation)
2. Assign Phases 2, 3, 4 to parallel development tracks
3. Once all scrapers are complete, build Phase 5
4. Build Phase 6 after Phase 5 has data
5. Test thoroughly in Phase 7
6. Document and deploy in Phase 8

**Critical Path:**
Phase 1 → Phase 4 (Tier 3 is most complex) → Phase 5 → Phase 6 → Phase 7 → Phase 8

---

## 📞 Support & Maintenance

**Weekly Tasks:**
- Monitor sync logs for failures
- Review cost usage (Zyte + Gemini)
- Update vendor configs as needed

**Monthly Tasks:**
- Review order fulfillment metrics
- Optimize slow scrapers
- Update AI prompts if extraction quality degrades

**Quarterly Tasks:**
- Audit vendor data sources for changes
- Review and update documentation
- Evaluate new scraping tools/APIs

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-19  
**Author:** Codegen AI  
**Project:** Vendor Product Data Collection System

