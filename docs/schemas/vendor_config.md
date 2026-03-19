# Vendor Config Table Schema

## Overview
The `vendor_config` table stores configuration information for all vendors in the system. This table is the central registry for vendor data sources and determines which tier of data collection will be used for each vendor.

## Table Name
`vendor_config`

## Schema Definition

| Field Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `vendor_id` | Text | PRIMARY KEY | Unique identifier for the vendor (e.g., "vendor_001") |
| `vendor_name` | Text | NOT NULL | Human-readable vendor name (e.g., "Acme Products") |
| `data_source_type` | Text | NOT NULL | Type of data source: 'drive', 'dropbox', 'sheets', 'shopify', 'website' |
| `source_url` | Text | NULLABLE | URL or path to the vendor's data source (e.g., folder URL, Shopify store URL, website URL) |
| `credentials` | JSON | NULLABLE, ENCRYPTED | Encrypted credentials for accessing the data source (API keys, OAuth tokens, etc.) |
| `last_sync_date` | DateTime | NULLABLE | Timestamp of the last successful sync |
| `sync_status` | Text | DEFAULT 'pending' | Current sync status: 'pending', 'success', 'failed', 'in_progress' |
| `tier_used` | Number | NOT NULL | Data collection tier: 1 (Direct Integration), 2 (Shopify JSON), 3 (Zyte + AI) |
| `notes` | Long Text | NULLABLE | Additional notes about the vendor, special handling requirements, or configuration details |

## Field Details

### vendor_id
- **Format**: `vendor_XXX` where XXX is a zero-padded number (e.g., vendor_001, vendor_042)
- **Purpose**: Serves as the primary key and foreign key reference in other tables
- **Uniqueness**: Must be unique across all vendors

### vendor_name
- **Format**: Free text, typically company name
- **Purpose**: Human-readable identifier for reports and UI
- **Example**: "Acme Products", "Beta Supplies Co."

### data_source_type
- **Valid Values**:
  - `drive` - Google Drive folder
  - `dropbox` - Dropbox folder
  - `sheets` - Google Sheets spreadsheet
  - `shopify` - Shopify store
  - `website` - Custom website requiring scraping
- **Purpose**: Determines which scraper/integration to use
- **Tier Mapping**:
  - `drive`, `dropbox`, `sheets` → Tier 1 (Free)
  - `shopify` → Tier 2 (Free)
  - `website` → Tier 3 (Usage-based cost)

### source_url
- **Format**: Full URL or path
- **Examples**:
  - Google Drive: `https://drive.google.com/drive/folders/ABC123`
  - Dropbox: `https://www.dropbox.com/sh/xyz789`
  - Google Sheets: `https://docs.google.com/spreadsheets/d/SHEET_ID`
  - Shopify: `https://store-name.myshopify.com`
  - Website: `https://vendor-website.com/products`
- **Nullable**: Can be null if credentials contain the path or for manual data entry

### credentials
- **Format**: JSON object with service-specific fields
- **Security**: MUST be encrypted at rest in n8n
- **Examples**:
  ```json
  // Google Drive/Sheets
  {
    "type": "oauth2",
    "refresh_token": "ENCRYPTED_TOKEN",
    "client_id": "CLIENT_ID",
    "client_secret": "ENCRYPTED_SECRET"
  }
  
  // Dropbox
  {
    "type": "access_token",
    "token": "ENCRYPTED_TOKEN"
  }
  
  // Shopify
  {
    "type": "api_key",
    "api_key": "ENCRYPTED_KEY",
    "password": "ENCRYPTED_PASSWORD"
  }
  
  // Website (if needed)
  {
    "type": "basic_auth",
    "username": "USERNAME",
    "password": "ENCRYPTED_PASSWORD"
  }
  ```
- **Nullable**: Can be null if using shared credentials or public access

### last_sync_date
- **Format**: ISO 8601 DateTime (e.g., "2026-03-19T02:00:00Z")
- **Purpose**: Track data freshness and sync schedule
- **Updated**: Set to current timestamp after each successful sync
- **Nullable**: Will be null for vendors that have never been synced

### sync_status
- **Valid Values**:
  - `pending` - Never synced or waiting for next sync
  - `in_progress` - Currently syncing
  - `success` - Last sync completed successfully
  - `failed` - Last sync failed (check sync_log for details)
- **Default**: `pending`
- **Purpose**: Quick status check without querying sync_log table

### tier_used
- **Valid Values**: 1, 2, or 3
- **Tier Definitions**:
  - **Tier 1**: Direct integrations (Google Drive, Dropbox, Google Sheets) - FREE
  - **Tier 2**: Shopify public JSON endpoints - FREE
  - **Tier 3**: Zyte API + Gemini AI scraping - USAGE-BASED COST
- **Purpose**: Cost tracking and scraper routing
- **Must Match**: Should align with `data_source_type`

### notes
- **Format**: Free text, supports markdown
- **Purpose**: Document special cases, configuration quirks, contact info, etc.
- **Examples**:
  - "Products organized by category folders"
  - "Contact: john@vendor.com for access issues"
  - "Requires VPN access"
  - "Sync only on weekends due to rate limits"

## Relationships

