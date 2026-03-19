# Phase 6 Agent Prompt - Order Fulfillment Workflow (UPDATED FOR HYBRID SCHEMA)

---

## 🎯 Your Mission

Complete **Phase 6: Flow 2 - Order Fulfillment Workflow**. This handles customer orders by retrieving product data and delivering digital assets.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

**Flow 2** is triggered when customer places order:
1. Parse order (from MarketTime email or Shopify webhook)
2. Extract SKUs and customer info
3. Query product_data table for each SKU
4. If SKU not found → On-demand scraping fallback
5. Compile digital asset package
6. Deliver to customer (email/Drive/Dropbox)

**Goal:** <10 seconds for cached SKUs, <60 seconds with on-demand scraping

---

## 🚨 CRITICAL: Hybrid Schema Information

**IMPORTANT:** Phase 5 created a board game/toy-specific product catalog with 31 fields. Phase 6 needs to ADD 4 new fields for media asset collection while preserving all existing fields.

### **Existing product_data Schema (31 fields from Phase 5):**
- `id`, `UPC`, `EAN`, `SKU`, `MSRP`, `MAP`, `Cost`, `Case`, `MOQ`, `Increments_Above_MOQ`
- `Weight_lbs`, `Length_in`, `Width_in`, `Height_in`
- `Shopify_ID`, `ClickUp_ID`, `Name`, `Copy_html`, `Tagline`
- `Ages_text`, `keywords_list`, `Player_Count_text`, `Piece_Count_text`, `Play_Time`
- `BGG_weight`, `Mechanics_list`, `Rules_url`, `Category_list`
- `Shopify_Vendor_Id`, `createdAt`, `updatedAt`

### **NEW Fields to Add (Phase 6 requirements):**
- `image_urls` (JSON) - Array of product image URLs
- `video_urls` (JSON) - Array of product video URLs
- `vendor_id` (Text) - Link to vendor_config table (different from Shopify_Vendor_Id)
- `source_tier` (Number) - Which tier collected this data (1, 2, or 3)

### **Field Mapping for Phase 6:**
- Use `SKU` for lookups (already exists)
- Use `Name` as product_name (already exists)
- Use `Copy_html` as marketing_copy (already exists)
- Use NEW `image_urls` for images (must add)
- Use NEW `video_urls` for videos (must add)
- Use NEW `vendor_id` for vendor routing (must add)
- Use NEW `source_tier` for tier tracking (must add)

---

## 🎯 Phase 6 Deliverables

### **PREREQUISITE: Update product_data Table Schema**

Before building workflows, you MUST add 4 new fields to the existing product_data table in n8n:

1. **Add Field:** `image_urls`
   - Type: JSON
   - Nullable: Yes
   - Description: Array of product image URLs

2. **Add Field:** `video_urls`
   - Type: JSON
   - Nullable: Yes
   - Description: Array of product video URLs

3. **Add Field:** `vendor_id`
   - Type: Text
   - Nullable: Yes
   - Description: Foreign key to vendor_config.vendor_id

4. **Add Field:** `source_tier`
   - Type: Number
   - Nullable: Yes
   - Description: Tier used to collect data (1, 2, or 3)

**Document these changes in:** `docs/schemas/product_data.md` (append to existing schema)

---

### Task 1: Update Phase 5 Workflows to Populate New Fields

**CRITICAL:** Before building Phase 6, you must update Phase 5's data_upserter to populate the new fields.

**Update workflow:** `workflows/flow1/data_upserter.json`

**Changes needed:**
1. When inserting/updating products, include:
   - `vendor_id`: Get from vendor being processed
   - `source_tier`: Get from vendor.tier_used
   - `image_urls`: Get from scraper output (if available)
   - `video_urls`: Get from scraper output (if available)

2. Ensure tier scrapers output these fields:
   - Tier 1 scrapers: Extract image/video URLs from Drive/Dropbox/Sheets
   - Tier 2 scraper: Extract from Shopify product.images array
   - Tier 3 scraper: Extract from Gemini AI extraction

