# Vendor Product Data Collection & Order Fulfillment System
## Implementation Plan - UPDATED 2026-03-19

---

## 🎯 Project Overview

**Goal:** Build an automated system that collects board game/toy product data from 70+ vendors and delivers digital asset packages to customers when they place orders.

**Key Requirements:**
- Usage-based pricing (no monthly subscriptions)
- Handle varying vendor data sources (Google Drive, Dropbox, Sheets, Shopify sites, custom websites)
- Weekly automated catalog sync
- On-demand scraping fallback for missing SKUs
- Fast order fulfillment (<10 seconds for cached products)
- Board game/toy industry specific data (BGG integration, player counts, mechanics, etc.)

**Tech Stack:**
- **Orchestration:** n8n Cloud (runs all workflows 24/7)
- **Web Scraping:** Zyte API (pay-per-success, $0.00013/request)
- **AI Extraction:** Google AI Studio Gemini API (usage-based)
- **Data Storage:** n8n Data Tables (built-in database)
- **Triggers:** MarketTime email confirmations, Shopify order webhooks
- **MarketTime Integration:** Google Drive CSV for item list sync

---

## 📋 System Architecture

### **Flow 1: Weekly Product Catalog Sync**
- Runs weekly (Sunday 2:00 AM via cron trigger)
- Fetches MarketTime items list from Google Drive
- Loops through all vendors in vendor_config table
- Routes each vendor to appropriate tier scraper (1, 2, or 3)
- Upserts product data to product_data table (batch processing)
- Logs results to sync_log table
- Sends error notifications via Slack/Email
- Supports tier fallback (Tier 2 → Tier 3 if Shopify fails)

### **Flow 2: Order Fulfillment**
- Triggered by MarketTime email (IMAP monitor) or Shopify webhook
- Parses order details (customer, SKUs, quantities)
- Queries product_data table for SKUs (cache lookup)
- Falls back to on-demand scraping if SKU not found
- Compiles digital asset package (images, videos, descriptions)
- Delivers to customer via Email/Google Drive/Dropbox/Download Link
- Handles failures with admin alerts and customer notifications

### **Tiered Data Collection Strategy:**
1. **Tier 1:** Direct integrations (Google Drive/Dropbox/Sheets) - FREE
2. **Tier 2:** Shopify public JSON endpoints - FREE
3. **Tier 3:** Zyte API + Gemini AI - USAGE-BASED (~$222/year for 20 vendors)

---

## 🗂️ Data Models

### **Vendor Config Table** (n8n Data Table)
```
- vendor_id (primary key, string)
- vendor_name (string)
- data_source_type (enum: 'drive', 'dropbox', 'sheets', 'shopify', 'website')
- source_url (string, nullable)
- credentials (JSON, encrypted, nullable)
- last_sync_date (timestamp)
- sync_status (enum: 'success', 'failed', 'pending')
- tier_used (integer: 1, 2, or 3)
- fallback_to_tier3 (boolean) - Enable Tier 2 → Tier 3 fallback
- notes (text, nullable)
```

### **Product Data Table** (n8n Data Table: "Products List")
**HYBRID SCHEMA - 35 Fields Total**

**Core Identifiers (5 fields):**
```
- id (auto-increment, primary key)
- UPC (string, nullable) - 12-digit barcode
- EAN (string, nullable) - 13-digit barcode
- SKU (string, unique, NOT NULL) - Primary product identifier
- Shopify_ID (string, nullable) - Shopify product ID
```

**Pricing & Ordering (6 fields):**
```
- MSRP (number, nullable) - Manufacturer's Suggested Retail Price
- MAP (number, nullable) - Minimum Advertised Price
- Cost (number, nullable) - Wholesale cost
- Case (number, nullable) - Units per case
- MOQ (number, nullable) - Minimum Order Quantity
- Increments_Above_MOQ (number, nullable) - Order increments
```

**Physical Dimensions (4 fields):**
```
- Weight_lbs (number, nullable)
- Length_in (number, nullable)
- Width_in (number, nullable)
- Height_in (number, nullable)
```

