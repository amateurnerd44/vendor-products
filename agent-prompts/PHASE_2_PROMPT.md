# Phase 2 Agent Prompt - Tier 1 Direct Integration Scrapers

---

## 🎯 Your Mission

You are tasked with completing **Phase 2: Tier 1 - Direct Integration Scrapers** for the Vendor Product Data Collection System. You will build n8n workflows that collect product data from Google Drive, Dropbox, and Google Sheets vendors.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

### What This System Does

This system automates collection of product data (images, videos, marketing copy) from 70+ vendors and delivers digital asset packages to customers when they place orders.

**Two Main Workflows:**
1. **Flow 1 - Weekly Product Catalog Sync:** Collects data from all vendors weekly
2. **Flow 2 - Order Fulfillment:** Delivers assets when customers place orders

### Tech Stack

- **Orchestration:** n8n Cloud
- **Data Storage:** n8n Data Tables
- **Tier 1 Integrations:** Google Drive API, Dropbox API, Google Sheets API (all FREE)

### Tier 1 Strategy

**Tier 1 vendors** provide data through direct cloud storage integrations:
- **Google Drive:** Product files organized in folders
- **Dropbox:** Product files in shared folders
- **Google Sheets:** Product catalogs in spreadsheets

**Cost:** $0 (uses free APIs)

---

## 🗂️ Data Models (From Phase 1)

### Vendor Config Table Schema

```
vendor_id (Text, PRIMARY KEY)
vendor_name (Text)
data_source_type (Text) - 'drive', 'dropbox', or 'sheets'
source_url (Text) - Folder/sheet URL
credentials (JSON, ENCRYPTED)
last_sync_date (DateTime)
sync_status (Text)
tier_used (Number) - Always 1 for this phase
notes (Text)
```

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
source_tier (Number) - Always 1 for this phase
```

---

## 🎯 Phase 2 Deliverables (Your Tasks)

### Task 1: Google Drive Integration Workflow

**Create n8n workflow:** `tier1_google_drive_scraper.json`

**Workflow Steps:**
1. **Input:** Receive vendor_id and source_url (Drive folder ID)
2. **Authenticate:** Use Google Drive OAuth2 credentials
3. **List Files:** Recursively list all files in vendor folder
4. **Parse Structure:** Detect folder organization pattern:
   - Pattern A: `/products/{SKU}/images/` and `/products/{SKU}/videos/`
   - Pattern B: Flat structure with `{SKU}_image1.jpg`, `{SKU}_video1.mp4`
   - Pattern C: Custom structure (use folder metadata)
5. **Extract Data:**
   - SKU from folder name or filename
   - Product name from folder metadata or description file
   - Image URLs (get shareable links)
   - Video URLs (get shareable links)
   - Marketing copy from `description.txt` or `README.md` files
6. **Normalize Data:** Map to product_data table schema
7. **Output:** Return array of product objects

**Error Handling:**
- Handle missing folders gracefully
- Skip files that don't match expected patterns
- Log parsing errors for manual review
- Retry failed API calls (max 3 attempts)

**Example Output:**
```json
{
  "sku": "ACME-12345",
  "vendor_id": "vendor_001",
  "product_name": "Premium Widget Pro",
  "image_urls": ["https://drive.google.com/file/d/xyz789/view", "..."],
  "video_urls": ["https://drive.google.com/file/d/xyz791/view"],
  "marketing_copy": "The Premium Widget Pro revolutionizes...",
  "metadata": {"folder_path": "/products/ACME-12345"},
  "last_updated": "2026-03-19T10:00:00Z",
  "source_tier": 1
}
```

### Task 2: Dropbox Integration Workflow

**Create n8n workflow:** `tier1_dropbox_scraper.json`

**Workflow Steps:**
1. **Input:** Receive vendor_id and source_url (Dropbox shared folder path)
2. **Authenticate:** Use Dropbox OAuth2 credentials
3. **List Files:** Recursively list all files in shared folder
4. **Parse Structure:** Detect organization pattern (same as Drive)
5. **Extract Data:**
   - SKU from folder/file names
   - Product name from folder metadata
   - Image URLs (get shareable links with `dl=1` parameter)
   - Video URLs (get shareable links with `dl=1` parameter)
   - Marketing copy from text files
6. **Normalize Data:** Map to product_data table schema
7. **Output:** Return array of product objects

**Dropbox-Specific Considerations:**
- Use `files/list_folder` and `files/list_folder/continue` for pagination
- Convert Dropbox share links to direct download links (`dl=1`)
- Handle Dropbox rate limiting (max 600 requests/hour per app)
- Cache folder listings to reduce API calls

### Task 3: Google Sheets Integration Workflow

**Create n8n workflow:** `tier1_google_sheets_scraper.json`

**Workflow Steps:**
1. **Input:** Receive vendor_id and source_url (Google Sheets ID)
2. **Authenticate:** Use Google Sheets Service Account credentials
3. **Read Sheet:** Fetch all rows from main sheet (usually "Sheet1" or "Products")
4. **Detect Columns:** Auto-detect column mapping:
   - Look for columns: SKU, Product Name, Images, Videos, Description
   - Handle variations: "Product_Name", "ProductName", "Name", etc.
   - Support comma-separated URLs in single cells
5. **Parse Data:**
   - SKU (required)
   - Product name (required)
   - Image URLs (split by comma/newline if multiple)
   - Video URLs (split by comma/newline if multiple)
   - Marketing copy (from description column)
6. **Handle Multi-Sheet Structures:**
   - If separate "Images" and "Videos" sheets exist, join by SKU
   - Support relational structure: Products → Images (1:many) → Videos (1:many)
7. **Normalize Data:** Map to product_data table schema
8. **Output:** Return array of product objects

**Google Sheets-Specific Considerations:**
- Use `spreadsheets.values.get` API
- Handle empty cells gracefully
- Support multiple sheet tabs (Products, Images, Videos)
- Validate URLs before storing
- Handle merged cells and formatting

### Task 4: Data Normalization Module

**Create n8n subworkflow:** `tier1_data_normalizer.json`

**Purpose:** Standardize data from all Tier 1 sources into consistent format

**Normalization Rules:**
1. **SKU Cleaning:**
   - Trim whitespace
   - Convert to uppercase
   - Remove special characters except hyphens and underscores
   - Validate format (alphanumeric + hyphens/underscores only)

2. **URL Validation:**
   - Verify URLs are accessible (HTTP HEAD request)
   - Convert to shareable/direct download links
   - Remove tracking parameters
   - Ensure HTTPS

3. **Text Cleaning:**
   - Trim whitespace from product names and marketing copy
   - Remove HTML tags if present
   - Normalize line breaks
   - Limit marketing copy to 5000 characters

4. **Metadata Enrichment:**
   - Add `source_tier: 1`
   - Add `last_updated: <current_timestamp>`
   - Add `data_source_type` from vendor config
   - Add `extraction_method` (e.g., "drive_folder_scan")

5. **Missing Data Handling:**
   - Set empty arrays for missing image_urls/video_urls
   - Set null for missing marketing_copy
   - Log warnings for missing required fields (SKU, product_name)

**Output:** Validated and normalized product object ready for database insertion

### Task 5: Testing & Validation

**Create test workflows for each integration:**

1. **Test Google Drive:**
   - Create test Drive folder with sample products
   - Run scraper workflow
   - Verify all products extracted correctly
   - Check shareable links work

2. **Test Dropbox:**
   - Create test Dropbox folder with sample products
   - Run scraper workflow
   - Verify all products extracted correctly
   - Check download links work

3. **Test Google Sheets:**
   - Create test spreadsheet with sample products
   - Test single-sheet and multi-sheet structures
   - Run scraper workflow
   - Verify column auto-detection works

**Validation Checklist:**
- [ ] All SKUs extracted correctly
- [ ] Product names populated
- [ ] Image URLs are accessible
- [ ] Video URLs are accessible
- [ ] Marketing copy extracted (if available)
- [ ] Data normalized to schema
- [ ] No duplicate SKUs
- [ ] Error handling works for missing data

---

## 📁 Deliverable Files

Create these files in the repository:

```
vendor-products/
├── workflows/
│   ├── tier1/
│   │   ├── google_drive_scraper.json
│   │   ├── dropbox_scraper.json
│   │   ├── google_sheets_scraper.json
│   │   └── data_normalizer.json
│   └── tests/
│       ├── test_drive_scraper.json
│       ├── test_dropbox_scraper.json
│       └── test_sheets_scraper.json
└── docs/
    └── workflows/
        └── TIER1_SCRAPERS.md (documentation)
