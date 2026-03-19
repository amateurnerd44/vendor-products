# Vendor Management Strategy

## Overview

This system uses **Shopify's built-in vendor management** instead of maintaining a separate `vendor_config` table. This approach simplifies the data model and leverages Shopify's existing vendor relationships.

---

## Why Shopify Vendor Management?

### Advantages ✅
- **No duplicate data** - Vendor information lives in one place (Shopify)
- **Automatic sync** - Vendor changes in Shopify automatically propagate
- **Simpler architecture** - One less table to maintain
- **Native integration** - Works seamlessly with Shopify workflows
- **Vendor portal** - Vendors can manage their own information in Shopify

### Trade-offs ⚠️
- **Shopify dependency** - Requires Shopify for vendor management
- **Limited customization** - Constrained by Shopify's vendor fields
- **No vendor credentials** - Cannot store vendor-specific API keys/tokens in Shopify

---

## Data Model

### Product Table Schema
Products link to vendors via `Shopify_Vendor_Id`:

```
product_data
├── id (primary key)
├── SKU
├── Name
├── ... (other product fields)
└── Shopify_Vendor_Id ← Links to Shopify vendor
```

### Shopify Vendor Data
Shopify stores vendor information:
- Vendor name
- Vendor ID
- Contact information
- Payment terms
- Shipping preferences

---

## Vendor Operations

### Get All Products from a Vendor

```javascript
// n8n workflow
const vendorId = '12345678';
const vendorProducts = $('Data Table').all().filter(product => 
  product.Shopify_Vendor_Id === vendorId
);
```

### Get List of All Vendors

```javascript
// Get unique vendor IDs from products
const vendorIds = [...new Set(
  $('Data Table').all()
    .map(p => p.Shopify_Vendor_Id)
    .filter(id => id !== null && id !== '')
)];

// Fetch vendor details from Shopify
const vendors = [];
for (const vendorId of vendorIds) {
  const vendor = await $('Shopify').getVendor(vendorId);
  vendors.push(vendor);
}
```

### Add New Vendor

1. **Create vendor in Shopify**:
   - Go to Shopify Admin > Products > Vendors
   - Click "Add vendor"
   - Enter vendor details
   - Save

2. **Note the Vendor ID** from Shopify

3. **Add products** with `Shopify_Vendor_Id` set to the new vendor's ID

### Update Vendor Information

Update vendor details directly in Shopify:
- Shopify Admin > Products > Vendors > [Select Vendor]
- Make changes
- Save

Changes automatically propagate to all products via `Shopify_Vendor_Id`.

---

## Vendor Data Sources

### Tier 1: Vendor-Supplied Data (Drive/Dropbox/Sheets)

For vendors who provide data files:

1. **Store credentials separately** (not in Shopify)
   - Use n8n Credentials for OAuth tokens
   - Use environment variables for API keys
   - Use secure credential storage

2. **Map vendor to data source**:
   ```javascript
   // Configuration mapping (stored in n8n workflow or env vars)
   const vendorDataSources = {
     '12345678': {  // Shopify_Vendor_Id
       type: 'google_drive',
       folder_id: 'ABC123XYZ',
       credential: 'google_drive_oauth'
     },
     '87654321': {
       type: 'dropbox',
       folder_path: '/vendor_products',
       credential: 'dropbox_token'
     }
   };
   ```

3. **Sync workflow**:
   - Loop through vendor data sources
   - Fetch data from Drive/Dropbox/Sheets
   - Parse and normalize data
   - Update products with matching `Shopify_Vendor_Id`

### Tier 2: Shopify Data

Shopify products already have vendor information:

```javascript
// Fetch products from Shopify
const shopifyProducts = await $('Shopify').getProducts();

// Each product has vendor info
shopifyProducts.forEach(product => {
  const vendorId = product.vendor_id;
  const vendorName = product.vendor;
  // Update local product_data table
});
```

### Tier 3: Zyte + AI Scraping

For vendors with websites:

1. **Map vendor to website**:
   ```javascript
   const vendorWebsites = {
     '12345678': 'https://vendor-website.com/products',
     '87654321': 'https://another-vendor.com/catalog'
   };
   ```

