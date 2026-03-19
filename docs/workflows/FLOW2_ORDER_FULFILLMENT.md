# Flow 2: Order Fulfillment Workflow

## Overview

Flow 2 handles customer order fulfillment by retrieving product data from the cache, performing on-demand scraping for missing SKUs, compiling digital asset packages, and delivering them to customers.

**Performance Goals:**
- **Cached SKUs**: <10 seconds fulfillment time
- **On-demand scraping**: <60 seconds fulfillment time
- **Cache hit rate**: >90%

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ORDER TRIGGERS                            │
├─────────────────────────────────────────────────────────────────┤
│  MarketTime Email    │    Shopify Webhook                       │
│  (IMAP Monitor)      │    (HMAC Verified)                       │
└──────────┬───────────┴──────────┬──────────────────────────────┘
           │                      │
           └──────────┬───────────┘
                      ▼
           ┌──────────────────────┐
           │  SKU Lookup Module   │
           │  (Query product_data)│
           └──────────┬───────────┘
                      │
           ┌──────────┴───────────┐
           │                      │
           ▼                      ▼
    ┌──────────┐          ┌──────────────┐
    │  Found   │          │   Missing    │
    │  (Cache  │          │   (Cache     │
    │   Hit)   │          │    Miss)     │
    └────┬─────┘          └──────┬───────┘
         │                       │
         │                       ▼
         │              ┌─────────────────┐
         │              │  On-Demand      │
         │              │  Scraper        │
         │              │  (Tier 1/2/3)   │
         │              └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ▼
          ┌──────────────────────┐
          │  Asset Compiler      │
          │  (Package Builder)   │
          └──────────┬───────────┘
                     ▼
          ┌──────────────────────┐
          │  Customer Delivery   │
          │  (Email/Drive/etc)   │
          └──────────────────────┘
```

---

## Workflow Components

### 1. MarketTime Email Trigger
**File**: `workflows/flow2/markettime_trigger.json`

**Purpose**: Monitor IMAP inbox for MarketTime order confirmation emails

**Trigger**: IMAP email check every 5 minutes

**Email Format Expected**:
```
From: orders@markettime.com
Subject: Order Confirmation #12345

Customer: John Doe
Email: john@example.com
Order Date: 2026-03-19

Items:
- SKU: ACME-12345, Qty: 2, Vendor: Acme Products
- SKU: BETA-789, Qty: 1, Vendor: Beta Supplies

Total: $299.97
```

**Processing Steps**:
1. Filter emails from `orders@markettime.com` with "Order Confirmation" in subject
2. Extract order number from subject line
3. Parse email body for customer info and line items
4. Validate extracted data (email format, SKU presence, quantities)
5. If valid → Trigger fulfillment workflow
6. If invalid → Send admin alert email

**Output Format**:
```json
{
  "order_id": "MT-12345",
  "source": "markettime",
  "customer_name": "John Doe",
  "customer_email": "john@example.com",
  "order_date": "2026-03-19",
  "items": [
    {"sku": "ACME-12345", "qty": 2, "vendor": "Acme Products"},
    {"sku": "BETA-789", "qty": 1, "vendor": "Beta Supplies"}
  ],
  "total_items": 2,
  "total_quantity": 3
}
```

---

### 2. Shopify Order Webhook Trigger
**File**: `workflows/flow2/shopify_trigger.json`

**Purpose**: Receive and process Shopify order webhooks

**Trigger**: Webhook endpoint `/webhook/shopify-orders`

**Security**: HMAC-SHA256 signature verification using `SHOPIFY_WEBHOOK_SECRET`

**Webhook Payload** (Shopify format):
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
      "vendor": "Acme Products",
      "product_id": 789,
      "variant_id": 456
    }
  ],
  "financial_status": "paid",
  "created_at": "2026-03-19T10:00:00Z"
}
```

**Processing Steps**:
1. Verify HMAC signature (reject if invalid)
2. Extract customer and line item data
3. Filter out items without valid SKUs
4. Validate payment status (must be "paid" or "authorized")
5. Map to standard order format (same as MarketTime)
6. If valid → Trigger fulfillment workflow
7. If invalid → Send admin alert and return 400 error
8. Return 200 OK response to Shopify

