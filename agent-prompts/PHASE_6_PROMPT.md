# Phase 6 Agent Prompt - Order Fulfillment Workflow

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

## 🎯 Phase 6 Deliverables

### Task 1: MarketTime Email Trigger

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

### Task 2: Shopify Order Webhook Trigger

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

### Task 3: SKU Lookup Logic

**Create workflow:** `flow2_sku_lookup.json`

**Purpose:** Retrieve product data for ordered SKUs

**Workflow Steps:**

1. **Input:** Array of SKUs from order

2. **Batch Lookup:**
   ```
   FOR EACH sku IN order.items:
     - Query product_data table WHERE sku = {sku}
     
     IF found:
       - Add to found_products array
       - Mark as cached_hit
     
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
     "found_products": [...],
     "missing_skus": ["SKU-123", "SKU-456"],
     "cache_hit_rate": 0.85
   }
   ```

### Task 4: On-Demand Scraping Fallback

**Create workflow:** `flow2_on_demand_scraper.json`

**Purpose:** Scrape missing SKUs in real-time

**Workflow Steps:**

1. **Input:** Array of missing SKUs

2. **For Each Missing SKU:**
   ```
   - Lookup vendor_id from vendor_config (match by SKU pattern or vendor name)
   
   IF vendor found:
     - Get vendor.tier_used and data_source_type
     - Route to appropriate scraper
     - Run scraper for specific SKU (not full catalog)
     - Save result to product_data table
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

### Task 5: Digital Asset Package Compiler

**Create workflow:** `flow2_asset_compiler.json`

**Purpose:** Collect all images, videos, and copy for ordered products

**Workflow Steps:**

1. **Input:** Array of products (from cache + on-demand scraping)

2. **For Each Product:**
   ```
   - Collect image_urls
   - Collect video_urls
   - Collect marketing_copy
   - Collect metadata
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
         "description": "Marketing copy here...",
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

### Task 6: Customer Delivery Module

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
{marketing_copy}

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
   - Create `description.txt` with marketing copy
3. Share folder with customer email
4. Send email with Drive folder link

**Method 3: Dropbox Delivery**
- Similar to Drive, but use Dropbox API

**Method 4: Download Link**
- Create ZIP files (from Task 5)
- Upload to temporary storage
- Send email with download links

**Configuration:**
- Delivery method set in environment variable: `DELIVERY_METHOD`
- Default: email (fastest, no file hosting needed)

### Task 7: Failure Handling

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
├── workflows/
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

## 📖 Documentation Requirements

Create `docs/workflows/FLOW2_ORDER_FULFILLMENT.md`:

1. **Overview:** How order fulfillment works
2. **Trigger Sources:** MarketTime email vs Shopify webhook
3. **SKU Lookup:** Cache hits vs on-demand scraping
4. **Delivery Methods:** Email, Drive, Dropbox, Download links
5. **Performance:** Target times and optimization tips
6. **Failure Handling:** What happens when things go wrong
7. **Testing:** How to test with sample orders

---

## ✅ Completion Checklist

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

- **Time to Complete:** 8-10 hours
- **Workflows Created:** 7
- **Fulfillment Speed:** <10s (cached), <60s (on-demand)
- **Cache Hit Rate:** >90%
- **Delivery Success Rate:** >99%
- **Manual Intervention Rate:** <5%

---

## 🚀 Next Steps

- **Phase 7:** Testing & Validation

---

## 💡 Implementation Tips

1. **Test with sample orders first:** Create test MarketTime emails and Shopify webhooks
2. **Monitor cache hit rate:** If low, improve Phase 5 sync coverage
3. **Optimize on-demand scraping:** Cache results immediately for future orders
4. **Choose delivery method wisely:** Email is fastest, Drive/Dropbox better for large files
5. **Set up alerts:** Know immediately when fulfillment fails
6. **Log everything:** Track fulfillment times and success rates

---

**Phase:** 6 of 8  
**Dependencies:** Phase 5 (data must be populated)  
**Estimated Time:** 8-10 hours  
**Critical Path:** Yes (blocks Phase 7)