2. **Scrape workflow**:
   - Get vendor website URL
   - Use Zyte API to fetch pages
   - Use Gemini AI to extract product data
   - Update products with matching `Shopify_Vendor_Id`

---

## Vendor Configuration Storage

Since we're not using a `vendor_config` table, store vendor-specific configuration in:

### Option 1: n8n Workflow Variables
```javascript
// In n8n workflow
const VENDOR_CONFIG = {
  '12345678': {
    name: 'Acme Products',
    data_source: 'google_drive',
    folder_id: 'ABC123',
    sync_frequency: 'weekly',
    tier: 1
  },
  '87654321': {
    name: 'Beta Supplies',
    data_source: 'shopify',
    tier: 2
  }
};
```

### Option 2: Environment Variables
```bash
# .env file
VENDOR_12345678_NAME="Acme Products"
VENDOR_12345678_SOURCE="google_drive"
VENDOR_12345678_FOLDER="ABC123"
VENDOR_12345678_TIER=1
```

### Option 3: External Configuration File
```json
// config/vendors.json
{
  "vendors": [
    {
      "shopify_vendor_id": "12345678",
      "name": "Acme Products",
      "data_source": {
        "type": "google_drive",
        "folder_id": "ABC123",
        "credential": "google_drive_oauth"
      },
      "tier": 1,
      "sync_schedule": "0 2 * * 0"
    }
  ]
}
```

**Recommended**: Use Option 3 (external config file) for:
- Easy updates without modifying workflows
- Version control
- Scalability (70+ vendors)
- Clear documentation

---

## Vendor Sync Workflow

### Weekly Sync Process

1. **Load vendor configuration**:
   ```javascript
   const vendorConfig = require('./config/vendors.json');
   ```

2. **Loop through vendors**:
   ```javascript
   for (const vendor of vendorConfig.vendors) {
     const vendorId = vendor.shopify_vendor_id;
     const dataSource = vendor.data_source;
     
     // Route to appropriate tier
     if (vendor.tier === 1) {
       await syncTier1(vendorId, dataSource);
     } else if (vendor.tier === 2) {
       await syncTier2(vendorId);
     } else if (vendor.tier === 3) {
       await syncTier3(vendorId, dataSource);
     }
   }
   ```

3. **Update products**:
   ```javascript
   // After fetching vendor data
   products.forEach(product => {
     $('Data Table').update({
       SKU: product.sku,
       Shopify_Vendor_Id: vendorId,
       // ... other fields
       updatedAt: new Date().toISOString()
     });
   });
   ```

---

## Vendor Credentials Management

### Storing Credentials Securely

**DO NOT** store credentials in:
- ❌ Product data table
- ❌ Shopify vendor fields
- ❌ Plain text files in repo

**DO** store credentials in:
- ✅ n8n Credentials (encrypted at rest)
- ✅ Environment variables (for API keys)
- ✅ External secret management (AWS Secrets Manager, etc.)

### Example: Google Drive OAuth

1. **Create credential in n8n**:
   - Name: `google_drive_vendor_acme`
   - Type: Google Drive OAuth2
   - Authorize with vendor's Google account

2. **Reference in workflow**:
   ```javascript
   // Use credential by name
   $('Google Drive').setCredential('google_drive_vendor_acme');
   ```

3. **Map to vendor**:
   ```json
   {
     "shopify_vendor_id": "12345678",
     "credential_name": "google_drive_vendor_acme"
   }
   ```

---

## Vendor Reporting

### Products by Vendor

```javascript
// Get product count by vendor
const vendorCounts = {};
$('Data Table').all().forEach(product => {
  const vendorId = product.Shopify_Vendor_Id;
  if (vendorId) {
    vendorCounts[vendorId] = (vendorCounts[vendorId] || 0) + 1;
  }
});

// Output: { '12345678': 45, '87654321': 32, ... }
```

### Vendor Sync Status

