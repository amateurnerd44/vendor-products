# Product Data Table Schema

## Overview
The `product_data` table stores all collected product information from vendors. This is the primary data store for product images, videos, marketing copy, and metadata that will be delivered to customers during order fulfillment.

## Table Name
`product_data`

## Schema Definition

| Field Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `sku` | Text | PRIMARY KEY | Unique product SKU (Stock Keeping Unit) identifier |
| `vendor_id` | Text | NOT NULL, FOREIGN KEY | Reference to vendor_config.vendor_id |
| `product_name` | Text | NOT NULL | Product name/title |
| `image_urls` | JSON | NULLABLE | Array of image URLs for the product |
| `video_urls` | JSON | NULLABLE | Array of video URLs for the product |
| `marketing_copy` | Long Text | NULLABLE | Product description, marketing text, specifications |
| `metadata` | JSON | NULLABLE | Additional product metadata (price, dimensions, tags, etc.) |
| `last_updated` | DateTime | NOT NULL | Timestamp of when this product data was last updated |
| `source_tier` | Number | NOT NULL | Tier used to collect this data: 1, 2, or 3 |

## Field Details

### sku
- **Format**: Alphanumeric string, vendor-specific format
- **Purpose**: Primary key for product lookup during order fulfillment
- **Examples**: "ACME-12345", "PROD-ABC-001", "SKU123456"
- **Uniqueness**: Must be globally unique across all vendors
- **Case Sensitivity**: Recommend storing in uppercase for consistency
- **Validation**: Should not contain special characters that break URLs

### vendor_id
- **Format**: `vendor_XXX` matching vendor_config.vendor_id
- **Purpose**: Links product to its vendor for relationship tracking
- **Foreign Key**: References `vendor_config.vendor_id`
- **Required**: Cannot be null
- **Usage**: Used to filter products by vendor, track vendor catalog size

### product_name
- **Format**: Free text, typically 50-200 characters
- **Purpose**: Human-readable product identifier
- **Examples**: 
  - "Acme Widget Pro 3000"
  - "Blue Cotton T-Shirt - Size L"
  - "Industrial Bearing Set (10-pack)"
- **Required**: Cannot be null
- **Searchability**: Should be indexed for product search functionality

### image_urls
- **Format**: JSON array of URL strings
- **Structure**:
  ```json
  [
    "https://cdn.vendor.com/images/product1-main.jpg",
    "https://cdn.vendor.com/images/product1-alt1.jpg",
    "https://cdn.vendor.com/images/product1-alt2.jpg"
  ]
  ```
- **Purpose**: Store all product images for customer delivery
- **Nullable**: Can be empty array `[]` or null if no images available
- **Order**: First image is typically the primary/hero image
- **Validation**: URLs should be accessible and return valid image content
- **Supported Formats**: JPG, PNG, WebP, GIF

### video_urls
- **Format**: JSON array of URL strings
- **Structure**:
  ```json
  [
    "https://youtube.com/watch?v=ABC123",
    "https://vimeo.com/123456789",
    "https://cdn.vendor.com/videos/product-demo.mp4"
  ]
  ```
- **Purpose**: Store product videos for customer delivery
- **Nullable**: Can be empty array `[]` or null if no videos available
- **Supported Platforms**: YouTube, Vimeo, direct MP4/MOV links
- **Validation**: URLs should be accessible

### marketing_copy
- **Format**: Long text, supports markdown
- **Purpose**: Store product descriptions, specifications, marketing text
- **Content Types**:
  - Product descriptions
  - Technical specifications
  - Features and benefits
  - Usage instructions
  - Warranty information
- **Nullable**: Can be null if no copy available
- **Length**: No strict limit, but typically 500-5000 characters
- **Formatting**: Preserve line breaks and basic formatting

### metadata
- **Format**: JSON object with flexible schema
- **Purpose**: Store additional product information not covered by other fields
- **Common Fields**:
  ```json
  {
    "price": 29.99,
    "currency": "USD",
    "dimensions": {
      "length": 10,
      "width": 5,
      "height": 3,
      "unit": "inches"
    },
    "weight": {
      "value": 2.5,
      "unit": "lbs"
    },
    "tags": ["electronics", "gadgets", "bestseller"],
    "category": "Electronics > Gadgets",
    "brand": "Acme",
    "color": "Blue",
    "size": "Large",
    "material": "Plastic",
    "in_stock": true,
    "stock_quantity": 150,
    "upc": "012345678901",
    "manufacturer_sku": "MFG-12345"
  }
  ```
- **Nullable**: Can be null or empty object `{}`
- **Flexibility**: Schema can vary by vendor and product type
- **Extensibility**: Easy to add new fields without schema changes

### last_updated
- **Format**: ISO 8601 DateTime (e.g., "2026-03-19T14:30:00Z")
- **Purpose**: Track data freshness for cache invalidation
- **Required**: Cannot be null
- **Updated**: Set to current timestamp whenever product data is updated
- **Usage**: 
  - Determine if product needs re-sync
  - Show data age in reports
  - Trigger alerts for stale data (>30 days)