**Product Information (6 fields):**
```
- Name (string, NOT NULL) - Product name/title
- Copy_html (long text, nullable) - Product description in HTML
- Tagline (string, nullable) - Short marketing tagline
- keywords_list (string, nullable) - Comma-separated keywords
- Category_list (string, nullable) - Comma-separated categories
- Rules_url (string, nullable) - URL to rulebook PDF
```

**Board Game Specific (6 fields):**
```
- Ages_text (string, nullable) - Recommended age range (e.g., "8+")
- Player_Count_text (string, nullable) - Number of players (e.g., "2-4")
- Piece_Count_text (string, nullable) - Number of game pieces
- Play_Time (string, nullable) - Average play time (e.g., "30 minutes")
- BGG_weight (number, nullable) - BoardGameGeek complexity rating (1.0-5.0)
- Mechanics_list (string, nullable) - Comma-separated game mechanics
```

**Integration & Tracking (4 fields):**
```
- ClickUp_ID (string, nullable) - ClickUp task/item ID
- Shopify_Vendor_Id (string, nullable) - Shopify vendor ID
- createdAt (timestamp, auto) - Record creation timestamp
- updatedAt (timestamp, auto) - Last update timestamp
```

**NEW - Media Assets & Routing (4 fields - ADDED IN PHASE 6):**
```
- image_urls (JSON, nullable) - Array of product image URLs
- video_urls (JSON, nullable) - Array of product video URLs
- vendor_id (string, nullable) - Foreign key to vendor_config.vendor_id
- source_tier (number, nullable) - Tier used to collect data (1, 2, or 3)
```

### **Sync Log Table** (n8n Data Table)
```
- log_id (auto-increment, primary key)
- sync_date (timestamp)
- vendor_id (string)
- vendor_name (string)
- status (enum: 'success', 'partial', 'failed')
- products_updated (integer)
- products_added (integer)
- products_skipped (integer)
- new_markettime_items (integer) - Count of new MarketTime items found
- errors (JSON array)
- duration_seconds (integer)
- tier_used (integer: 1, 2, or 3)
- cost_estimate (number) - Estimated cost for Tier 3 vendors
```

---

## 🏗️ Implementation Phases

### **PHASE 1: Foundation & Infrastructure** ✅ COMPLETE
**Status:** ✅ Completed
**Duration:** 2-4 hours

**Deliverables:**
1. ✅ Repository structure created
2. ✅ n8n Data Tables schemas documented (vendor_config, product_data, sync_log)
3. ✅ Vendor configuration templates (CSV + JSON, 10 sample vendors)
4. ✅ Environment variables documented (.env.example)
5. ✅ Setup guide created (SETUP_GUIDE.md)

**Key Files:**
- `docs/schemas/n8n-data-tables.md`
- `docs/schemas/vendor_config.md`
- `docs/schemas/product_data.md`
- `docs/schemas/sync_log.md`
- `templates/vendor-config-template.csv`
- `templates/vendor-config-template.json`
- `.env.example`
- `docs/guides/SETUP_GUIDE.md`

---

### **PHASE 2: Tier 1 - Direct Integration Scrapers** ✅ COMPLETE
**Status:** ✅ Completed
**Duration:** 6-8 hours
**Dependencies:** Phase 1

**Deliverables:**
1. ✅ Google Drive scraper workflow
2. ✅ Dropbox scraper workflow
3. ✅ Google Sheets scraper workflow
4. ✅ Data normalizer subworkflow
5. ✅ Test workflows

**Key Features:**
- Google Drive: Handles multiple folder structures, generates shareable links
- Dropbox: Shared folder support, download links with `dl=1` parameter
- Google Sheets: Auto-detects columns, handles multi-sheet structures
- Data Normalizer: Cleans SKUs, validates URLs, removes HTML, adds metadata

**Key Files:**
- `workflows/tier1/google_drive_scraper.json`
- `workflows/tier1/dropbox_scraper.json`
- `workflows/tier1/google_sheets_scraper.json`
- `workflows/tier1/data_normalizer.json`
- `docs/workflows/TIER1_SCRAPERS.md`

---

### **PHASE 3: Tier 2 - Shopify Public Endpoint Scraper** ✅ COMPLETE
**Status:** ✅ Completed
**Duration:** 4-6 hours
**Dependencies:** Phase 1

