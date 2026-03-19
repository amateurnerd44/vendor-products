# Vendor Configuration Templates

This directory contains template files for configuring the 70 vendors in the Vendor Product Data Collection System.

## Files

### `vendor_config_template.sql`
SQL INSERT statements for populating the `vendor_config` table in n8n Data Tables.

**Usage:**
1. Open the file in a text editor
2. Replace all placeholders (marked with `[BRACKETS]`) with actual vendor information
3. Execute the SQL in n8n Data Tables or via n8n's database interface

### `vendor_config_template.csv`
CSV format for easier editing in spreadsheet applications (Excel, Google Sheets, etc.).

**Usage:**
1. Open in Excel/Google Sheets
2. Replace all placeholders with actual vendor information
3. Import into n8n Data Tables via CSV import feature

## Placeholder Guide

### Common Placeholders

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `[VENDOR_NAME_X]` | Official vendor business name | "Acme Board Games" |
| `[VENDOR_SLUG]` | Lowercase hyphenated ID | "acme-board-games" |

### Tier 1 Placeholders (Drive/Dropbox/Sheets)

**Google Drive:**
- `[GOOGLE_DRIVE_FOLDER_ID_X]` → Folder ID from Drive URL
- `[FOLDER_ID_X]` → Same as above (for credentials JSON)

**Dropbox:**
- `[DROPBOX_FOLDER_PATH_X]` → Path to folder (e.g., "/Vendors/AcmeGames")
- `[PATH_X]` → Same as above (for credentials JSON)

**Google Sheets:**
- `[GOOGLE_SHEET_ID_X]` → Sheet ID from Sheets URL
- `[SHEET_ID_X]` → Same as above (for credentials JSON)

### Tier 2 Placeholders (Shopify)

- `[SHOPIFY_STORE_X]` → Shopify store subdomain
- `[STORE_X]` → Same as above (for credentials JSON)

**Example:**
- If store URL is `https://acmegames.myshopify.com`
- Replace `[SHOPIFY_STORE_X]` with `acmegames`

### Tier 3 Placeholders (Website Scraping)

- `[VENDOR_WEBSITE_X]` → Vendor website domain
- `[WEBSITE_X]` → Same as above (for credentials JSON)

**Example:**
- If website is `https://acmegames.com/products`
- Replace `[VENDOR_WEBSITE_X]` with `acmegames`

## Vendor Distribution

The templates are pre-configured for the recommended tier distribution:

- **Tier 1 (30 vendors):** FREE - Drive/Dropbox/Sheets
  - 10 Google Drive vendors
  - 10 Dropbox vendors
  - 10 Google Sheets vendors
- **Tier 2 (20 vendors):** FREE - Shopify stores
- **Tier 3 (20 vendors):** PAID (~$4.26/week each) - Web scraping

## Step-by-Step Instructions

### 1. Gather Vendor Information

For each vendor, collect:
- ✅ Vendor name
- ✅ Data source type (Drive/Dropbox/Sheets/Shopify/Website)
- ✅ Source URL or ID
- ✅ Credentials (if applicable)
- ✅ Estimated product count

### 2. Determine Tier Assignment

Use the tier selection decision tree in `docs/deployment/VENDOR_ONBOARDING_GUIDE.md`:

- **Tier 1:** Vendor provides structured data via Drive/Dropbox/Sheets
- **Tier 2:** Vendor has Shopify/WooCommerce store
- **Tier 3:** Vendor has custom website requiring scraping

### 3. Fill in Templates

**Option A: Use SQL Template**
1. Open `vendor_config_template.sql`
2. Find the appropriate section (Tier 1/2/3)
3. Replace placeholders with actual values
4. Save the file

**Option B: Use CSV Template**
1. Open `vendor_config_template.csv` in Excel/Google Sheets
2. Replace placeholders in each row
3. Save as CSV

### 4. Import to n8n

**SQL Method:**
1. Copy the filled SQL statements
2. Open n8n Data Tables
3. Navigate to `vendor_config` table
4. Execute SQL or use import feature

**CSV Method:**
1. Open n8n Data Tables
2. Navigate to `vendor_config` table
3. Click "Import CSV"
4. Upload your filled CSV file

### 5. Verify Import

Run this query in n8n to verify:

```sql
SELECT tier_used, COUNT(*) as vendor_count
FROM vendor_config
GROUP BY tier_used;
```

