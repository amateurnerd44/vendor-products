# Vendor Product Data Collection System - Technical Architecture
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026

---

## System Overview

### Purpose
Automated system for collecting board game/toy product data from 70+ vendors and delivering digital asset packages to customers on-demand.

### Key Metrics
- **Vendors:** 70+ across 3 tiers
- **Weekly Sync:** 6-8 hours, Sunday 2 AM
- **Order Fulfillment:** <10s cached, <60s on-demand
- **Success Rate:** >95% sync success
- **Annual Cost:** <$300/year

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VENDOR DATA SOURCES                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────┐│
│  │ Tier 1 (30)  │  │ Tier 2 (20)  │  │ Tier 3 (20)  │  │MarketTime│
│  │ Drive/Dropbox│  │   Shopify    │  │ Zyte + AI    │  │  Drive  ││
│  │    Sheets    │  │  WooCommerce │  │  Scraping    │  │   CSV   ││
│  └──────────────┘  └──────────────┘  └──────────────┘  └─────────┘│
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    n8n CLOUD ORCHESTRATION                           │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  FLOW 1: WEEKLY CATALOG SYNC (Cron: Sunday 2 AM)             │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │ │
│  │  │  Fetch   │→ │  Route   │→ │  Scrape  │→ │  Upsert  │     │ │
│  │  │MarketTime│  │ by Tier  │  │   Data   │  │ Products │     │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │ │
│  └───────────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │  FLOW 2: ORDER FULFILLMENT (IMAP/Webhook Trigger)            │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │ │
│  │  │  Parse   │→ │  Query   │→ │ Compile  │→ │ Deliver  │     │ │
│  │  │  Order   │  │  Cache   │  │  Assets  │  │ Customer │     │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │ │
│  └───────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      n8n DATA TABLES (Database)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │vendor_config │  │ product_data │  │  sync_log    │             │
│  │  (70 rows)   │  │ (35 fields)  │  │  (history)   │             │
│  └──────────────┘  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                                  ↓
┌─────────────────────────────────────────────────────────────────────┐
│                      DELIVERY CHANNELS                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Email   │  │  Drive   │  │ Dropbox  │  │ Download │           │
│  │   ZIP    │  │  Folder  │  │  Folder  │  │   Link   │           │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Flow 1: Weekly Catalog Sync

### Purpose
Proactively sync all vendor product catalogs weekly to maintain fresh cache for fast order fulfillment.

### Trigger
- **Type:** Cron schedule
- **Schedule:** Sunday 2:00 AM (system timezone)
- **Frequency:** Weekly

### Process Flow

**Step 1: Initialize Sync**
```
1. Fetch MarketTime items list from Google Drive CSV
2. Load all vendors from vendor_config table
3. Filter vendors with sync_status != 'disabled'
4. Initialize sync_log entry
```

**Step 2: Vendor Loop**
```
FOR EACH vendor IN vendor_config:
  1. Check tier_used (1, 2, or 3)
  2. Route to appropriate scraper
  3. Execute scraping logic
  4. Collect product data
  5. Handle errors with fallback if enabled
```

**Step 3: Tier 1 Scraping (Drive/Dropbox/Sheets)**
```
1. Authenticate with OAuth credentials
2. Fetch file/folder from source_url
3. Parse CSV/Excel/Sheets data
4. Map columns to 35-field schema
5. Return product array
```

**Step 4: Tier 2 Scraping (Shopify)**
```
1. Construct Shopify JSON endpoint URL
   Format: https://{domain}/products.json
2. Fetch all products (paginated)
3. Extract product data from JSON
4. Map to 35-field schema
5. If fails and fallback_to_tier3=true, route to Tier 3
```

**Step 5: Tier 3 Scraping (Zyte + AI)**
```
1. Send website URL to Zyte API
2. Zyte renders JavaScript and returns HTML
3. Send HTML to Gemini AI with extraction prompt
4. AI extracts structured product data
5. Map to 35-field schema
6. Return product array
```