**Deliverables:**
1. ✅ Shopify detector workflow
2. ✅ Shopify scraper workflow (with pagination)
3. ✅ SKU mapper workflow
4. ✅ HTML cleaner subworkflow
5. ✅ Test workflows

**Key Features:**
- Detects Shopify stores via `/products.json` endpoint
- Handles pagination (250 products per page)
- Extracts SKUs from product variants
- Respects rate limits (2 req/sec, 500ms delay)
- Extracts videos from HTML descriptions

**Key Files:**
- `workflows/tier2/shopify_detector.json`
- `workflows/tier2/shopify_scraper.json`
- `workflows/tier2/sku_mapper.json`
- `workflows/tier2/html_cleaner.json`
- `docs/workflows/TIER2_SHOPIFY.md`

---

### **PHASE 4: Tier 3 - Zyte + AI Scraper** ✅ COMPLETE
**Status:** ✅ Completed
**Duration:** 8-10 hours
**Dependencies:** Phase 1

**Deliverables:**
1. ✅ Zyte API integration workflow
2. ✅ Gemini AI extraction workflow
3. ✅ AI prompt templates (3+ scenarios)
4. ✅ Error handler workflow
5. ✅ Cost tracker workflow
6. ✅ Test workflows

**Key Features:**
- Zyte API: JavaScript rendering, anti-bot protection ($0.00013/request)
- Gemini AI: Structured data extraction (~$0.002/request)
- Retry logic with exponential backoff (max 3 attempts)
- Fallback to simpler prompts or regex on failure
- Cost tracking per vendor with budget alerts

**Key Files:**
- `workflows/tier3/zyte_fetcher.json`
- `workflows/tier3/gemini_extractor.json`
- `workflows/tier3/error_handler.json`
- `workflows/tier3/cost_tracker.json`
- `config/ai_prompts.json`
- `docs/workflows/TIER3_ZYTE_AI.md`

---

### **PHASE 5: Flow 1 - Weekly Catalog Sync Workflow** ✅ COMPLETE
**Status:** ✅ Completed
**Duration:** 6-8 hours
**Dependencies:** Phases 2, 3, 4

**Deliverables:**
1. ✅ Main sync orchestrator workflow (cron trigger: Sunday 2 AM)
2. ✅ Data upserter workflow (batch processing, 50 products)
3. ✅ Sync logger workflow
4. ✅ Error notifier workflow (Slack + Email)
5. ✅ Tier fallback handler workflow (Tier 2 → Tier 3)
6. ✅ Test workflows

**Key Features:**
- Fetches MarketTime items list from Google Drive (CSV)
- Loops through all vendors, routes to appropriate tier scraper
- Batch upserts products (INSERT new, UPDATE if newer)
- Tracks new MarketTime items
- Logs all sync operations to sync_log table
- Sends notifications for failed syncs
- Automatic tier fallback for Tier 2 vendors

**Key Files:**
- `workflows/flow1/weekly_sync.json`
- `workflows/flow1/data_upserter.json`
- `workflows/flow1/sync_logger.json`
- `workflows/flow1/error_notifier.json`
- `workflows/flow1/tier_fallback.json`
- `docs/workflows/FLOW1_WEEKLY_SYNC.md`

**Environment Variables Required:**
- `MARKETTIME_ITEMS_FOLDER_ID` - Google Drive file ID for MarketTime items list

---

### **PHASE 6: Flow 2 - Order Fulfillment Workflow** 🔄 IN PROGRESS
**Status:** 🔄 In Progress
**Duration:** 4-6 hours (includes Phase 5 updates)
**Dependencies:** Phase 5 (data must be populated)

**CRITICAL: Hybrid Schema Approach**
Phase 6 adds 4 new fields to the existing 31-field product_data schema:
- `image_urls` (JSON) - Product images
- `video_urls` (JSON) - Product videos
- `vendor_id` (Text) - Vendor routing
- `source_tier` (Number) - Tier tracking

**Deliverables:**
1. ⏳ Update product_data table schema (add 4 new fields)
2. ⏳ Update Phase 5 data_upserter to populate new fields
3. ⏳ Update tier scrapers to extract image/video URLs
4. ⏳ MarketTime email trigger workflow (IMAP monitor)
5. ⏳ Shopify webhook trigger workflow
6. ⏳ SKU lookup workflow (queries product_data table)
7. ⏳ On-demand scraper workflow (real-time scraping fallback)
8. ⏳ Asset compiler workflow (collects images/videos/copy)
9. ⏳ Customer delivery workflow (4 methods: Email/Drive/Dropbox/Download)
10. ⏳ Failure handler workflow (admin alerts, customer notifications)
11. ⏳ Test workflows