---

### 3. SKU Lookup Module
**File**: `workflows/flow2/sku_lookup.json`

**Purpose**: Query `product_data` table for ordered SKUs

**Input**:
```json
{
  "skus": ["ACME-12345", "BETA-789", "GAMMA-456"]
}
```

**Processing Steps**:
1. Split SKU array into individual items
2. For each SKU:
   - Query `product_data` table WHERE `SKU = {sku}`
   - If found → Extract product data (name, description, image_urls, video_urls)
   - If not found → Mark as cache miss
3. Aggregate results and calculate cache hit rate
4. Log performance metrics

**Output**:
```json
{
  "found_products": [
    {
      "sku": "ACME-12345",
      "name": "Premium Widget Pro",
      "description": "<p>Marketing copy...</p>",
      "image_urls": ["url1", "url2"],
      "video_urls": ["url3"],
      "vendor_id": "vendor_001",
      "source_tier": 2,
      "cache_hit": true
    }
  ],
  "missing_skus": ["GAMMA-456"],
  "total_skus": 3,
  "cache_hits": 2,
  "cache_misses": 1,
  "cache_hit_rate": 0.67,
  "cache_hit_percentage": "66.67%"
}
```

**Performance Tracking**:
- Logs cache hit rate for monitoring
- Warns if cache hit rate < 90%
- Goal: >90% cache hit rate

---

### 4. On-Demand Scraper
**File**: `workflows/flow2/on_demand_scraper.json`

**Purpose**: Scrape missing SKUs in real-time using appropriate tier scrapers

**Input**:
```json
{
  "missing_skus": ["GAMMA-456"],
  "order_context": {
    "order_id": "MT-12345"
  }
}
```

**Processing Steps**:
1. Load all vendor configurations from `vendor_config` table
2. For each missing SKU:
   - Match SKU to vendor (by SKU prefix or vendor name pattern)
   - If vendor not found → Mark as failed
   - If vendor found → Route to appropriate tier scraper:
     - **Tier 1**: Call Drive/Dropbox/Sheets scraper
     - **Tier 2**: Call Shopify single-product scraper
     - **Tier 3**: Call Zyte+AI scraper (60s timeout)
3. Process scraper responses
4. Save successfully scraped products to `product_data` table
5. Aggregate results

**Tier Routing Logic**:
```javascript
// Match vendor by SKU prefix
if (sku.startsWith(vendor.sku_prefix)) {
  matchedVendor = vendor;
}

// Or match by vendor name in SKU
if (sku.toLowerCase().includes(vendor.vendor_name.toLowerCase())) {
  matchedVendor = vendor;
}
```

**Output**:
```json
{
  "scraped_products": [
    {
      "sku": "GAMMA-456",
      "name": "Product Name",
      "description": "...",
      "image_urls": ["..."],
      "video_urls": ["..."],
      "vendor_id": "vendor_003",
      "source_tier": 3,
      "scrape_success": true,
      "duration_ms": 45000
    }
  ],
  "failed_skus": [],
  "total_attempted": 1,
  "successful_scrapes": 1,
  "failed_scrapes": 0,
  "success_rate": 1.0,
  "total_duration_ms": 45000,
  "avg_duration_ms": 45000
}
```

**Timeout Handling**:
- Tier 1/2: 30 second timeout
- Tier 3: 60 second timeout (Zyte+AI is slower)
- If timeout → Mark as failed and continue

---

### 5. Asset Compiler
**File**: `workflows/flow2/asset_compiler.json`

**Purpose**: Compile digital asset packages for customer delivery

**Input**:
```json
{
  "order": {
    "order_id": "MT-12345",
    "customer_name": "John Doe",
    "customer_email": "john@example.com"
  },
  "products": [
    {
      "sku": "ACME-12345",
      "name": "Premium Widget Pro",
      "description": "<p>Marketing copy...</p>",
      "image_urls": ["url1", "url2"],
      "video_urls": ["url3"],
      "quantity": 2
    }
  ]
}
```