**Test:** Run a test sync for 1 vendor per tier, verify new fields are populated

---

### Task 2: MarketTime Email Trigger

**Create workflow:** `flow2_markettime_trigger.json`

**Trigger:** IMAP email monitor (check every 5 minutes)

**Email Parsing:**
```
From: orders@markettime.com
Subject: Order Confirmation #12345

Customer: John Doe
Email: john@example.com
Order Date: 2026-03-19

Items:
- SKU: ACME-12345, Qty: 2, Vendor: Acme Products
- SKU: BETA-789, Qty: 1, Vendor: Beta Supplies
- SKU: GAMMA-456, Qty: 3, Vendor: Gamma Industries

Total: $299.97
```

**Extraction Logic:**
1. Detect MarketTime confirmation email (subject contains "Order Confirmation")
2. Extract order number from subject
3. Parse email body for:
   - Customer name
   - Customer email
   - Order date
   - Line items (SKU, quantity, vendor)
4. Create order object:
   ```json
   {
     "order_id": "MT-12345",
     "customer_name": "John Doe",
     "customer_email": "john@example.com",
     "order_date": "2026-03-19",
     "items": [
       {"sku": "ACME-12345", "qty": 2, "vendor": "Acme Products"},
       {"sku": "BETA-789", "qty": 1, "vendor": "Beta Supplies"}
     ]
   }
   ```
5. Pass to fulfillment workflow

**Error Handling:**
- If parsing fails, forward email to admin for manual processing
- Log unparseable emails for pattern analysis

---

### Task 3: Shopify Order Webhook Trigger

**Create workflow:** `flow2_shopify_trigger.json`

**Trigger:** Webhook endpoint `/webhook/shopify-orders`

**Webhook Payload:**
```json
{
  "id": 123456789,
  "order_number": 1001,
  "email": "john@example.com",
  "customer": {
    "first_name": "John",
    "last_name": "Doe"
  },
  "line_items": [
    {
      "sku": "ACME-12345",
      "quantity": 2,
      "vendor": "Acme Products"
    }
  ],
  "created_at": "2026-03-19T10:00:00Z"
}
```

**Extraction Logic:**
1. Verify webhook signature (HMAC validation)
2. Extract order data
3. Map to standard order object (same format as MarketTime)
4. Pass to fulfillment workflow

---

### Task 4: SKU Lookup Logic

**Create workflow:** `flow2_sku_lookup.json`

**Purpose:** Retrieve product data for ordered SKUs

**Workflow Steps:**

1. **Input:** Array of SKUs from order

2. **Batch Lookup:**
   ```
   FOR EACH sku IN order.items:
     - Query product_data table WHERE SKU = {sku}
     
     IF found:
       - Add to found_products array
       - Mark as cached_hit
       - Extract: SKU, Name, Copy_html, image_urls, video_urls, vendor_id
     
     ELSE:
       - Add to missing_skus array
       - Mark for on-demand scraping
   ```

3. **Cache Hit Rate Tracking:**
   - Log: "Cache hits: {found}/{total} ({percentage}%)"
   - Goal: >90% cache hit rate

4. **Output:**
   ```json
   {
     "found_products": [
       {
         "sku": "ACME-12345",
         "name": "Premium Widget Pro",
         "description": "<p>Marketing copy...</p>",
         "image_urls": ["url1", "url2"],
         "video_urls": ["url3"],
         "vendor_id": "vendor_001"
       }
     ],
     "missing_skus": ["SKU-123", "SKU-456"],
     "cache_hit_rate": 0.85
   }
   ```

---

### Task 5: On-Demand Scraping Fallback

**Create workflow:** `flow2_on_demand_scraper.json`

**Purpose:** Scrape missing SKUs in real-time

**Workflow Steps:**

1. **Input:** Array of missing SKUs