**Step 6: Upsert Products**
```
FOR EACH product IN scraped_products:
  1. Check if SKU exists in product_data table
  2. If exists: UPDATE with new data
  3. If not exists: INSERT new row
  4. Update vendor_id and source_tier fields
```

**Step 7: Log Results**
```
1. Update vendor_config:
   - last_sync_date = NOW()
   - sync_status = 'success' | 'failed'
   - products_synced = COUNT(products)
2. Insert sync_log entry:
   - vendor_id, sync_date, status, error_message
3. Send notifications if errors occurred
```

### Error Handling

**Tier 1 Errors:**
- Authentication failures → Refresh OAuth tokens
- File not found → Alert operations team
- Parsing errors → Log and skip vendor

**Tier 2 Errors:**
- Shopify API rate limit → Retry with backoff
- Invalid JSON → Enable Tier 3 fallback
- 404 errors → Verify vendor URL

**Tier 3 Errors:**
- Zyte API failures → Retry up to 3 times
- AI extraction failures → Log and alert
- Website blocking → Adjust scraping strategy

### Performance Optimization

**Parallel Processing:**
- Process multiple vendors concurrently (n8n parallel execution)
- Limit: 5 concurrent vendors to avoid rate limits

**Batch Upserts:**
- Collect all products per vendor
- Upsert in batches of 100 products
- Reduces database round trips

**Caching:**
- Cache vendor credentials in memory
- Cache MarketTime items list for sync duration
- Reuse HTTP connections

---

## Flow 2: Order Fulfillment

### Purpose
Deliver digital asset packages to customers when orders are placed via MarketTime or Shopify.

### Triggers

**Trigger 1: MarketTime Email (IMAP)**
```
- Monitor: Specific email inbox
- Filter: Subject contains "Order Confirmation"
- Parse: Order details from email body
- Extract: Customer email, SKU list, quantities
```

**Trigger 2: Shopify Webhook**
```
- Endpoint: https://[n8n-instance].n8n.cloud/webhook/shopify-order
- Event: orders/create
- Payload: Full order JSON
- Extract: Customer, line items, SKUs
```

### Process Flow

**Step 1: Parse Order**
```
1. Extract customer information:
   - Email, name, company
2. Extract SKU list:
   - SKU, quantity, product name
3. Validate order data:
   - Ensure SKUs are present
   - Verify customer email is valid
```

**Step 2: Query Cache (product_data table)**
```
FOR EACH sku IN order_skus:
  1. SELECT * FROM product_data WHERE SKU = sku
  2. If found: Add to cached_products array
  3. If not found: Add to missing_skus array
```

**Step 3: On-Demand Scraping (if needed)**
```
IF missing_skus.length > 0:
  FOR EACH sku IN missing_skus:
    1. Find vendor for SKU (from MarketTime list)
    2. Determine vendor tier
    3. Execute appropriate scraper
    4. Insert product into product_data table
    5. Add to scraped_products array
```

**Step 4: Compile Asset Package**
```
1. Collect all product data (cached + scraped)
2. Download images from image_urls
3. Download videos from video_urls
4. Generate product descriptions
5. Create ZIP file or folder structure
6. Include metadata (SKU, name, pricing, etc.)
```

**Step 5: Deliver to Customer**
```
SWITCH delivery_method:
  CASE 'email':
    - Attach ZIP file to email
    - Include product descriptions in body
    - Send via SMTP
  
  CASE 'drive':
    - Create Google Drive folder
    - Upload assets to folder
    - Share folder with customer email
    - Send notification email with link
  
  CASE 'dropbox':
    - Create Dropbox folder
    - Upload assets to folder
    - Share folder with customer
    - Send notification email with link
  
  CASE 'download_link':
    - Upload ZIP to cloud storage
    - Generate temporary download link (7-day expiry)
    - Send email with download link
```