**Processing Steps**:
1. For each product:
   - Parse `image_urls` and `video_urls` (handle JSON strings)
   - Convert HTML description to plain text
   - Extract tagline and other metadata
   - Count total assets (images + videos)
2. Aggregate all products into single package
3. Calculate total assets across all products

**Output**:
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
      "description": "Marketing copy...",
      "description_html": "<p>Marketing copy...</p>",
      "asset_count": 3
    }
  ],
  "total_products": 1,
  "total_assets": 3,
  "package_created_at": "2026-03-19T10:05:00Z"
}
```

---

### 6. Customer Delivery Module
**File**: `workflows/flow2/customer_delivery.json`

**Purpose**: Deliver digital assets to customers via email

**Delivery Method**: Email (configurable via `DELIVERY_METHOD` env var)

**Input**: Asset package from compiler

**Email Template**:
```html
<h2>Your Product Assets - Order #MT-12345</h2>
<p>Hi John Doe,</p>
<p>Your product assets are ready! Here's what's included:</p>

<hr>
<h3>Premium Widget Pro (SKU: ACME-12345)</h3>
<p><strong>Quantity:</strong> 2</p>

<p><strong>Images:</strong></p>
<ul>
  <li><a href="url1">url1</a></li>
  <li><a href="url2">url2</a></li>
</ul>

<p><strong>Videos:</strong></p>
<ul>
  <li><a href="url3">url3</a></li>
</ul>

<p><strong>Description:</strong></p>
<p>Marketing copy...</p>

<hr>
<p>Thank you for your order!</p>
```

**Processing Steps**:
1. Format email body with product details and asset links
2. Send email to customer
3. Check delivery status
4. If success → Mark as delivered
5. If failure → Mark as failed (triggers failure handler)

**Output**:
```json
{
  "delivery_success": true,
  "order_id": "MT-12345",
  "customer_email": "john@example.com",
  "delivered_at": "2026-03-19T10:06:00Z"
}
```

---

## Phase 6 Schema Changes

### New Fields Added to `product_data` Table

| Field | Type | Description |
|-------|------|-------------|
| `image_urls` | JSON | Array of product image URLs |
| `video_urls` | JSON | Array of product video URLs |
| `vendor_id` | Text | Foreign key to vendor_config table |
| `source_tier` | Number | Tier used to collect data (1, 2, or 3) |

### Updated Phase 5 Data Upserter

**File**: `workflows/flow1/data_upserter.json`

**Changes**:
- Added logic to extract image/video URLs from various sources
- Populates `image_urls`, `video_urls`, `vendor_id`, `source_tier` fields
- Handles multiple input formats (arrays, JSON strings, single URLs)

**Image/Video URL Extraction**:
```javascript
// Extract from various possible sources
if (productData.images && Array.isArray(productData.images)) {
  imageUrls = productData.images.map(img => 
    typeof img === 'string' ? img : img.src || img.url
  );
} else if (productData.image_urls) {
  imageUrls = productData.image_urls;
} else if (productData.image_url) {
  imageUrls = [productData.image_url];
}
```

---

## Environment Variables

### Required Configuration

```bash
# Email Configuration
ADMIN_EMAIL=admin@example.com
DELIVERY_EMAIL=noreply@example.com

# Delivery Method
DELIVERY_METHOD=email  # Options: email, drive, dropbox, download_link

# Shopify Webhook Security
SHOPIFY_WEBHOOK_SECRET=your_shopify_webhook_secret

