# n8n Data Tables Schema Documentation

This document defines the schema for all n8n Data Tables used in the Vendor Product Data Collection System.

---

## Table 1: Vendor Config

**Purpose:** Store configuration and metadata for all vendor data sources.

**Table Name:** `vendor_config`

### Schema

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `vendor_id` | String | PRIMARY KEY, NOT NULL | Unique identifier for vendor (e.g., "vendor_001") |
| `vendor_name` | String | NOT NULL | Human-readable vendor name |
| `data_source_type` | String | NOT NULL | Type of data source: 'drive', 'dropbox', 'sheets', 'shopify', 'website' |
| `source_url` | String | NULLABLE | URL or path to data source |
| `credentials` | JSON | NULLABLE, ENCRYPTED | Encrypted credentials for accessing data source |
| `last_sync_date` | Timestamp | NULLABLE | Last successful sync timestamp |
| `sync_status` | String | DEFAULT 'pending' | Current sync status: 'success', 'failed', 'pending' |
| `tier_used` | Integer | NOT NULL | Tier classification: 1 (Direct), 2 (Shopify), 3 (Zyte+AI) |
| `notes` | Text | NULLABLE | Additional notes or special instructions |

### Example Records

```json
{
  "vendor_id": "vendor_001",
  "vendor_name": "Acme Products",
  "data_source_type": "drive",
  "source_url": "https://drive.google.com/drive/folders/abc123",
  "credentials": {"type": "oauth", "token": "encrypted_token_here"},
  "last_sync_date": "2026-03-12T02:00:00Z",
  "sync_status": "success",
  "tier_used": 1,
  "notes": "Products organized by SKU in subfolders"
}
```

```json
{
  "vendor_id": "vendor_002",
  "vendor_name": "Beta Supplies",
  "data_source_type": "shopify",
  "source_url": "https://beta-supplies.myshopify.com",
  "credentials": null,
  "last_sync_date": "2026-03-12T02:15:00Z",
  "sync_status": "success",
  "tier_used": 2,
  "notes": "Uses public /products.json endpoint"
}
```

---

## Table 2: Product Data

**Purpose:** Store product information collected from all vendors.

**Table Name:** `product_data`

### Schema

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `sku` | String | PRIMARY KEY, NOT NULL | Product SKU (Stock Keeping Unit) |
| `vendor_id` | String | FOREIGN KEY, NOT NULL | References vendor_config.vendor_id |
| `product_name` | String | NOT NULL | Product name/title |
| `image_urls` | JSON | NULLABLE | Array of image URLs |
| `video_urls` | JSON | NULLABLE | Array of video URLs |
| `marketing_copy` | Text | NULLABLE | Product description/marketing text |
| `metadata` | JSON | NULLABLE | Additional product metadata (price, variants, etc.) |
| `last_updated` | Timestamp | NOT NULL | Last update timestamp |
| `source_tier` | Integer | NOT NULL | Tier used to collect data: 1, 2, or 3 |

### Example Records

```json
{
  "sku": "ACME-12345",
  "vendor_id": "vendor_001",
  "product_name": "Premium Widget Pro",
  "image_urls": [
    "https://drive.google.com/file/d/xyz789/view",
    "https://drive.google.com/file/d/xyz790/view"
  ],
  "video_urls": [
    "https://drive.google.com/file/d/xyz791/view"
  ],
  "marketing_copy": "The Premium Widget Pro revolutionizes your workflow with cutting-edge technology...",
  "metadata": {
    "color": "blue",
    "size": "large",
    "weight": "2.5 lbs"
  },
  "last_updated": "2026-03-12T02:05:00Z",
  "source_tier": 1
}
```

```json
{
  "sku": "BETA-SKU-789",
  "vendor_id": "vendor_002",
  "product_name": "Essential Tool Kit",
  "image_urls": [
    "https://cdn.shopify.com/s/files/1/0123/4567/products/tool-kit.jpg"
  ],
  "video_urls": [],
  "marketing_copy": "Complete tool kit for professionals and hobbyists alike.",
  "metadata": {
    "price": "$49.99",
    "variants": ["Standard", "Deluxe"],
    "shopify_handle": "essential-tool-kit"
  },
  "last_updated": "2026-03-12T02:16:00Z",
  "source_tier": 2
}
```

---

## Table 3: Sync Log (Optional but Recommended)

**Purpose:** Track sync operations for monitoring and debugging.

**Table Name:** `sync_log`

### Schema

| Column Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `log_id` | Integer | PRIMARY KEY, AUTO_INCREMENT | Unique log entry ID |
| `sync_date` | Timestamp | NOT NULL | Sync operation timestamp |
| `vendor_id` | String | NOT NULL | Vendor being synced |
| `status` | String | NOT NULL | Sync result: 'success', 'partial', 'failed' |
| `products_updated` | Integer | DEFAULT 0 | Number of products updated |
| `products_added` | Integer | DEFAULT 0 | Number of new products added |
| `errors` | JSON | NULLABLE | Array of error messages if any |
| `duration_seconds` | Integer | NULLABLE | Sync duration in seconds |

### Example Records

```json
{
  "log_id": 1,
  "sync_date": "2026-03-12T02:00:00Z",
  "vendor_id": "vendor_001",
  "status": "success",
  "products_updated": 45,
  "products_added": 5,
  "errors": [],
  "duration_seconds": 120
}
```

```json
{
  "log_id": 2,
  "sync_date": "2026-03-12T02:15:00Z",
  "vendor_id": "vendor_003",
  "status": "failed",
  "products_updated": 0,
  "products_added": 0,
  "errors": [
    {
      "error": "Authentication failed",
      "details": "Invalid API credentials"
    }
  ],
  "duration_seconds": 15
}
```

---

## n8n Implementation Notes

### Creating Tables in n8n

1. Navigate to **Data** → **Tables** in n8n Cloud
2. Click **Create Table**
3. Define columns according to schemas above
4. Set primary keys and constraints
5. Enable encryption for sensitive fields (credentials)

### Data Types Mapping

| Schema Type | n8n Data Type |
|------------|---------------|
| String | Text |
| Integer | Number |
| Timestamp | DateTime |
| JSON | JSON |
| Text | Long Text |

### Indexing Recommendations

For optimal query performance, create indexes on:
- `vendor_config.vendor_id` (primary key, auto-indexed)
- `product_data.sku` (primary key, auto-indexed)
- `product_data.vendor_id` (foreign key, for joins)
- `sync_log.vendor_id` (for filtering logs by vendor)
- `sync_log.sync_date` (for time-based queries)

### Security Considerations

- **Credentials Field:** Use n8n's built-in encryption for the `credentials` JSON field
- **Access Control:** Limit table access to authorized n8n workflows only
- **Backup:** Enable automatic backups for all tables
- **Audit Logging:** Enable n8n's audit logging for table modifications

---

## Maintenance

### Regular Tasks

- **Weekly:** Review sync_log for failures
- **Monthly:** Archive old sync_log entries (>90 days)
- **Quarterly:** Audit vendor_config for outdated credentials

### Schema Updates

When updating schemas:
1. Document changes in this file
2. Update version number below
3. Test with sample data before production deployment
4. Backup existing data before migration

---

**Schema Version:** 1.0  
**Last Updated:** 2026-03-19  
**Author:** Codegen AI

