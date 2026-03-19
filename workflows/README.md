# Workflows Directory

This directory contains all n8n workflow JSON files for the Vendor Product Data Collection System.

## Directory Structure

```
workflows/
├── tier1/                      # Tier 1 scrapers (free APIs)
│   ├── google_drive_scraper.json
│   ├── dropbox_scraper.json
│   ├── google_sheets_scraper.json
│   └── data_normalizer.json
├── tier2/                      # Tier 2 scrapers (Shopify) - Coming in Phase 3
├── tier3/                      # Tier 3 scrapers (Zyte + AI) - Coming in Phase 4
├── flows/                      # Main orchestration workflows - Coming in Phase 5 & 6
└── tests/                      # Test plans and test workflows
    └── test_tier1_scrapers.md
```

## Workflow Import Instructions

### Step 1: Access n8n Cloud
1. Log in to your n8n Cloud account
2. Navigate to your workspace

### Step 2: Import Workflow
1. Click "Add Workflow" or "+" button
2. Select "Import from File"
3. Choose the JSON file from this directory
4. Click "Import"

### Step 3: Configure Credentials
Each workflow requires specific credentials:

**Google Drive Scraper:**
- Google Drive OAuth2 API

**Dropbox Scraper:**
- Dropbox OAuth2 API

**Google Sheets Scraper:**
- Google Sheets OAuth2 API

**Data Normalizer:**
- No external credentials needed (uses n8n Data Tables)

### Step 4: Set Up Data Tables
Ensure these n8n Data Tables exist:
- `product_data` - Stores product information
- `vendor_config` - Stores vendor configurations
- `sync_log` - Stores sync logs

See `/docs/schemas/` for table schemas.

### Step 5: Test Workflow
1. Click "Execute Workflow" button
2. Provide test input data
3. Verify output in Data Tables

---

## Tier 1 Workflows

### Google Drive Scraper
**File:** `tier1/google_drive_scraper.json`

**Purpose:** Extract product data from Google Drive folders

**Input:**
```json
{
  "vendor_config": {
    "source_url": "DRIVE_FOLDER_ID",
    "vendor_id": "VENDOR_001"
  }
}
```

**Output:** Product data saved to `product_data` table

**Documentation:** See `/docs/workflows/TIER1_SCRAPERS.md`

---

### Dropbox Scraper
**File:** `tier1/dropbox_scraper.json`

**Purpose:** Extract product data from Dropbox folders

**Input:**
```json
{
  "vendor_config": {
    "source_url": "/path/to/vendor/folder",
    "vendor_id": "VENDOR_002"
  }
}
```

**Output:** Product data saved to `product_data` table

**Documentation:** See `/docs/workflows/TIER1_SCRAPERS.md`

---

### Google Sheets Scraper
**File:** `tier1/google_sheets_scraper.json`

**Purpose:** Extract product data from Google Sheets

**Input:**
```json
{
  "vendor_config": {
    "source_url": "SPREADSHEET_ID",
    "vendor_id": "VENDOR_003"
  }
}
```

**Output:** Product data saved to `product_data` table

**Documentation:** See `/docs/workflows/TIER1_SCRAPERS.md`

---

### Data Normalizer
**File:** `tier1/data_normalizer.json`

**Purpose:** Clean and standardize data from all sources

**Input:** Raw product data from any scraper

**Output:** Normalized product data with quality scores

**Documentation:** See `/docs/workflows/TIER1_SCRAPERS.md`

---

## Workflow Execution Order

### For Weekly Sync (Flow 1 - Phase 5)
1. Read vendor from `vendor_config` table
2. Route to appropriate Tier 1 scraper based on `data_source_type`
3. Scraper extracts data and saves to `product_data` table
4. Data Normalizer runs automatically (can be integrated into each scraper)
5. Log results to `sync_log` table

### For Order Fulfillment (Flow 2 - Phase 6)
1. Receive order trigger
2. Query `product_data` table for SKUs
3. If SKU not found, trigger on-demand scraping
4. Return product data for fulfillment

---

## Workflow Tags

All workflows are tagged for easy filtering:

- `tier1` - Tier 1 scrapers
- `tier2` - Tier 2 scrapers (coming soon)
- `tier3` - Tier 3 scrapers (coming soon)
- `scraper` - Data extraction workflows
- `normalizer` - Data cleaning workflows
- `orchestrator` - Main flow workflows (coming soon)

---

## Workflow Versioning

Workflows follow semantic versioning in their metadata:

```json
{
  "meta": {
    "version": "1.0.0",
    "instanceId": "n8n-cloud"
  }
}
```

**Version Format:** `MAJOR.MINOR.PATCH`
- **MAJOR:** Breaking changes
- **MINOR:** New features, backward compatible
- **PATCH:** Bug fixes

---

## Troubleshooting

### Common Issues

**Issue: "Workflow import failed"**
- Ensure JSON file is valid
- Check n8n version compatibility
- Verify all required nodes are available

**Issue: "Credentials not found"**
- Configure credentials before running workflow
- Ensure credential names match workflow expectations

**Issue: "Data table not found"**
- Create required data tables first
- See `/docs/schemas/n8n-data-tables.md`

**Issue: "Execution timeout"**
- Increase workflow timeout in settings
- Optimize workflow for large datasets
- Consider batch processing

---

## Best Practices

### Workflow Development
1. **Test with small datasets first**
2. **Use error handling nodes**
3. **Log important events**
4. **Add descriptive node names**
5. **Document complex logic in code nodes**

### Credential Management
1. **Use OAuth2 when possible**
2. **Never hardcode credentials**
3. **Rotate credentials regularly**
4. **Use separate credentials for testing**

### Performance Optimization
1. **Batch API requests**
2. **Use pagination for large datasets**
3. **Implement rate limiting**
4. **Cache frequently accessed data**

---

## Contributing

When adding new workflows:

1. **Follow naming convention:** `{tier}_{source}_{action}.json`
2. **Add comprehensive documentation**
3. **Include test cases**
4. **Tag appropriately**
5. **Update this README**

---

## Support

For issues or questions:
- Check `/docs/workflows/TIER1_SCRAPERS.md`
- Review test plans in `/workflows/tests/`
- Consult main `IMPLEMENTATION_PLAN.md`

---

**Directory Version:** 1.0  
**Last Updated:** 2026-03-19  
**Phase:** 2 - Tier 1 Scrapers Complete

