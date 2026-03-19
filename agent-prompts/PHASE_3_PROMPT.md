# Phase 3 Agent Prompt - Tier 2 Shopify Public Endpoint Scraper

---

## 🎯 Your Mission

You are tasked with completing **Phase 3: Tier 2 - Shopify Public Endpoint Scraper** for the Vendor Product Data Collection System. You will build n8n workflows that collect product data from Shopify stores using their free public JSON endpoints.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

### What This System Does

This system automates collection of product data from 70+ vendors. **Tier 2 vendors** are Shopify stores that expose product data through public `/products.json` endpoints - no authentication required!

### Tech Stack

- **Orchestration:** n8n Cloud
- **Data Storage:** n8n Data Tables
- **Tier 2 Method:** HTTP requests to Shopify public JSON endpoints (FREE)

### Tier 2 Strategy

**Shopify Public Endpoints:**
- URL Pattern: `https://{store}.myshopify.com/products.json`
- Pagination: `?page={n}` or `?limit=250&page={n}`
- No authentication required
- Rate limit: ~2 requests/second per store
- Returns: Product title, handle, images, variants, description

**Cost:** $0 (public endpoints)

---

## 🗂️ Data Models (From Phase 1)

### Product Data Table Schema

```
sku (Text, PRIMARY KEY)
vendor_id (Text, FOREIGN KEY)
product_name (Text)
image_urls (JSON array)
video_urls (JSON array)
marketing_copy (Text)
metadata (JSON)
last_updated (DateTime)
source_tier (Number) - Always 2 for this phase
```

---

## 🎯 Phase 3 Deliverables (Your Tasks)

### Task 1: Shopify Store Detection Workflow

**Create n8n workflow:** `tier2_shopify_detector.json`

**Purpose:** Verify if a URL is a Shopify store and test endpoint availability

**Workflow Steps:**
1. **Input:** Receive vendor source_url
2. **Parse URL:** Extract store domain
3. **Test Endpoint:** HTTP GET to `/products.json`
4. **Validate Response:**
   - Check for `products` array in JSON
   - Verify response status is 200
   - Check if pagination is available
5. **Detect Store Type:**
   - Standard Shopify: `{store}.myshopify.com`
   - Custom domain: Check for Shopify headers (`X-ShopId`)
6. **Output:** Boolean (is_shopify) + store metadata

**Error Handling:**
- Handle 404 (not a Shopify store or endpoint disabled)
- Handle 429 (rate limited - wait and retry)
- Handle custom domains (may need different URL patterns)

### Task 2: Shopify Catalog Scraper Workflow

**Create n8n workflow:** `tier2_shopify_scraper.json`

**Workflow Steps:**

1. **Input:** Receive vendor_id and source_url

2. **Initialize:**
   - Set page = 1
   - Set products_collected = []
   - Set rate_limit_delay = 500ms

3. **Fetch Products Loop:**
   ```
   WHILE has_more_products:
     - HTTP GET: {source_url}/products.json?limit=250&page={page}
     - Parse JSON response
     - Extract products array
     - Add to products_collected
     - Increment page
     - Wait {rate_limit_delay} ms
     - Check if response has fewer than 250 products (last page)
   ```

4. **Parse Each Product:**
   ```json
   {
     "id": 123456789,
     "title": "Premium Widget Pro",
     "handle": "premium-widget-pro",
     "body_html": "<p>Product description...</p>",
     "vendor": "Acme Products",
     "product_type": "Widgets",
     "tags": ["premium", "widget"],
     "variants": [
       {
         "id": 987654321,
         "sku": "ACME-12345",
         "price": "49.99",
         "inventory_quantity": 100
       }
     ],
     "images": [
       {
         "src": "https://cdn.shopify.com/s/files/.../image1.jpg"
       }
     ]
   }
   ```

5. **Extract Data:**
   - **SKU:** From `variants[].sku` (handle multiple variants)
   - **Product Name:** From `title`
   - **Image URLs:** From `images[].src` array
   - **Video URLs:** Parse from `body_html` (look for `<video>` or YouTube embeds)
   - **Marketing Copy:** Strip HTML from `body_html`
   - **Metadata:** Store handle, product_type, tags, variants info

6. **Handle Variants:**
   - If product has multiple variants with different SKUs, create separate product records
   - Link variants in metadata: `{"parent_product_id": 123, "variant_id": 987}`

7. **Normalize Data:** Map to product_data table schema

8. **Output:** Return array of product objects

**Shopify-Specific Considerations:**
- **Rate Limiting:** Max 2 requests/second per store
- **Pagination:** Shopify limits to 250 products per page
- **Variants:** One product can have multiple SKUs (variants)
- **Images:** CDN URLs are permanent and publicly accessible
- **Videos:** Not directly in JSON, must parse from HTML description

### Task 3: SKU Mapping Database

**Create n8n workflow:** `tier2_sku_mapper.json`

**Purpose:** Build and maintain SKU → Shopify handle mapping for fast lookups

**Workflow Steps:**
1. **Input:** Array of products from scraper
2. **Create Mappings:**
   ```json
   {
     "sku": "ACME-12345",
     "shopify_handle": "premium-widget-pro",
     "shopify_product_id": 123456789,
     "shopify_variant_id": 987654321,
     "store_url": "https://store.myshopify.com",
     "last_updated": "2026-03-19T10:00:00Z"
   }
   ```
3. **Store in Metadata:** Add to product_data.metadata field
4. **Enable Fast Lookup:** When order comes in with SKU, can quickly find Shopify product

**Use Case:** Order fulfillment can use handle to fetch latest product data if needed