**Step 6: Log Fulfillment**
```
1. Insert order_fulfillment_log entry:
   - order_id, customer_email, sku_count
   - fulfillment_status, delivery_method
   - processing_time_seconds, created_at
2. Send success notification to operations team
3. If errors: Send alert with error details
```

### Performance Targets

| Metric | Target | Actual (Monitor) |
|--------|--------|------------------|
| Cached SKU fulfillment | <10 seconds | TBD |
| On-demand scraping | <60 seconds | TBD |
| Success rate | >98% | TBD |
| Email delivery | <5 seconds | TBD |
| Drive/Dropbox delivery | <30 seconds | TBD |

---

## Data Models

### vendor_config Table

```sql
CREATE TABLE vendor_config (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  vendor_id VARCHAR(100) UNIQUE NOT NULL,
  vendor_name VARCHAR(255) NOT NULL,
  data_source_type ENUM('drive', 'dropbox', 'sheets', 'shopify', 'website') NOT NULL,
  source_url TEXT,
  credentials JSON,  -- Encrypted, stored in n8n credential manager
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

### product_data Table (35 Fields)

See `docs/schemas/product_data.md` for complete schema.

**Key Fields:**
- **Identifiers:** UPC, EAN, SKU, Shopify_ID
- **Pricing:** MSRP, MAP, Cost, Case, MOQ
- **Dimensions:** Weight, Length, Width, Height
- **Board Game Data:** Ages, Player_Count, Play_Time, BGG_Rating, Mechanics
- **Marketing:** Title, Description, Category, Tags
- **Assets:** image_urls (JSON array), video_urls (JSON array)
- **Metadata:** vendor_id, source_tier, last_updated

### sync_log Table

```sql
CREATE TABLE sync_log (
  id INTEGER PRIMARY KEY AUTO_INCREMENT,
  vendor_id VARCHAR(100) NOT NULL,
  sync_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  sync_status ENUM('success', 'failed', 'partial') NOT NULL,
  products_synced INTEGER DEFAULT 0,
  error_message TEXT,
  processing_time_seconds INTEGER,
  tier_used INTEGER,
  FOREIGN KEY (vendor_id) REFERENCES vendor_config(vendor_id)
);
```

---

## Technology Stack

### Core Platform
- **n8n Cloud:** Workflow orchestration, data tables, credential management
- **Version:** Latest stable (2024+)
- **Hosting:** n8n Cloud (managed service)

### Data Collection
- **Tier 1:** Google Drive API, Dropbox API, Google Sheets API
- **Tier 2:** Shopify REST API, WooCommerce REST API
- **Tier 3:** Zyte API (web scraping), Google AI Studio Gemini API (extraction)

### Integrations
- **Email:** IMAP (MarketTime order monitoring), SMTP (customer delivery)
- **Webhooks:** Shopify order webhooks
- **Cloud Storage:** Google Drive, Dropbox (asset delivery)

### Cost Structure
- **n8n Cloud:** Included in existing subscription
- **Tier 1 APIs:** FREE (within generous quotas)
- **Tier 2 APIs:** FREE (public endpoints)
- **Tier 3 APIs:** 
  - Zyte: $0.00013/request
  - Gemini: ~$0.001/product
  - Total: ~$222/year for 20 vendors

---

## Component Responsibilities

### n8n Workflows

**Flow 1: Weekly Catalog Sync**
- Orchestrates vendor scraping
- Routes vendors to appropriate tier
- Handles errors and retries
- Logs results and sends notifications

**Flow 2: Order Fulfillment**
- Parses order triggers
- Queries product cache
- Performs on-demand scraping
- Compiles and delivers assets

### Tier Scrapers

**Tier 1 Scraper (Drive/Dropbox/Sheets)**
- Authenticates with OAuth
- Fetches files/folders
- Parses structured data
- Maps to 35-field schema

**Tier 2 Scraper (Shopify)**
- Constructs JSON endpoint URLs
- Fetches paginated products
- Extracts product data
- Handles rate limits

**Tier 3 Scraper (Zyte + AI)**
- Sends URLs to Zyte API
- Receives rendered HTML
- Sends HTML to Gemini AI
- Extracts structured data

### Data Tables

**vendor_config**
- Stores vendor settings
- Manages credentials (encrypted)
- Tracks sync status
- Enables tier fallback

**product_data**
- Stores 35-field product data
- Serves as fulfillment cache
- Updated weekly via Flow 1
- Queried by Flow 2

**sync_log**
- Tracks sync history
- Logs errors and warnings
- Enables performance monitoring
- Supports troubleshooting

---

## Security Considerations

### Credential Management
- **Storage:** n8n encrypted credential manager
- **Access:** Restricted to workflow execution context
- **Rotation:** Quarterly credential rotation recommended
- **Audit:** Log all credential access

### API Security
- **Rate Limiting:** Respect vendor API rate limits
- **Authentication:** Use OAuth 2.0 where available
- **Encryption:** All API calls over HTTPS
- **Secrets:** Never log API keys or tokens

### Data Privacy
- **Customer Data:** Encrypt customer emails and names
- **Vendor Data:** Respect vendor terms of service
- **Compliance:** GDPR/CCPA compliant data handling
- **Retention:** Delete old sync logs after 90 days

---

## Monitoring & Observability

### Key Metrics

**Sync Performance:**
- Sync duration (target: <8 hours)
- Vendor success rate (target: >95%)
- Products synced per vendor
- Error rate by tier

**Order Fulfillment:**
- Fulfillment time (target: <10s cached, <60s on-demand)
- Cache hit rate (target: >90%)
- Delivery success rate (target: >98%)
- On-demand scraping frequency

**Cost Tracking:**
- Zyte API usage (requests/month)
- Gemini API usage (tokens/month)
- Total monthly cost (target: <$25)

### Alerting

**Critical Alerts:**
- Sync failure rate >10%
- Order fulfillment failures
- API credential expiration
- Cost exceeding budget

**Warning Alerts:**
- Sync duration >10 hours
- Cache hit rate <80%
- Individual vendor failures
- API rate limit warnings

### Logging

**Sync Logs:**
- Vendor ID, sync date, status
- Products synced, errors
- Processing time, tier used

**Fulfillment Logs:**
- Order ID, customer, SKUs
- Fulfillment status, delivery method
- Processing time, errors

---

## Scalability Considerations

### Current Capacity
- **Vendors:** 70 (30 Tier 1, 20 Tier 2, 20 Tier 3)
- **Products:** ~3,500 (50 products/vendor average)
- **Orders:** ~100/week (estimated)

### Growth Scenarios

**Scenario 1: Add 30 More Vendors (Total: 100)**
- Sync duration: +2-3 hours (total: 8-11 hours)
- Cost impact: +$111/year if all Tier 3 (total: $333/year)
- Recommendation: Prioritize Tier 1/2 vendors

**Scenario 2: Double Order Volume (200/week)**
- Fulfillment load: Minimal impact (cache-based)
- On-demand scraping: May increase slightly
- Recommendation: Monitor cache hit rate

**Scenario 3: Add More Product Fields**
- Database impact: Minimal (n8n Data Tables flexible)
- Workflow impact: Update mapping logic
- Recommendation: Test thoroughly before production

### Performance Optimization

**Database:**
- Index SKU field for fast lookups
- Index vendor_id for sync queries
- Partition sync_log by date

**Workflows:**
- Increase parallel vendor processing
- Implement smart caching
- Optimize Gemini AI prompts

**Cost Optimization:**
- Move Tier 3 vendors to Tier 1/2 when possible
- Reduce sync frequency for stable catalogs
- Negotiate bulk pricing with Zyte

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Technical Team  
**Last Updated:** March 19, 2026