**Key Features:**
- Parses MarketTime order confirmation emails
- Receives Shopify order webhooks (HMAC validation)
- Queries product_data for SKUs (cache lookup)
- On-demand scraping for missing SKUs (30s timeout per SKU)
- Compiles digital asset packages
- Multiple delivery methods (configurable via DELIVERY_METHOD env var)
- Graceful failure handling with admin alerts

**Target Performance:**
- <10 seconds for cached SKUs
- <60 seconds with on-demand scraping
- >90% cache hit rate
- >99% delivery success rate
- <5% manual intervention rate

**Key Files (To Be Created):**
- `workflows/flow2/markettime_trigger.json`
- `workflows/flow2/shopify_trigger.json`
- `workflows/flow2/sku_lookup.json`
- `workflows/flow2/on_demand_scraper.json`
- `workflows/flow2/asset_compiler.json`
- `workflows/flow2/customer_delivery.json`
- `workflows/flow2/failure_handler.json`
- `docs/workflows/FLOW2_ORDER_FULFILLMENT.md`

---

### **PHASE 7: Testing & Validation** ⏳ PENDING
**Status:** ⏳ Pending (blocked by Phase 6)
**Duration:** 6-8 hours
**Dependencies:** Phases 5, 6

**Deliverables:**
1. ⏳ Unit tests for all tier scrapers
2. ⏳ Integration tests for Flow 1 (weekly sync)
3. ⏳ Integration tests for Flow 2 (order fulfillment)
4. ⏳ Edge case tests (20+ scenarios)
5. ⏳ Performance tests (sync time, fulfillment time, database queries)
6. ⏳ Cost validation tests (verify <$300/year)
7. ⏳ Test documentation (TEST_PLAN.md, TEST_RESULTS.md, PERFORMANCE_REPORT.md)

**Test Scenarios:**
- Missing SKUs, failed scrapes, invalid configs
- Rate limiting scenarios (Shopify, Gemini)
- Malformed data, large orders (50+ SKUs)
- Duplicate SKUs, empty catalogs
- Tier fallback scenarios

**Success Criteria:**
- >90% test coverage
- >95% sync success rate
- >99% fulfillment success rate
- <$300/year cost
- All issues resolved

---

### **PHASE 8: Documentation & Deployment** ⏳ PENDING
**Status:** ⏳ Pending (blocked by Phase 7)
**Duration:** 4-6 hours
**Dependencies:** Phase 7

**Deliverables:**
1. ⏳ User guide (adding vendors, monitoring, troubleshooting)
2. ⏳ Architecture documentation (system diagrams, data models, workflows)
3. ⏳ API integration guide (Drive, Dropbox, Sheets, Shopify, Zyte, Gemini)
4. ⏳ Runbook (weekly monitoring, error procedures, vendor onboarding)
5. ⏳ Deployment checklist (pre-deployment, n8n config, data setup, triggers, monitoring)
6. ⏳ Vendor onboarding guide (information collection, tier assignment, testing)
7. ⏳ Populate vendor_config with all 70 vendors
8. ⏳ Run initial full sync
9. ⏳ Deploy to production

**Success Criteria:**
- All documentation complete
- 70 vendors onboarded
- Initial sync >95% success
- System deployed and operational
- Ready for daily use

---

## 💰 Cost Analysis

### **Weekly Sync Cost (70 vendors, ~100 products each)**
- **Tier 1 (30 vendors):** FREE (Google Drive, Dropbox, Sheets APIs)
- **Tier 2 (20 vendors):** FREE (Shopify public endpoints)
- **Tier 3 (20 vendors):**
  - Zyte: 2,000 pages × $0.00013 = $0.26/week
  - Gemini: 2,000 pages × $0.002 = $4.00/week
  - **Weekly Total:** ~$4.26
  - **Annual Total:** ~$221

