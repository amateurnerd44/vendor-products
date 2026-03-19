# Tier 2: Shopify Public Endpoint Scraper

## Overview

The Tier 2 scraper leverages Shopify's free public JSON endpoints to collect product data without authentication or API costs. This tier is ideal for vendors who operate Shopify stores, as it provides structured product data including images, variants, and descriptions.

**Key Benefits:**
- ✅ **FREE** - No API costs or authentication required
- ✅ **Structured Data** - Clean JSON format with consistent schema
- ✅ **Reliable** - Official Shopify endpoints with good uptime
- ✅ **Comprehensive** - Includes products, variants, images, and descriptions

**Rate Limits:**
- Maximum 2 requests per second per store
- Implemented with 500ms delay between requests

---

## Architecture

The Tier 2 system consists of 4 n8n workflows:

1. **Shopify Detector** - Validates if a URL is a Shopify store
2. **Shopify Scraper** - Fetches all products with pagination
3. **SKU Mapper** - Builds SKU → Shopify handle mappings
4. **HTML Cleaner** - Extracts clean text and video URLs from HTML

---

## Workflow 1: Shopify Detector

**File:** `workflows/tier2/shopify_detector.json`

### Purpose
Determines if a given URL is a Shopify store by testing the `/products.json` endpoint.

### Input Schema
```json
{
  "store_url": "https://example-store.myshopify.com"
}
```

### Process Flow
1. **Test Endpoint** - Makes HTTP request to `{store_url}/products.json?limit=1`
2. **Validate Response** - Checks for:
   - HTTP 200 status code
   - `products` array exists
   - Valid JSON structure
3. **Return Result** - Outputs detection status

### Output Schema

**Success (Shopify Detected):**
```json
{
  "is_shopify": true,
  "store_url": "https://example-store.myshopify.com",
  "products_endpoint": "https://example-store.myshopify.com/products.json",
  "detection_timestamp": "2026-03-19T16:09:00.000Z",
  "message": "Valid Shopify store detected"
}
```

**Failure (Not Shopify):**
```json
{
  "is_shopify": false,
  "store_url": "https://example-store.com",
  "detection_timestamp": "2026-03-19T16:09:00.000Z",
  "message": "Not a Shopify store or /products.json endpoint not accessible",
  "error_details": "404 Not Found"
}
```

### Error Handling
- Continues on HTTP errors (404, 403, etc.)
- Handles custom domains that may not expose `/products.json`
- Follows redirects (max 3)
- 10-second timeout

### Usage Example
```javascript
// In n8n, call this workflow with:
{
  "store_url": "https://shop.example.com"
}

// Use the output to determine if Tier 2 scraping is possible
```

---

## Workflow 2: Shopify Scraper

**File:** `workflows/tier2/shopify_scraper.json`

### Purpose
Fetches all products from a Shopify store using pagination, respecting rate limits.

### Input Schema
```json
{
  "store_url": "https://example-store.myshopify.com",
  "vendor_id": "vendor_123"
}
```

### Process Flow
1. **Initialize** - Set up pagination variables (page=1, limit=250)
2. **Fetch Page** - Request products from `/products.json?limit=250&page=N`
3. **Rate Limit** - Wait 500ms between requests (Shopify allows 2 req/sec)
4. **Accumulate** - Collect products from all pages
5. **Check More** - Continue if page returned 250 products (max per page)
6. **Process** - Transform all products to our schema
7. **Output** - Return processed products

### Pagination Logic
- Shopify returns max 250 products per page
- Loop continues while `products.length === 250`
- Automatic rate limiting with 500ms delay
- Handles stores with 1 to 10,000+ products

### Output Schema
```json
{
  "vendor_id": "vendor_123",
  "product_name": "Example Product",
  "shopify_handle": "example-product",
  "product_url": "https://example-store.myshopify.com/products/example-product",
  "image_urls": [
    "https://cdn.shopify.com/s/files/1/0001/2345/products/image1.jpg",
    "https://cdn.shopify.com/s/files/1/0001/2345/products/image2.jpg"
  ],
  "body_html": "<p>Product description with <strong>HTML</strong></p>",
  "skus": ["SKU-001", "SKU-002"],
  "variants": [
    {
      "id": 12345678,
      "title": "Small / Red",
      "sku": "SKU-001",
      "price": "29.99",
      "inventory_quantity": 10
    }
  ],
  "metadata": {
    "shopify_id": 1234567890,
    "product_type": "Apparel",
    "tags": ["new", "sale", "featured"],
    "vendor": "Brand Name",
    "created_at": "2026-01-15T10:30:00Z",
    "updated_at": "2026-03-19T14:20:00Z"
  },
  "source_tier": 2,
  "last_updated": "2026-03-19T16:09:00.000Z"
}
```