Expected output:
```
tier_used | vendor_count
----------|-------------
    1     |     30
    2     |     20
    3     |     20
```

## Example: Filling in a Tier 1 Vendor

**Before:**
```sql
('vendor-drive-01', '[VENDOR_NAME_1]', 'drive', '[GOOGLE_DRIVE_FOLDER_ID_1]', 
 '{"drive_folder_id": "[FOLDER_ID_1]", "oauth_credential": "Google Drive - Vendor Sync"}', 
 1, false, 'pending', 'Replace with actual vendor details')
```

**After:**
```sql
('acme-board-games', 'Acme Board Games', 'drive', '1a2b3c4d5e6f7g8h9i0j', 
 '{"drive_folder_id": "1a2b3c4d5e6f7g8h9i0j", "oauth_credential": "Google Drive - Vendor Sync"}', 
 1, false, 'pending', 'Board game distributor - 50 products')
```

## Example: Filling in a Tier 2 Vendor

**Before:**
```sql
('vendor-shopify-01', '[VENDOR_NAME_31]', 'shopify', 'https://[SHOPIFY_STORE_1].myshopify.com', 
 '{"shopify_domain": "[STORE_1].myshopify.com"}', 
 2, true, 'pending', 'Replace with actual Shopify store domain')
```

**After:**
```sql
('acme-board-games', 'Acme Board Games', 'shopify', 'https://acmegames.myshopify.com', 
 '{"shopify_domain": "acmegames.myshopify.com"}', 
 2, true, 'pending', 'Board game retailer - Shopify store with 75 products')
```

## Example: Filling in a Tier 3 Vendor

**Before:**
```sql
('vendor-website-01', '[VENDOR_NAME_51]', 'website', 'https://[VENDOR_WEBSITE_1].com/products', 
 '{"website_url": "https://[WEBSITE_1].com/products", "zyte_credential": "Zyte API", "gemini_credential": "Gemini API"}', 
 3, false, 'pending', 'Replace with actual website URL')
```

**After:**
```sql
('acme-board-games', 'Acme Board Games', 'website', 'https://acmegames.com/products', 
 '{"website_url": "https://acmegames.com/products", "zyte_credential": "Zyte API", "gemini_credential": "Gemini API"}', 
 3, false, 'pending', 'Board game manufacturer - custom website with 100 products')
```

## Tips

### Finding Google Drive Folder IDs
1. Open the folder in Google Drive
2. Look at the URL: `https://drive.google.com/drive/folders/[FOLDER_ID]`
3. Copy the `[FOLDER_ID]` part

### Finding Google Sheet IDs
1. Open the sheet in Google Sheets
2. Look at the URL: `https://docs.google.com/spreadsheets/d/[SHEET_ID]/edit`
3. Copy the `[SHEET_ID]` part

### Finding Shopify Store Domains
1. Visit the vendor's Shopify store
2. The domain is usually `[store-name].myshopify.com`
3. You can also use custom domains (e.g., `shop.acmegames.com`)

### Testing Shopify Stores
Before adding a Shopify vendor, test the JSON endpoint:
```bash
curl https://[store-name].myshopify.com/products.json
```

If it returns JSON with products, the store is accessible!

## Troubleshooting

### Issue: SQL Import Fails
- **Cause:** Invalid JSON in credentials field
- **Solution:** Ensure JSON is properly escaped (use double quotes, escape special characters)

### Issue: CSV Import Fails
- **Cause:** Incorrect CSV format
- **Solution:** Ensure CSV uses commas as delimiters, quotes around JSON fields

### Issue: Vendor Not Syncing
- **Cause:** Incorrect credentials or source URL
- **Solution:** Verify credentials in n8n, test data source manually

## Next Steps

After filling in the templates:

1. ✅ Import vendors to n8n Data Tables
2. ✅ Configure API credentials in n8n (Drive, Dropbox, Sheets, Zyte, Gemini)
3. ✅ Test 3 vendors per tier (9 total) with manual sync
4. ✅ Run initial sync with all 70 vendors
5. ✅ Verify >95% success rate
6. ✅ Deploy to production

See `docs/deployment/DEPLOYMENT_CHECKLIST.md` for complete deployment procedures.

---

**Last Updated:** March 19, 2026  
**Version:** 1.0 (Pre-Deployment)