### source_tier
- **Valid Values**: 1, 2, or 3
- **Purpose**: Track which data collection method was used
- **Tier Definitions**:
  - **Tier 1**: Direct integration (Drive/Dropbox/Sheets) - FREE
  - **Tier 2**: Shopify JSON endpoint - FREE
  - **Tier 3**: Zyte API + Gemini AI - USAGE-BASED COST
- **Usage**: Cost analysis, quality tracking, optimization decisions
- **Required**: Cannot be null

## Relationships

### Foreign Key References
- `vendor_id` references `vendor_config.vendor_id` (many-to-one)

### Referenced By
- Order fulfillment workflows query by `sku`
- Sync workflows update by `sku` and `vendor_id`

## Indexes

### Recommended Indexes
1. **Primary Key**: `sku` (automatic)
2. **Vendor Lookup**: `vendor_id` (for vendor-specific queries)
3. **Freshness**: `last_updated` (for finding stale products)
4. **Tier Analysis**: `source_tier` (for cost tracking)
5. **Composite**: `(vendor_id, last_updated)` (for vendor sync optimization)

## Usage Examples

### Adding a Product (Tier 1 - Google Drive)
```json
{
  "sku": "ACME-WIDGET-001",
  "vendor_id": "vendor_001",
  "product_name": "Acme Widget Pro 3000",
  "image_urls": [
    "https://drive.google.com/uc?id=ABC123",
    "https://drive.google.com/uc?id=DEF456"
  ],
  "video_urls": [
    "https://youtube.com/watch?v=XYZ789"
  ],
  "marketing_copy": "The Acme Widget Pro 3000 is the ultimate solution for all your widget needs. Features include:\n- Durable construction\n- Easy to use\n- 5-year warranty",
  "metadata": {
    "price": 149.99,
    "currency": "USD",
    "category": "Widgets > Professional",
    "in_stock": true
  },
  "last_updated": "2026-03-19T14:30:00Z",
  "source_tier": 1
}
```

### Adding a Product (Tier 2 - Shopify)
```json
{
  "sku": "BETA-SUPPLY-42",
  "vendor_id": "vendor_002",
  "product_name": "Industrial Bearing Set",
  "image_urls": [
    "https://cdn.shopify.com/s/files/1/0123/4567/products/bearing-main.jpg",
    "https://cdn.shopify.com/s/files/1/0123/4567/products/bearing-detail.jpg"
  ],
  "video_urls": [],
  "marketing_copy": "High-quality industrial bearings suitable for heavy machinery. Set includes 10 bearings of various sizes.",
  "metadata": {
    "price": 89.99,
    "currency": "USD",
    "shopify_product_id": 1234567890,
    "shopify_handle": "industrial-bearing-set",
    "tags": ["industrial", "bearings", "machinery"],
    "variants": [
      {"size": "small", "sku": "BETA-SUPPLY-42-S"},
      {"size": "large", "sku": "BETA-SUPPLY-42-L"}
    ]
  },
  "last_updated": "2026-03-19T14:35:00Z",
  "source_tier": 2
}
```

### Adding a Product (Tier 3 - Website Scraping)
```json
{
  "sku": "EPSILON-TOOL-999",
  "vendor_id": "vendor_005",
  "product_name": "Professional Power Drill",
  "image_urls": [
    "https://epsilon-wholesale.com/images/drill-main.jpg",
    "https://epsilon-wholesale.com/images/drill-side.jpg",
    "https://epsilon-wholesale.com/images/drill-accessories.jpg"
  ],
  "video_urls": [
    "https://epsilon-wholesale.com/videos/drill-demo.mp4"
  ],
  "marketing_copy": "Professional-grade power drill with variable speed control and ergonomic design. Perfect for contractors and DIY enthusiasts.",
  "metadata": {
    "price": 199.99,
    "currency": "USD",
    "extracted_by": "gemini-1.5-pro",
    "extraction_confidence": 0.95,
    "source_url": "https://epsilon-wholesale.com/products/power-drill-999",
    "scraped_at": "2026-03-19T14:40:00Z"
  },
  "last_updated": "2026-03-19T14:40:00Z",
  "source_tier": 3
}
```

## n8n Implementation Notes

### Creating the Table in n8n
1. Navigate to **Data** → **Data Tables** in n8n Cloud
2. Click **Create Table**
3. Name: `product_data`
4. Add fields according to schema above
5. Set `sku` as Primary Key
6. Create index on `vendor_id` for performance

### Querying Products by SKU
```javascript
// Single SKU lookup (order fulfillment)
const product = $('Data Table').get({ sku: 'ACME-WIDGET-001' });

// Multiple SKUs lookup
const skus = ['SKU1', 'SKU2', 'SKU3'];
const products = $('Data Table').all().filter(p => skus.includes(p.sku));
```