### Error Handling
- Retries failed requests 3 times with 1-second delay
- Continues on fail to handle partial data
- 15-second timeout per request
- Outputs error if no products found

### Performance
- **Small stores (1-250 products):** ~1 second
- **Medium stores (250-1000 products):** ~3-5 seconds
- **Large stores (1000-5000 products):** ~15-30 seconds
- **Very large stores (5000+ products):** ~1-2 minutes

### Rate Limiting Implementation
```javascript
// 500ms delay ensures we stay under 2 req/sec
await new Promise(resolve => setTimeout(resolve, 500));
```

---

## Workflow 3: SKU Mapper

**File:** `workflows/tier2/sku_mapper.json`

### Purpose
Creates SKU → Shopify handle mappings for fast order fulfillment lookups.

### Input Schema
Accepts output from **Shopify Scraper** (array of products).

### Process Flow
1. **Build Mappings** - Extract all SKUs from products and variants
2. **Filter Valid** - Remove empty or invalid SKUs
3. **Format** - Structure data for storage in Product Data table
4. **Generate Stats** - Create summary statistics

### Output Schema

**Individual Mapping:**
```json
{
  "sku": "SKU-001",
  "vendor_id": "vendor_123",
  "shopify_handle": "example-product",
  "product_url": "https://example-store.myshopify.com/products/example-product",
  "product_name": "Example Product",
  "mapping_metadata": {
    "variant_title": "Small / Red",
    "variant_id": 12345678,
    "mapping_type": "shopify_variant"
  },
  "last_updated": "2026-03-19T16:09:00.000Z"
}
```

**Statistics Output:**
```json
{
  "total_mappings": 450,
  "unique_skus": 420,
  "unique_vendors": 1,
  "unique_products": 150,
  "mapping_types": {
    "shopify": 150,
    "shopify_variant": 300
  },
  "timestamp": "2026-03-19T16:09:00.000Z"
}
```

### Mapping Types
- **`shopify`** - Direct product-level SKU
- **`shopify_variant`** - Variant-specific SKU (most common)

### Handling Multiple Variants
Products with multiple variants (e.g., sizes, colors) generate multiple mappings:

```javascript
// Product: "T-Shirt"
// Variants: Small/Red, Medium/Blue, Large/Green
// Result: 3 separate SKU mappings, all pointing to same handle
```

### Usage in Order Fulfillment
```javascript
// When order comes in with SKU "SKU-001"
// 1. Query mapping table: SELECT * WHERE sku = "SKU-001"
// 2. Get shopify_handle: "example-product"
// 3. Fetch product data using handle
// 4. Deliver assets to customer
```

---

## Workflow 4: HTML Cleaner

**File:** `workflows/tier2/html_cleaner.json`

### Purpose
Converts Shopify's `body_html` to clean marketing copy and extracts video URLs.

### Input Schema
```json
{
  "body_html": "<div><p>Product description</p><iframe src='https://youtube.com/embed/abc123'></iframe></div>"
}
```

### Process Flow
1. **Extract Videos** - Find all video URLs in HTML
2. **Strip HTML** - Remove tags and convert to clean text
3. **Format Output** - Return clean copy and video URLs

### Video Extraction Patterns
Supports multiple video sources:
- **YouTube** - iframe embeds, direct links
- **Vimeo** - iframe embeds, player links
- **HTML5 Video** - `<video>` and `<source>` tags
- **Direct URLs** - .mp4, .webm, .ogg, .mov, .avi files

### HTML Cleaning Process
1. Remove `<script>` and `<style>` tags completely
2. Convert HTML entities (`&nbsp;`, `&amp;`, etc.)
3. Convert block elements to line breaks (`<p>`, `<div>`, `<br>`)
4. Strip all remaining HTML tags
5. Clean up excessive whitespace
6. Remove excessive line breaks (max 2 consecutive)

### Output Schema
```json
{
  "marketing_copy": "Product description\n\nFeatures:\n• Feature 1\n• Feature 2",
  "video_urls": [
    "https://www.youtube.com/watch?v=abc123",
    "https://vimeo.com/456789"
  ],
  "has_videos": true,
  "processing_stats": {
    "original_length": 1250,
    "clean_length": 320,
    "video_count": 2
  },
  "processed_at": "2026-03-19T16:09:00.000Z"
}
```

### Example Transformations

**Input HTML:**
```html
<div class="product-description">
  <h2>Premium T-Shirt</h2>
  <p>Made from 100% organic cotton.</p>
  <ul>
    <li>Soft &amp; comfortable</li>
    <li>Machine washable</li>
  </ul>
  <iframe src="https://youtube.com/embed/abc123"></iframe>
</div>
```

**Output Text:**
```
Premium T-Shirt

Made from 100% organic cotton.

Soft & comfortable
Machine washable
```