### Foreign Key References
- `vendor_id` is referenced by:
  - `product_data.vendor_id` (one-to-many)
  - `sync_log.vendor_id` (one-to-many)

## Indexes

### Recommended Indexes
1. **Primary Key**: `vendor_id` (automatic)
2. **Sync Status**: `sync_status` (for filtering active/failed syncs)
3. **Tier**: `tier_used` (for cost analysis and tier-specific queries)
4. **Last Sync**: `last_sync_date` (for finding stale data)

## Usage Examples

### Adding a New Vendor (Tier 1 - Google Drive)
```json
{
  "vendor_id": "vendor_001",
  "vendor_name": "Acme Products",
  "data_source_type": "drive",
  "source_url": "https://drive.google.com/drive/folders/1ABC123XYZ",
  "credentials": {
    "type": "oauth2",
    "refresh_token": "ENCRYPTED_TOKEN"
  },
  "last_sync_date": null,
  "sync_status": "pending",
  "tier_used": 1,
  "notes": "Products organized in subfolders by category. Images in /images, videos in /videos."
}
```

### Adding a New Vendor (Tier 2 - Shopify)
```json
{
  "vendor_id": "vendor_002",
  "vendor_name": "Beta Supplies",
  "data_source_type": "shopify",
  "source_url": "https://beta-supplies.myshopify.com",
  "credentials": null,
  "last_sync_date": null,
  "sync_status": "pending",
  "tier_used": 2,
  "notes": "Public Shopify store, no credentials needed. Rate limit: 2 req/sec."
}
```

### Adding a New Vendor (Tier 3 - Website)
```json
{
  "vendor_id": "vendor_005",
  "vendor_name": "Epsilon Wholesale",
  "data_source_type": "website",
  "source_url": "https://epsilon-wholesale.com/catalog",
  "credentials": null,
  "last_sync_date": null,
  "sync_status": "pending",
  "tier_used": 3,
  "notes": "Requires Zyte API + Gemini AI. Product pages follow pattern: /product/{sku}"
}
```

## n8n Implementation Notes

### Creating the Table in n8n
1. Navigate to **Data** → **Data Tables** in n8n Cloud
2. Click **Create Table**
3. Name: `vendor_config`
4. Add fields according to schema above
5. Set `vendor_id` as Primary Key
6. Enable encryption for `credentials` field

### Querying the Table
```javascript
// Get all vendors needing sync (older than 7 days)
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const vendors = $('Data Table').all().filter(v => 
  !v.last_sync_date || new Date(v.last_sync_date) < sevenDaysAgo
);
```

### Updating Sync Status
```javascript
// Update vendor after successful sync
$('Data Table').update({
  vendor_id: 'vendor_001',
  last_sync_date: new Date().toISOString(),
  sync_status: 'success'
});
```

## Validation Rules

### Required Field Validation
- `vendor_id`: Must match pattern `vendor_\d{3,}`
- `vendor_name`: Must not be empty
- `data_source_type`: Must be one of: drive, dropbox, sheets, shopify, website
- `tier_used`: Must be 1, 2, or 3
- `tier_used` must align with `data_source_type`:
  - drive/dropbox/sheets → tier 1
  - shopify → tier 2
  - website → tier 3

### Data Integrity
- `source_url` should be a valid URL if provided
- `credentials` must be valid JSON if provided
- `last_sync_date` must be a valid ISO 8601 datetime if provided
- `sync_status` must be one of: pending, in_progress, success, failed

## Migration Notes

### Initial Population
When first setting up the system:
1. Import vendor data from CSV/JSON template
2. Set all `sync_status` to 'pending'
3. Set all `last_sync_date` to null
4. Verify `tier_used` matches `data_source_type`
5. Encrypt all `credentials` fields
6. Run test sync on 2-3 vendors before full deployment

### Adding New Vendors
1. Assign next available `vendor_id`
2. Verify data source accessibility
3. Test credentials if applicable
4. Set `sync_status` to 'pending'
5. Run manual sync to validate configuration

## Security Considerations

### Credential Storage
- **NEVER** store credentials in plain text
- Use n8n's built-in encryption for the `credentials` field
- Consider using n8n's Credential system instead of storing in table
- Rotate credentials regularly (quarterly recommended)

### Access Control
- Limit access to vendor_config table to authorized workflows only
- Use n8n's role-based access control
- Audit credential access regularly
- Log all credential updates

## Troubleshooting

### Common Issues

**Issue**: Vendor sync fails repeatedly
- Check `sync_status` and `last_sync_date`
- Review `sync_log` table for error details
- Verify `source_url` is still valid
- Test credentials manually
- Check if vendor changed their data structure

**Issue**: Credentials not working
- Verify credentials are properly encrypted
- Check if tokens have expired (OAuth refresh needed)
- Verify API keys are still active
- Check if vendor changed authentication method

**Issue**: Wrong tier assigned
- Verify `tier_used` matches `data_source_type`
- Update tier if vendor changed data source
- Re-sync vendor after tier change

## Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-19 | Initial schema definition |

---

**Related Documentation:**
- [Product Data Schema](./product_data.md)
- [Sync Log Schema](./sync_log.md)
- [Vendor Configuration Templates](../../templates/vendor_config_template.json)
- [Setup Guide](../guides/SETUP_GUIDE.md)