# n8n Webhook Base URL
N8N_WEBHOOK_BASE_URL=https://your-n8n-instance.com
```

---

## Performance Metrics

### Target Performance

| Metric | Target | Measurement |
|--------|--------|-------------|
| Cache Hit Rate | >90% | SKU Lookup Module |
| Cached SKU Fulfillment | <10 seconds | End-to-end time |
| On-Demand Scraping | <60 seconds | Including Tier 3 |
| Email Delivery | <5 seconds | SMTP send time |

### Monitoring

**Cache Performance**:
```javascript
console.log('=== SKU Lookup Performance ===');
console.log(`Cache Hit Rate: ${cache_hit_percentage}`);
if (cache_hit_rate < 0.9) {
  console.warn('⚠️ Cache hit rate below 90% target!');
}
```

**Scraping Performance**:
```javascript
console.log('=== On-Demand Scraping ===');
console.log(`Success Rate: ${success_rate}`);
console.log(`Avg Duration: ${avg_duration_ms}ms`);
```

---

## Error Handling

### Failure Scenarios

1. **SKU Not Found (After On-Demand Scraping)**
   - Send alert to admin
   - Email customer: "We're preparing your assets, will send within 24 hours"
   - Create manual task for admin

2. **Scraping Timeout**
   - Log error
   - Retry once
   - If still fails → Escalate to admin

3. **Delivery Failure**
   - Retry email send (max 3 attempts)
   - If email fails → Log error and alert admin
   - Store package for manual delivery

### Admin Alert Format

```
Subject: Order Fulfillment Failed - #MT-12345

Order: MT-12345
Customer: John Doe (john@example.com)
Failed SKUs: GAMMA-456
Error: Cannot determine vendor for SKU

Action Required:
1. Manually locate product assets for SKUs: GAMMA-456
2. Send to customer at: john@example.com
3. Update product_data table with correct data

Order Details: [link]
```

---

## Testing

### Test Scenarios

1. **Happy Path - All SKUs Cached**
   - Input: Order with 3 SKUs, all in cache
   - Expected: <10 second fulfillment, 100% cache hit

2. **Mixed - Some SKUs Missing**
   - Input: Order with 3 SKUs, 1 missing
   - Expected: On-demand scraping triggered, <60 second fulfillment

3. **All SKUs Missing**
   - Input: Order with 3 SKUs, all missing
   - Expected: All scraped on-demand, <60 second fulfillment

4. **Vendor Not Found**
   - Input: SKU with unknown vendor
   - Expected: Admin alert, customer notified of delay

5. **Email Delivery Failure**
   - Input: Invalid customer email
   - Expected: Retry attempts, admin alert

---

## Deployment Checklist

### Prerequisites

- [ ] n8n Cloud instance running
- [ ] `product_data` table updated with 4 new fields
- [ ] `vendor_config` table populated
- [ ] Phase 5 data_upserter updated and tested
- [ ] Email SMTP credentials configured
- [ ] Shopify webhook secret configured

### Import Workflows

1. Import all Flow 2 workflows to n8n:
   - `markettime_trigger.json`
   - `shopify_trigger.json`
   - `sku_lookup.json`
   - `on_demand_scraper.json`
   - `asset_compiler.json`
   - `customer_delivery.json`

2. Configure credentials:
   - IMAP (MarketTime email)
   - SMTP (Admin alerts)
   - SMTP (Customer delivery)

3. Set environment variables in n8n

4. Activate workflows

### Testing

1. Send test MarketTime email
2. Trigger test Shopify webhook
3. Verify SKU lookup performance
4. Test on-demand scraping
5. Verify customer email delivery

---

## Future Enhancements

### Planned Features

1. **Google Drive Delivery**
   - Create folder per order
   - Upload images/videos to Drive
   - Share folder with customer

2. **Dropbox Delivery**
   - Similar to Drive delivery
   - Use Dropbox API

3. **Download Link Delivery**
   - Create ZIP files for images/videos
   - Upload to temporary storage
   - Generate expiring download links

4. **Retry Logic**
   - Automatic retry for failed deliveries
   - Exponential backoff

5. **Order Status Tracking**
   - Create `order_fulfillment` table
   - Track order status (pending, processing, delivered, failed)
   - Customer-facing status page

---

## Related Documentation

- [Product Data Schema](../schemas/product_data.md)
- [Vendor Config Schema](../schemas/vendor_config.md)
- [Flow 1: Weekly Sync Workflow](FLOW1_WEEKLY_SYNC.md)
- [Implementation Plan](../../IMPLEMENTATION_PLAN.md)