2. **For Each Missing SKU:**
   ```
   - Lookup vendor from vendor_config (match by SKU pattern or vendor name from order)
   
   IF vendor found:
     - Get vendor.tier_used and data_source_type
     - Route to appropriate scraper:
       - Tier 1: Call Drive/Dropbox/Sheets scraper
       - Tier 2: Call Shopify scraper with specific SKU
       - Tier 3: Call Zyte+AI scraper
     - Collect product data including image_urls, video_urls
     - Save result to product_data table with vendor_id and source_tier
     - Add to found_products array
   
   ELSE:
     - Log error: "Cannot determine vendor for SKU {sku}"
     - Add to failed_skus array
   ```

3. **Optimization:**
   - For Shopify (Tier 2): Use `/products/{handle}.json` for single product
   - For Drive/Dropbox (Tier 1): Search for SKU in folder names
   - For Zyte+AI (Tier 3): Scrape product page directly if URL pattern known

4. **Timeout:**
   - Max 30 seconds per SKU
   - If timeout, mark as failed and continue

5. **Output:**
   ```json
   {
     "scraped_products": [...],
     "failed_skus": ["SKU-999"],
     "scraping_duration": 45
   }
   ```

---

### Task 6: Digital Asset Package Compiler

**Create workflow:** `flow2_asset_compiler.json`

**Purpose:** Collect all images, videos, and copy for ordered products

**Workflow Steps:**

1. **Input:** Array of products (from cache + on-demand scraping)

2. **For Each Product:**
   ```
   - Collect image_urls (from new field)
   - Collect video_urls (from new field)
   - Collect Copy_html (existing field, use as marketing copy)
   - Collect Name (existing field)
   - Collect SKU (existing field)
   ```

3. **Create Package:**
   ```json
   {
     "order_id": "MT-12345",
     "customer": {
       "name": "John Doe",
       "email": "john@example.com"
     },
     "products": [
       {
         "sku": "ACME-12345",
         "name": "Premium Widget Pro",
         "quantity": 2,
         "images": ["url1", "url2"],
         "videos": ["url3"],
         "description": "<p>Marketing copy...</p>",
         "download_links": {
           "images_zip": "https://...",
           "videos_zip": "https://..."
         }
       }
     ],
     "package_created_at": "2026-03-19T10:05:00Z"
   }
   ```

4. **Optional: Create ZIP Files:**
   - If delivery method is download link
   - Zip all images for each product
   - Zip all videos for each product
   - Upload to temporary storage (Google Drive/Dropbox)
   - Generate shareable links (expire in 7 days)

---

### Task 7: Customer Delivery Module

**Create workflow:** `flow2_customer_delivery.json`

**Purpose:** Deliver digital assets to customer

**Delivery Methods:**

**Method 1: Email Delivery**
```
To: {customer_email}
Subject: Your Product Assets - Order #{order_id}

Hi {customer_name},

Your product assets are ready! Here's what's included:

Product: {product_name} (SKU: {sku})
Quantity: {qty}

Images:
- {image_url_1}
- {image_url_2}

Videos:
- {video_url_1}

Description:
{Copy_html converted to plain text}

---

[Repeat for each product]

Thank you for your order!
```

**Method 2: Google Drive Delivery**
1. Create folder: `/Customer_Orders/{order_id}/`
2. For each product:
   - Create subfolder: `/{sku}/`
   - Download images and upload to Drive
   - Download videos and upload to Drive
   - Create `description.txt` with Copy_html (converted to plain text)
3. Share folder with customer email
4. Send email with Drive folder link

**Method 3: Dropbox Delivery**
- Similar to Drive, but use Dropbox API

**Method 4: Download Link**
- Create ZIP files (from Task 6)
- Upload to temporary storage
- Send email with download links

**Configuration:**
- Delivery method set in environment variable: `DELIVERY_METHOD`
- Default: email (fastest, no file hosting needed)

---

### Task 8: Failure Handling

**Create workflow:** `flow2_failure_handler.json`

**Failure Scenarios:**