```

---

## 📖 Documentation Requirements

Create `docs/workflows/TIER1_SCRAPERS.md` with:

1. **Overview:** Purpose and scope of Tier 1 scrapers
2. **Architecture:** How the three scrapers work together
3. **Workflow Diagrams:** Visual representation of each scraper
4. **Configuration:** How to set up credentials and test
5. **Folder Structure Patterns:** Supported organization patterns
6. **Error Handling:** How errors are logged and handled
7. **Testing Guide:** How to test each scraper
8. **Troubleshooting:** Common issues and solutions

---

## ✅ Completion Checklist

- [ ] Google Drive scraper workflow created and tested
- [ ] Dropbox scraper workflow created and tested
- [ ] Google Sheets scraper workflow created and tested
- [ ] Data normalizer subworkflow created
- [ ] All workflows handle errors gracefully
- [ ] Test workflows created for each integration
- [ ] Documentation created
- [ ] Workflows exported as JSON files
- [ ] All files committed to repository

---

## 📊 Success Metrics

- **Time to Complete:** 6-8 hours
- **Workflows Created:** 4 (3 scrapers + 1 normalizer)
- **Test Workflows:** 3
- **Supported Folder Patterns:** 3+ per integration
- **Error Handling:** Comprehensive (retries, logging, graceful failures)
- **Data Validation:** 100% of extracted products pass normalization

---

## 🚀 Next Steps After Phase 2

- **Phase 3:** Tier 2 Scrapers (Shopify) - can run in parallel
- **Phase 4:** Tier 3 Scrapers (Zyte + AI) - can run in parallel
- **Phase 5:** Weekly Sync Workflow (requires Phases 2, 3, 4 complete)

---

## 💡 Implementation Tips

1. **Start with Google Sheets** - easiest to test and validate
2. **Use n8n's built-in nodes** - Google Drive, Dropbox, Google Sheets nodes available
3. **Test incrementally** - test each step before moving to next
4. **Handle edge cases** - empty folders, missing files, malformed data
5. **Log everything** - detailed logging helps debugging
6. **Use subworkflows** - modularize for reusability

---

**Phase:** 2 of 8  
**Dependencies:** Phase 1 (data models)  
**Can Run in Parallel With:** Phases 3, 4  
**Estimated Time:** 6-8 hours