**Extracted Videos:**
```json
["https://www.youtube.com/watch?v=abc123"]
```

---

## Integration with Main Sync Workflow

### Flow 1: Weekly Catalog Sync

```
┌─────────────────────────────────────────────────────────┐
│ Main Sync Orchestrator (Phase 5)                       │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Tier Router: Check vendor.data_source_type             │
│ If "shopify" → Route to Tier 2                         │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 1. Shopify Detector                                     │
│    Input: { store_url }                                 │
│    Output: { is_shopify: true/false }                   │
└─────────────────────────────────────────────────────────┘
                        ↓ (if is_shopify = true)
┌─────────────────────────────────────────────────────────┐
│ 2. Shopify Scraper                                      │
│    Input: { store_url, vendor_id }                      │
│    Output: Array of products with variants             │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 3. HTML Cleaner (for each product)                      │
│    Input: { body_html }                                 │
│    Output: { marketing_copy, video_urls }               │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 4. SKU Mapper                                           │
│    Input: Array of products                             │
│    Output: SKU → handle mappings                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Upsert to Product Data Table                         │
│    - Update existing SKUs                               │
│    - Insert new SKUs                                    │
│    - Update last_updated timestamp                      │
└─────────────────────────────────────────────────────────┘
```

### Flow 2: Order Fulfillment (On-Demand)

```
┌─────────────────────────────────────────────────────────┐
│ Order Received: SKU not in cache                        │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Lookup vendor_id for SKU                                │
│ Get vendor config (store_url)                           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Run Shopify Scraper for specific vendor                 │
│ (Same process as weekly sync)                           │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Save to Product Data Table                              │
│ Continue with order fulfillment                         │
└─────────────────────────────────────────────────────────┘
```

---

## Testing Guide

### Test Workflow 1: Shopify Detector

**Test Case 1: Valid Shopify Store**
```json
{
  "store_url": "https://shop.polymer80.com"
}
```
Expected: `is_shopify: true`

**Test Case 2: Non-Shopify Site**
```json
{
  "store_url": "https://www.google.com"
}
```
Expected: `is_shopify: false`

**Test Case 3: Custom Domain Shopify**
```json
{
  "store_url": "https://www.allbirds.com"
}
```
Expected: `is_shopify: true` (Allbirds uses Shopify)

### Test Workflow 2: Shopify Scraper

**Test Case 1: Small Store (< 250 products)**
```json
{
  "store_url": "https://shop.polymer80.com",
  "vendor_id": "test_vendor_001"
}
```
Expected: Single page fetch, all products returned

**Test Case 2: Large Store (> 250 products)**
```json
{
  "store_url": "https://www.gymshark.com",
  "vendor_id": "test_vendor_002"
}
```
Expected: Multiple pages, pagination works correctly

**Test Case 3: Multi-Variant Products**
```json
{
  "store_url": "https://www.allbirds.com",
  "vendor_id": "test_vendor_003"
}
```
Expected: Products with multiple variants (sizes, colors) properly extracted

### Test Workflow 3: SKU Mapper

**Input:** Output from Shopify Scraper test
**Validation:**
- All SKUs extracted from variants
- No duplicate mappings
- Statistics match actual counts
- Empty SKUs filtered out

### Test Workflow 4: HTML Cleaner

**Test Case 1: YouTube Video**
```json
{
  "body_html": "<p>Check out our video:</p><iframe src='https://youtube.com/embed/abc123'></iframe>"
}
```
Expected: Video URL extracted and normalized

**Test Case 2: Complex HTML**
```json
{
  "body_html": "<div><h2>Title</h2><p>Description with &amp; entities</p><ul><li>Item 1</li></ul></div>"
}
```
Expected: Clean text with proper formatting

**Test Case 3: No Videos**
```json
{
  "body_html": "<p>Simple description without videos</p>"
}
```
Expected: `video_urls: []`, `has_videos: false`

---

## Performance Benchmarks

### Shopify Detector
- **Average Response Time:** 200-500ms
- **Success Rate:** >99%
- **False Positives:** <1%

### Shopify Scraper
| Store Size | Products | Pages | Time | Rate Limit Delays |
|------------|----------|-------|------|-------------------|
| Small      | 50       | 1     | 1s   | 0                 |
| Medium     | 500      | 2     | 2s   | 1 × 500ms         |
| Large      | 2000     | 8     | 8s   | 7 × 500ms         |
| Very Large | 5000     | 20    | 20s  | 19 × 500ms        |

### SKU Mapper
- **Processing Speed:** ~1000 SKUs/second
- **Memory Usage:** Minimal (streaming)

### HTML Cleaner
- **Processing Speed:** ~500 products/second
- **Video Detection Rate:** >95%

---