1. **SKU Not Found (after on-demand scraping):**
   - Send alert to admin
   - Email customer: "We're preparing your assets, will send within 24 hours"
   - Create manual task for admin to fulfill

2. **Scraping Timeout:**
   - Log error
   - Retry once
   - If still fails, escalate to admin

3. **Delivery Failure:**
   - Retry email send (max 3 attempts)
   - If email fails, log error and alert admin
   - Store package for manual delivery

**Admin Alert Format:**
```
Subject: Order Fulfillment Failed - #{order_id}

Order: {order_id}
Customer: {customer_name} ({customer_email})
Failed SKUs: {sku_list}
Error: {error_message}

Action Required:
1. Manually locate product assets for SKUs: {sku_list}
2. Send to customer at: {customer_email}
3. Update product_data table with correct data

Order Details: {link_to_order}
```

---

## 📁 Deliverable Files

```
vendor-products/
├── docs/
│   └── schemas/
│       └── product_data.md (UPDATE with 4 new fields)
├── workflows/
│   ├── flow1/
│   │   └── data_upserter.json (UPDATE to populate new fields)
│   ├── flow2/
│   │   ├── markettime_trigger.json
│   │   ├── shopify_trigger.json
│   │   ├── sku_lookup.json
│   │   ├── on_demand_scraper.json
│   │   ├── asset_compiler.json
│   │   ├── customer_delivery.json
│   │   └── failure_handler.json
│   └── tests/
│       ├── test_order_fulfillment.json
│       └── test_on_demand_scraping.json
└── docs/
    └── workflows/
        └── FLOW2_ORDER_FULFILLMENT.md
```

---

## ✅ Completion Checklist

**Schema Updates:**
- [ ] 4 new fields added to product_data table in n8n
- [ ] Schema documentation updated

**Phase 5 Updates:**
- [ ] data_upserter.json updated to populate new fields
- [ ] Tier scrapers output image_urls, video_urls
- [ ] Test sync confirms new fields populated

**Phase 6 Workflows:**
- [ ] MarketTime email trigger created and tested
- [ ] Shopify webhook trigger created and tested
- [ ] SKU lookup workflow created
- [ ] On-demand scraping workflow created
- [ ] Asset compiler workflow created
- [ ] Customer delivery workflow created (all 4 methods)
- [ ] Failure handler workflow created
- [ ] Cache hit rate >90% (with Phase 5 data)
- [ ] Fulfillment time <10s for cached SKUs
- [ ] Fulfillment time <60s with on-demand scraping
- [ ] Test workflows created
- [ ] Documentation created
- [ ] All workflows exported and committed

---

## 📊 Success Metrics

- **Time to Complete:** 4-6 hours (includes Phase 5 updates)
- **Workflows Created:** 7 (Phase 6) + 1 updated (Phase 5)
- **Fulfillment Speed:** <10s (cached), <60s (on-demand)
- **Cache Hit Rate:** >90%
- **Delivery Success Rate:** >99%
- **Manual Intervention Rate:** <5%

---

## 🚀 Next Steps

- **Phase 7:** Testing & Validation

---

## 💡 Implementation Tips

1. **Start with schema updates** - Add 4 fields to product_data table first
2. **Update Phase 5 next** - Ensure new fields are populated before building Phase 6
3. **Test with sample orders** - Create test MarketTime emails and Shopify webhooks
4. **Monitor cache hit rate** - If low, improve Phase 5 sync coverage
5. **Optimize on-demand scraping** - Cache results immediately for future orders
6. **Choose delivery method wisely** - Email is fastest, Drive/Dropbox better for large files
7. **Set up alerts** - Know immediately when fulfillment fails
8. **Log everything** - Track fulfillment times and success rates

---

**Phase:** 6 of 8  
**Dependencies:** Phase 5 (data must be populated) + Schema updates  
**Estimated Time:** 4-6 hours  
**Critical Path:** Yes (blocks Phase 7)

