# Tier 2: Shopify Workflows

This directory contains n8n workflows for scraping Shopify stores via public JSON endpoints.

## Workflows

### 1. shopify_detector.json
**Purpose:** Detect if a URL is a Shopify store  
**Input:** `{ store_url: string }`  
**Output:** `{ is_shopify: boolean, ... }`

### 2. shopify_scraper.json
**Purpose:** Scrape all products from a Shopify store  
**Input:** `{ store_url: string, vendor_id: string }`  
**Output:** Array of products with variants, images, and metadata

### 3. sku_mapper.json
**Purpose:** Build SKU → Shopify handle mappings  
**Input:** Array of products from scraper  
**Output:** Array of SKU mappings for fast lookups

### 4. html_cleaner.json
**Purpose:** Extract clean text and videos from HTML  
**Input:** `{ body_html: string }`  
**Output:** `{ marketing_copy: string, video_urls: array }`

### 5. test_shopify_workflows.json
**Purpose:** Automated testing suite for all workflows  
**Input:** None (uses predefined test cases)  
**Output:** Test results summary

## Quick Start

### Import to n8n

1. Open n8n Cloud
2. Go to **Workflows** → **Import from File**
3. Select a workflow JSON file
4. Click **Import**
5. Save the workflow

### Test Individual Workflows

**Shopify Detector:**
```json
{
  "store_url": "https://shop.polymer80.com"
}
```

**Shopify Scraper:**
```json
{
  "store_url": "https://shop.polymer80.com",
  "vendor_id": "vendor_123"
}
```

**HTML Cleaner:**
```json
{
  "body_html": "<p>Product description</p><iframe src='https://youtube.com/embed/abc'></iframe>"
}
```

## Rate Limits

- **Shopify:** 2 requests/second per store
- **Implementation:** 500ms delay between requests
- **Handled automatically** in the scraper workflow

## Documentation

See [TIER2_SHOPIFY.md](../../docs/workflows/TIER2_SHOPIFY.md) for complete documentation.

## Support

For issues or questions:
1. Check the main documentation
2. Review n8n execution logs
3. Test with known working Shopify stores
4. Consult IMPLEMENTATION_PLAN.md

---

**Phase:** 3  
**Tier:** 2  
**Status:** Complete  
**Last Updated:** 2026-03-19