```javascript
// Track last sync per vendor
const vendorSyncStatus = {};
$('Data Table').all().forEach(product => {
  const vendorId = product.Shopify_Vendor_Id;
  const updated = new Date(product.updatedAt);
  
  if (!vendorSyncStatus[vendorId] || updated > vendorSyncStatus[vendorId]) {
    vendorSyncStatus[vendorId] = updated;
  }
});

// Find stale vendors (not synced in 7+ days)
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const staleVendors = Object.entries(vendorSyncStatus)
  .filter(([id, lastSync]) => lastSync < sevenDaysAgo)
  .map(([id]) => id);
```

---

## Migration from vendor_config Table

If you previously used a `vendor_config` table:

### Migration Steps

1. **Export vendor_config data**:
   ```javascript
   const vendors = $('Data Table').all('vendor_config');
   ```

2. **Create vendors in Shopify**:
   ```javascript
   for (const vendor of vendors) {
     const shopifyVendor = await $('Shopify').createVendor({
       name: vendor.vendor_name
     });
     
     // Map old vendor_id to new Shopify_Vendor_Id
     vendorMapping[vendor.vendor_id] = shopifyVendor.id;
   }
   ```

3. **Update products**:
   ```javascript
   $('Data Table').all('product_data').forEach(product => {
     const oldVendorId = product.vendor_id;
     const newVendorId = vendorMapping[oldVendorId];
     
     $('Data Table').update({
       id: product.id,
       Shopify_Vendor_Id: newVendorId
     });
   });
   ```

4. **Migrate vendor configuration**:
   - Export vendor data sources to `config/vendors.json`
   - Migrate credentials to n8n Credentials
   - Update workflows to use new vendor mapping

5. **Delete vendor_config table** (after verification)

---

## Best Practices

### ✅ DO

- **Use Shopify as source of truth** for vendor names and IDs
- **Store vendor-specific config** in external JSON file
- **Use n8n Credentials** for OAuth tokens and API keys
- **Document vendor data sources** clearly
- **Test vendor sync** with 2-3 vendors before full rollout
- **Monitor vendor sync status** regularly

### ❌ DON'T

- **Don't duplicate vendor data** across multiple systems
- **Don't store credentials** in product data or config files
- **Don't hardcode vendor IDs** in workflows (use config)
- **Don't skip vendor validation** when adding products
- **Don't forget to update** `Shopify_Vendor_Id` when syncing

---

## Troubleshooting

### Issue: Products missing Shopify_Vendor_Id

**Solution**:
```javascript
// Find products without vendor
const orphanProducts = $('Data Table').all().filter(p => 
  !p.Shopify_Vendor_Id
);

// Assign to default vendor or flag for review
orphanProducts.forEach(product => {
  console.log(`Product ${product.SKU} missing vendor`);
});
```

### Issue: Vendor not found in Shopify

**Solution**:
1. Verify vendor exists in Shopify Admin
2. Check Shopify_Vendor_Id is correct
3. Create vendor in Shopify if missing
4. Update products with correct vendor ID

### Issue: Vendor data source not syncing

**Solution**:
1. Check vendor configuration in `config/vendors.json`
2. Verify credentials are valid (OAuth tokens, API keys)
3. Test data source access manually
4. Check sync logs for errors
5. Verify vendor tier is correct

---

## Future Enhancements

### Potential Improvements

1. **Vendor Portal**:
   - Allow vendors to upload data directly
   - Self-service product updates
   - Real-time sync

2. **Vendor Analytics**:
   - Track vendor performance
   - Product catalog growth
   - Sync success rates

3. **Automated Vendor Onboarding**:
   - Wizard for adding new vendors
   - Automatic data source detection
   - Credential setup assistance

4. **Vendor Communication**:
   - Automated sync notifications
   - Error alerts to vendors
   - Product update confirmations

---

## Related Documentation

- [Product Data Schema](../schemas/product_data.md)
- [Setup Guide](../../SETUP_GUIDE.md)
- [Implementation Plan](../../IMPLEMENTATION_PLAN.md)
- [Shopify Integration](./shopify_integration.md)

---

**Document Version**: 1.0  
**Last Updated**: 2026-03-19  
**Author**: Codegen AI