### **Order Fulfillment Cost (on-demand scraping)**
- Assume 10% of orders need on-demand scraping
- 100 orders/month × 10% × 2 products/order = 20 scrapes/month
- Zyte: 20 × $0.00013 = $0.0026/month
- Gemini: 20 × $0.002 = $0.04/month
- **Monthly Total:** ~$0.04
- **Annual Total:** ~$0.50

### **Grand Total: ~$222/year** ✅ Within Budget

---

## 📊 Project Timeline

### **Optimized Timeline (With Parallelization): 25-35 hours**

```
Phase 1 (2-4h) ✅ COMPLETE
    ↓
┌───┴───┬───────┐
│       │       │
Phase 2 Phase 3 Phase 4 ✅ ALL COMPLETE
(6-8h)  (4-6h)  (8-10h)
│       │       │
└───┬───┴───────┘
    ↓
Phase 5 (6-8h) ✅ COMPLETE
    ↓
Phase 6 (4-6h) 🔄 IN PROGRESS
    ↓
Phase 7 (6-8h) ⏳ PENDING
    ↓
Phase 8 (4-6h) ⏳ PENDING
    ↓
  DONE
```

**Critical Path:** Phase 1 → Phase 4 → Phase 5 → Phase 6 → Phase 7 → Phase 8

**Current Progress:** 5 of 8 phases complete (62.5%)

---

## 🎯 Success Metrics

### **System Performance**
- ✅ Sync success rate: >95% (Target)
- ✅ Sync duration: <2 hours for 70 vendors (Target)
- ⏳ Order fulfillment time: <10 seconds (cached SKUs) - To be tested in Phase 7
- ⏳ Order fulfillment time: <60 seconds (on-demand scraping) - To be tested in Phase 7
- ⏳ Cache hit rate: >90% - To be measured in Phase 6

### **Cost Efficiency**
- ✅ Annual cost: <$300/year (Target: ~$222)
- ✅ Weekly sync cost: ~$4.26
- ✅ Order fulfillment cost: ~$0.04/month

### **Reliability**
- ⏳ Manual intervention rate: <5% - To be measured in Phase 7
- ✅ Data freshness: All products updated within 7 days (weekly sync)
- ✅ Error recovery: Automatic fallbacks for all failure modes (Tier 2 → Tier 3)

---

## 📝 Key Decisions & Changes

### **Phase 5 Implementation Decisions:**
1. **Board Game/Toy Focus:** Product schema expanded to 31 fields with industry-specific data (BGG integration, player counts, mechanics, etc.)
2. **MarketTime Google Drive Integration:** Added CSV-based item list sync from Google Drive (not in original plan)
3. **Tier Fallback:** Implemented automatic Tier 2 → Tier 3 fallback for Shopify vendors
4. **Batch Processing:** Data upserter processes 50 products at a time for performance

### **Phase 6 Hybrid Schema Approach:**
1. **Preserve Existing Fields:** Keep all 31 board game/toy fields from Phase 5
2. **Add Media Asset Fields:** Add 4 new fields (image_urls, video_urls, vendor_id, source_tier)
3. **Update Phase 5 Workflows:** Modify data_upserter and tier scrapers to populate new fields
4. **Backward Compatible:** Existing data remains intact, new fields added incrementally

---

## 🔗 Key Resources

### **Documentation**
- `docs/schemas/` - Data table schemas
- `docs/workflows/` - Workflow documentation
- `docs/guides/SETUP_GUIDE.md` - Setup instructions
- `IMPLEMENTATION_PLAN.md` - This file (single source of truth)

### **Workflows**
- `workflows/tier1/` - Tier 1 scrapers (Drive, Dropbox, Sheets)
- `workflows/tier2/` - Tier 2 scraper (Shopify)
- `workflows/tier3/` - Tier 3 scraper (Zyte + AI)
- `workflows/flow1/` - Weekly sync workflow
- `workflows/flow2/` - Order fulfillment workflow (in progress)

### **Configuration**
- `.env.example` - Environment variables template
- `templates/` - Vendor configuration templates
- `config/ai_prompts.json` - AI extraction prompts

---

**Last Updated:** 2026-03-19  
**Current Phase:** Phase 6 (Order Fulfillment)  
**Overall Progress:** 62.5% (5 of 8 phases complete)