### Updating Product Data
```javascript
// Upsert pattern (update if exists, insert if not)
const productData = {
  sku: 'ACME-WIDGET-001',
  vendor_id: 'vendor_001',
  product_name: 'Updated Product Name',
  image_urls: ['https://new-image.jpg'],
  video_urls: [],
  marketing_copy: 'Updated description',
  metadata: { price: 159.99 },
  last_updated: new Date().toISOString(),
  source_tier: 1
};

// Check if exists
const existing = $('Data Table').get({ sku: productData.sku });
if (existing) {
  $('Data Table').update(productData);
} else {
  $('Data Table').insert(productData);
}
```

### Finding Stale Products
```javascript
// Find products not updated in 30 days
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
const staleProducts = $('Data Table').all().filter(p => 
  new Date(p.last_updated) < thirtyDaysAgo
);
```

## Validation Rules

### Required Field Validation
- `sku`: Must not be empty, should be unique
- `vendor_id`: Must exist in vendor_config table
- `product_name`: Must not be empty
- `last_updated`: Must be valid ISO 8601 datetime
- `source_tier`: Must be 1, 2, or 3

### Data Quality Validation
- `image_urls`: If provided, should be valid URLs
- `video_urls`: If provided, should be valid URLs
- `metadata`: If provided, must be valid JSON
- `sku`: Should match vendor's SKU format conventions

### Business Logic Validation
- `source_tier` should match the vendor's `tier_used` in vendor_config
- `last_updated` should not be in the future
- Products should be re-synced if `last_updated` > 7 days old

## Data Quality Considerations

### Image URL Validation
- Verify URLs are accessible (HTTP 200 response)
- Check image file size (warn if >5MB)
- Validate image format (JPG, PNG, WebP, GIF)
- Consider creating thumbnails for large images

### Video URL Validation
- Verify URLs are accessible
- Check if video platform is supported (YouTube, Vimeo, direct links)
- Consider extracting video metadata (duration, resolution)

### Marketing Copy Quality
- Check for minimum length (e.g., >50 characters)
- Detect placeholder text ("Lorem ipsum", "TBD", etc.)
- Flag products with missing copy for review

### Metadata Completeness
- Track which metadata fields are populated
- Flag products missing critical fields (price, category)
- Generate data quality reports by vendor

## Order Fulfillment Integration

### Fast Lookup Pattern
```javascript
// Order comes in with SKUs
const orderSkus = ['SKU1', 'SKU2', 'SKU3'];

// Lookup all products
const products = [];
const missingSkus = [];

for (const sku of orderSkus) {
  const product = $('Data Table').get({ sku: sku });
  if (product) {
    products.push(product);
  } else {
    missingSkus.push(sku);
  }
}

// If missing SKUs, trigger on-demand scraping
if (missingSkus.length > 0) {
  // Trigger fallback scraping workflow
  $('Webhook').call('on-demand-scrape', { skus: missingSkus });
}

// Compile digital asset package
const assetPackage = products.map(p => ({
  sku: p.sku,
  name: p.product_name,
  images: p.image_urls,
  videos: p.video_urls,
  description: p.marketing_copy
}));
```

## Performance Optimization

### Caching Strategy
- Cache frequently accessed products in memory
- Implement TTL based on `last_updated` timestamp
- Pre-load popular products during off-peak hours

### Batch Operations
- Use batch inserts for weekly sync (faster than individual inserts)
- Batch update operations for multiple products
- Use transactions for data consistency

### Query Optimization
- Use indexes for common query patterns
- Avoid full table scans
- Paginate large result sets

## Troubleshooting

### Common Issues

**Issue**: SKU not found during order fulfillment
- Check if SKU exists in table
- Verify SKU format matches vendor's format
- Check if product was synced recently
- Trigger on-demand scraping as fallback

**Issue**: Image URLs not accessible
- Verify URLs are still valid
- Check if vendor changed CDN or hosting
- Re-sync vendor to get updated URLs
- Consider downloading and hosting images locally

**Issue**: Duplicate SKUs
- Investigate if multiple vendors use same SKU
- Implement vendor-specific SKU prefixes
- Update SKU format to ensure uniqueness

**Issue**: Stale product data
- Check `last_updated` timestamp
- Verify weekly sync is running
- Check sync_log for vendor sync failures
- Manually trigger sync for specific vendor

## Migration Notes

### Initial Population
1. Run initial sync for all vendors
2. Validate data quality for sample products
3. Check for duplicate SKUs
4. Verify image/video URLs are accessible
5. Generate data quality report

### Data Updates
- Weekly sync updates existing products and adds new ones
- Use upsert pattern (update if exists, insert if new)
- Track changes in sync_log table
- Archive old product data if needed

## Security Considerations

### Data Privacy
- Product data may contain proprietary vendor information
- Limit access to authorized workflows only
- Consider data retention policies
- Comply with vendor data usage agreements

### URL Validation
- Validate all URLs before storing
- Sanitize URLs to prevent injection attacks
- Check for malicious content in images/videos
- Use HTTPS URLs when possible

## Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-19 | Initial schema definition |

---

**Related Documentation:**
- [Vendor Config Schema](./vendor_config.md)
- [Sync Log Schema](./sync_log.md)
- [Order Fulfillment Workflow](../guides/order_fulfillment.md)
- [Setup Guide](../guides/SETUP_GUIDE.md)