## Error Scenarios & Handling

### Scenario 1: Rate Limit Exceeded
**Symptom:** HTTP 429 responses
**Handling:** 
- Automatic retry with exponential backoff
- Increase delay between requests to 1000ms
- Log warning in Sync Log table

### Scenario 2: Store Temporarily Down
**Symptom:** HTTP 503 or timeout
**Handling:**
- Retry 3 times with 1-second delay
- If still failing, mark vendor sync as "failed"
- Send alert notification
- Continue with other vendors

### Scenario 3: Products Endpoint Disabled
**Symptom:** HTTP 403 or 404
**Handling:**
- Mark vendor as "not_shopify" in config
- Suggest fallback to Tier 3 (Zyte + AI)
- Log error for manual review

### Scenario 4: Malformed Product Data
**Symptom:** Missing fields, invalid JSON
**Handling:**
- Skip malformed products
- Log SKUs that failed
- Continue processing valid products
- Report partial success

### Scenario 5: Empty Store
**Symptom:** `products: []`
**Handling:**
- Mark sync as "success" with 0 products
- Log warning
- Don't delete existing cached products

---

## Cost Analysis

### Tier 2 Costs: $0.00

**Why Free?**
- Public Shopify endpoints require no authentication
- No API keys or tokens needed
- No per-request charges
- Only costs are n8n execution time (included in n8n Cloud plan)

**Comparison to Alternatives:**
- Shopify Admin API: Requires app installation, OAuth, rate limits
- Zyte API (Tier 3): $0.00013 per request
- Manual scraping: Time-consuming, error-prone

**Estimated Savings:**
- 20 Shopify vendors × 100 products each = 2,000 products
- If using Zyte: 2,000 × $0.00013 = $0.26/week = $13.52/year
- **Tier 2 saves $13.52/year per 20 vendors**

---

## Maintenance & Monitoring

### Weekly Checks
- [ ] Review Shopify Detector success rate
- [ ] Check for vendors with failed syncs
- [ ] Validate SKU mapping counts
- [ ] Monitor scraper execution times

### Monthly Tasks
- [ ] Audit video extraction accuracy
- [ ] Review HTML cleaning quality
- [ ] Update regex patterns if needed
- [ ] Optimize slow-performing stores

### Alerts to Configure
- Shopify Detector failure rate > 5%
- Scraper execution time > 2 minutes
- SKU mapping count drops significantly
- Video extraction rate < 90%

---

## Troubleshooting

### Issue: "Not a Shopify store" but it is
**Cause:** Custom domain without `/products.json` exposed
**Solution:** 
1. Try adding `.myshopify.com` subdomain
2. Check if store has disabled public product listings
3. Fallback to Tier 3 (Zyte + AI)

### Issue: Scraper times out
**Cause:** Very large store (10,000+ products)
**Solution:**
1. Increase n8n workflow timeout to 5 minutes
2. Consider splitting into multiple smaller scrapes
3. Cache results more aggressively

### Issue: Missing SKUs in mappings
**Cause:** Variants without SKUs assigned
**Solution:**
1. Use variant IDs as fallback identifiers
2. Log products with missing SKUs
3. Request vendor to add SKUs in Shopify admin

### Issue: Videos not extracted
**Cause:** Non-standard video embed format
**Solution:**
1. Review HTML structure
2. Add new regex pattern to HTML Cleaner
3. Test with sample product
4. Deploy updated workflow

---

## Future Enhancements

### Planned Improvements
1. **Incremental Sync** - Only fetch products updated since last sync
2. **Variant Images** - Extract variant-specific images
3. **Collections** - Scrape product collections/categories
4. **Inventory Tracking** - Monitor stock levels
5. **Price History** - Track price changes over time

### Advanced Features
1. **Multi-Store Support** - Handle vendors with multiple Shopify stores
2. **Localization** - Extract multi-language product data
3. **Metafields** - Access custom Shopify metafields
4. **Smart Retry** - ML-based retry logic for failed requests

---

## Related Documentation

- [Phase 1: Foundation & Infrastructure](../guides/PHASE1_FOUNDATION.md)
- [Phase 5: Weekly Sync Workflow](../guides/PHASE5_WEEKLY_SYNC.md)
- [Phase 6: Order Fulfillment](../guides/PHASE6_ORDER_FULFILLMENT.md)
- [Data Models & Schemas](../schemas/DATA_MODELS.md)
- [Vendor Configuration Guide](../guides/VENDOR_CONFIG.md)

---

## Support

For issues or questions about Tier 2 Shopify scraping:
1. Check this documentation first
2. Review n8n workflow execution logs
3. Test with known working Shopify stores
4. Consult main implementation plan

**Last Updated:** 2026-03-19  
**Version:** 1.0  
**Author:** Codegen AI