### Task 4: HTML to Text Converter

**Create n8n subworkflow:** `tier2_html_cleaner.json`

**Purpose:** Convert Shopify's HTML descriptions to clean marketing copy

**Workflow Steps:**
1. **Input:** HTML string from `body_html`
2. **Extract Videos:**
   - Find `<iframe>` tags (YouTube, Vimeo embeds)
   - Find `<video>` tags with `src` attributes
   - Extract URLs and add to video_urls array
3. **Strip HTML:**
   - Remove all HTML tags
   - Convert `<br>` to newlines
   - Convert `<p>` to paragraphs with double newlines
   - Decode HTML entities (`&amp;` → `&`)
4. **Clean Text:**
   - Trim whitespace
   - Remove excessive newlines (max 2 consecutive)
   - Limit to 5000 characters
5. **Output:** Clean text + extracted video URLs

**Example:**
```
Input: "<p>The <strong>Premium Widget Pro</strong> revolutionizes...<br><iframe src='https://youtube.com/embed/xyz'></iframe></p>"

Output: {
  "text": "The Premium Widget Pro revolutionizes...",
  "video_urls": ["https://youtube.com/embed/xyz"]
}
```

### Task 5: Testing & Validation

**Create test workflows:**

1. **Test Shopify Detection:**
   - Test with known Shopify store: `https://shop.tesla.com`
   - Test with non-Shopify site: `https://amazon.com`
   - Test with custom domain Shopify store
   - Verify detection accuracy

2. **Test Catalog Scraping:**
   - Test with small store (<50 products)
   - Test with large store (>250 products, requires pagination)
   - Test with store that has variants
   - Verify all products extracted

3. **Test SKU Mapping:**
   - Verify SKU → handle mapping created
   - Test lookup by SKU
   - Verify metadata stored correctly

4. **Test HTML Cleaning:**
   - Test with simple HTML
   - Test with complex HTML (tables, lists, embeds)
   - Test with video embeds
   - Verify clean output

**Validation Checklist:**
- [ ] Shopify detection works for all store types
- [ ] Pagination handles stores with 250+ products
- [ ] All product variants extracted as separate SKUs
- [ ] Image URLs are valid and accessible
- [ ] Video URLs extracted from HTML
- [ ] Marketing copy is clean (no HTML tags)
- [ ] SKU mappings created correctly
- [ ] Rate limiting respected (no 429 errors)

---

## 📁 Deliverable Files

```
vendor-products/
├── workflows/
│   ├── tier2/
│   │   ├── shopify_detector.json
│   │   ├── shopify_scraper.json
│   │   ├── sku_mapper.json
│   │   └── html_cleaner.json
│   └── tests/
│       ├── test_shopify_detector.json
│       ├── test_shopify_scraper.json
│       └── test_html_cleaner.json
└── docs/
    └── workflows/
        └── TIER2_SHOPIFY.md
```

---

## 📖 Documentation Requirements

Create `docs/workflows/TIER2_SHOPIFY.md` with:

1. **Overview:** How Shopify public endpoints work
2. **Endpoint Documentation:** `/products.json` structure and pagination
3. **Workflow Architecture:** How detector, scraper, and mapper work together
4. **Rate Limiting:** How to respect Shopify's limits
5. **Variant Handling:** How products with multiple SKUs are processed
6. **Video Extraction:** How videos are found in HTML descriptions
7. **Testing Guide:** How to test with real Shopify stores
8. **Troubleshooting:** Common issues (rate limits, custom domains, etc.)

---

## ✅ Completion Checklist

- [ ] Shopify detector workflow created and tested
- [ ] Shopify scraper workflow created and tested
- [ ] SKU mapper workflow created
- [ ] HTML cleaner subworkflow created and tested
- [ ] Pagination handling works for large catalogs
- [ ] Variant handling creates separate SKU records
- [ ] Rate limiting implemented (500ms delay between requests)
- [ ] Video extraction from HTML works
- [ ] Test workflows created
- [ ] Documentation created
- [ ] All workflows exported as JSON
- [ ] All files committed to repository

---

## 📊 Success Metrics

- **Time to Complete:** 4-6 hours
- **Workflows Created:** 4 (detector, scraper, mapper, cleaner)
- **Test Workflows:** 3
- **Rate Limit Compliance:** 100% (no 429 errors)
- **Variant Handling:** Correctly splits multi-variant products
- **Video Extraction:** Successfully extracts YouTube/Vimeo embeds

---

## 🚀 Next Steps After Phase 3

- **Phase 4:** Tier 3 Scrapers (Zyte + AI) - can run in parallel
- **Phase 5:** Weekly Sync Workflow (requires Phases 2, 3, 4 complete)

---

## 💡 Implementation Tips

1. **Test with real stores:** Use public Shopify stores like `shop.tesla.com`, `shop.kylie.com`
2. **Handle pagination carefully:** Some stores have 1000+ products
3. **Respect rate limits:** Always add delays between requests
4. **Parse HTML carefully:** Use n8n's HTML node or regex for video extraction
5. **Log everything:** Track which products/variants are processed
6. **Handle errors gracefully:** Some stores may block scrapers or have custom configs

---

## 🔗 Useful Shopify Endpoints

- Products: `/products.json?limit=250&page=1`
- Single Product: `/products/{handle}.json`
- Collections: `/collections/{handle}/products.json`
- Product by ID: `/products/{id}.json`

---

**Phase:** 3 of 8  
**Dependencies:** Phase 1 (data models)  
**Can Run in Parallel With:** Phases 2, 4  
**Estimated Time:** 4-6 hours

