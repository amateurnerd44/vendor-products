# Tier 1 Scrapers Test Plan

## Test Environment Setup

### Prerequisites
- n8n Cloud account with active subscription
- Google Drive OAuth2 credentials configured
- Dropbox OAuth2 credentials configured
- Google Sheets OAuth2 credentials configured
- n8n Data Tables created (product_data, vendor_config, sync_log)

---

## Test 1: Google Drive Scraper - Pattern A

### Setup
1. Create a test Google Drive folder
2. Create the following structure:
```
/test-drive-vendor/
  ├── TESTSKU001/
  │   ├── product_image_1.jpg
  │   ├── product_image_2.png
  │   ├── demo_video.mp4
  │   └── product_description.pdf
  ├── TESTSKU002/
  │   ├── main_photo.jpg
  │   └── specs.pdf
```

### Test Steps
1. Import `google_drive_scraper.json` into n8n
2. Configure Google Drive credentials
3. Set input data:
```json
{
  "vendor_config": {
    "source_url": "YOUR_TEST_FOLDER_ID",
    "vendor_id": "TEST_VENDOR_DRIVE_A"
  }
}
```
4. Execute workflow
5. Check Product Data table

### Expected Results
- ✅ 2 products created (TESTSKU001, TESTSKU002)
- ✅ TESTSKU001 has 2 images, 1 video, 1 document
- ✅ TESTSKU002 has 1 image, 0 videos, 1 document
- ✅ All URLs are shareable Google Drive links
- ✅ `pattern_type: "A"` in metadata
- ✅ `source: "google_drive"` in metadata

---

## Test 2: Google Drive Scraper - Pattern B

### Setup
1. Create a test Google Drive folder
2. Create flat structure:
```
/test-drive-vendor-flat/
  ├── TESTSKU003_image1.jpg
  ├── TESTSKU003_image2.jpg
  ├── TESTSKU003_video.mp4
  ├── TESTSKU004_photo.png
  └── TESTSKU004_description.pdf
```

### Test Steps
1. Use same workflow as Test 1
2. Update input data with new folder ID
3. Execute workflow
4. Check Product Data table

### Expected Results
- ✅ 2 products created (TESTSKU003, TESTSKU004)
- ✅ TESTSKU003 has 2 images, 1 video
- ✅ TESTSKU004 has 1 image, 1 document
- ✅ `pattern_type: "B"` in metadata

---

## Test 3: Dropbox Scraper - Pattern A

### Setup
1. Create a test Dropbox folder
2. Create structure:
```
/test-dropbox-vendor/
  ├── TESTSKU005/
  │   ├── product.jpg
  │   └── video.mp4
  ├── TESTSKU006/
  │   └── image.png
```

### Test Steps
1. Import `dropbox_scraper.json` into n8n
2. Configure Dropbox credentials
3. Set input data:
```json
{
  "vendor_config": {
    "source_url": "/test-dropbox-vendor",
    "vendor_id": "TEST_VENDOR_DROPBOX_A"
  }
}
```
4. Execute workflow
5. Check Product Data table

### Expected Results
- ✅ 2 products created (TESTSKU005, TESTSKU006)
- ✅ TESTSKU005 has 1 image, 1 video
- ✅ TESTSKU006 has 1 image
- ✅ All URLs end with `?dl=1`
- ✅ URLs are direct download links
- ✅ `source: "dropbox"` in metadata

---

## Test 4: Dropbox Scraper - Pattern B

### Setup
1. Create flat Dropbox structure:
```
/test-dropbox-vendor-flat/
  ├── TESTSKU007_photo1.jpg
  ├── TESTSKU007_photo2.jpg
  ├── TESTSKU008_image.png
```

### Test Steps
1. Use same workflow as Test 3
2. Update input data with new folder path
3. Execute workflow
4. Check Product Data table

### Expected Results
- ✅ 2 products created (TESTSKU007, TESTSKU008)
- ✅ TESTSKU007 has 2 images
- ✅ TESTSKU008 has 1 image
- ✅ `pattern_type: "B"` in metadata

---

## Test 5: Google Sheets Scraper - Standard Format

### Setup
1. Create a test Google Sheet
2. Add data:

| SKU | Name | Images | Videos | Description |
|-----|------|--------|--------|-------------|
| TESTSKU009 | Test Product 9 | https://example.com/img1.jpg,https://example.com/img2.jpg | https://example.com/video.mp4 | This is a test product with multiple images and a video. |
| TESTSKU010 | Test Product 10 | https://example.com/img3.jpg | | Short description for product 10. |

### Test Steps
1. Import `google_sheets_scraper.json` into n8n
2. Configure Google Sheets credentials
3. Set input data:
```json
{
  "vendor_config": {
    "source_url": "YOUR_SPREADSHEET_ID",
    "vendor_id": "TEST_VENDOR_SHEETS"
  }
}
```
4. Execute workflow
5. Check Product Data table

### Expected Results
- ✅ 2 products created (TESTSKU009, TESTSKU010)
- ✅ TESTSKU009 has 2 images, 1 video, marketing copy
- ✅ TESTSKU010 has 1 image, 0 videos, marketing copy
- ✅ Comma-separated URLs are split correctly
- ✅ Column detection worked automatically
- ✅ `source: "google_sheets"` in metadata

---

## Test 6: Google Sheets Scraper - Variations

### Setup
1. Create a new sheet with different column names:

| Item | Product Name | Image URLs | Video URLs | Copy |
|------|--------------|------------|------------|------|
| TESTSKU011 | Product Eleven | https://example.com/img.jpg | https://example.com/vid.mp4 | Marketing copy here. |

### Test Steps
1. Use same workflow as Test 5
2. Update input data with new spreadsheet ID
3. Execute workflow
4. Check Product Data table

### Expected Results
- ✅ 1 product created (TESTSKU011)
- ✅ Column auto-detection worked with variations
- ✅ "Item" detected as SKU
- ✅ "Product Name" detected as name
- ✅ "Copy" detected as description

---

## Test 7: Data Normalizer - Clean Data

### Setup
1. Manually insert test data with issues:
```json
{
  "sku": "  test-sku-012  ",
  "product_name": "  Product   with   extra   spaces  ",
  "image_urls": [
    "https://example.com/img.jpg",
    "invalid-url",
    "  https://example.com/img2.jpg  "
  ],
  "video_urls": ["https://example.com/video.mp4"],
  "marketing_copy": "<p>HTML <b>tags</b> &amp; entities</p>",
  "vendor_id": "TEST_VENDOR_NORM"
}
```

### Test Steps
1. Import `data_normalizer.json` into n8n
2. Pass test data as input
3. Execute workflow
4. Check output

### Expected Results
- ✅ SKU cleaned: `TEST-SKU-012` (uppercase, trimmed)
- ✅ Product name normalized: `Product with extra spaces`
- ✅ Invalid URL removed
- ✅ Valid URLs trimmed
- ✅ HTML tags removed from marketing copy
- ✅ HTML entities decoded: `&` instead of `&amp;`
- ✅ Quality score calculated
- ✅ Metadata added

---

## Test 8: Data Normalizer - Quality Check

### Setup
1. Create test data with varying quality:

**High Quality (Score: 100)**
```json
{
  "sku": "HIGHQUAL001",
  "product_name": "High Quality Product",
  "image_urls": ["https://example.com/img.jpg"],
  "video_urls": ["https://example.com/video.mp4"],
  "marketing_copy": "This is a detailed marketing copy with more than 50 characters to meet the quality threshold.",
  "vendor_id": "TEST"
}
```

**Low Quality (Score: 20)**
```json
{
  "sku": "LOWQUAL001",
  "product_name": "",
  "image_urls": [],
  "video_urls": [],
  "marketing_copy": "",
  "vendor_id": "TEST"
}
```

### Test Steps
1. Run data normalizer on both items
2. Check Product Data table
3. Check Sync Log table

### Expected Results
- ✅ High quality item saved to Product Data table
- ✅ High quality score: 100
- ✅ Low quality item NOT saved to Product Data table
- ✅ Low quality item logged in Sync Log table
- ✅ Low quality score: 20

---

## Test 9: End-to-End Integration

### Setup
1. Create complete test vendor setup:
   - Google Drive folder with products
   - Vendor config entry in table

### Test Steps
1. Add vendor to vendor_config table:
```json
{
  "vendor_id": "E2E_TEST_VENDOR",
  "vendor_name": "End-to-End Test Vendor",
  "data_source_type": "drive",
  "source_url": "YOUR_TEST_FOLDER_ID",
  "tier_used": 1
}
```
2. Run Google Drive Scraper
3. Verify data in Product Data table
4. Run Data Normalizer on results
5. Check final data quality

### Expected Results
- ✅ All products extracted from Drive
- ✅ Data normalized and cleaned
- ✅ Quality scores calculated
- ✅ High-quality data saved
- ✅ Low-quality data logged
- ✅ Metadata complete

---

## Test 10: Error Handling

### Test Cases

**A. Invalid Folder ID**
- Input: Non-existent folder ID
- Expected: Error logged, workflow stops gracefully

**B. No Access Permission**
- Input: Folder without proper sharing
- Expected: Permission error, logged to sync log

**C. Empty Folder**
- Input: Folder with no files
- Expected: No products created, no errors

**D. Malformed Sheet**
- Input: Sheet with no SKU column
- Expected: No products created, column detection fails gracefully

**E. Invalid URLs**
- Input: Product with all invalid URLs
- Expected: Product created but with empty URL arrays

---

## Test Results Template

| Test # | Test Name | Status | Notes |
|--------|-----------|--------|-------|
| 1 | Drive Pattern A | ⬜ | |
| 2 | Drive Pattern B | ⬜ | |
| 3 | Dropbox Pattern A | ⬜ | |
| 4 | Dropbox Pattern B | ⬜ | |
| 5 | Sheets Standard | ⬜ | |
| 6 | Sheets Variations | ⬜ | |
| 7 | Normalizer Clean | ⬜ | |
| 8 | Normalizer Quality | ⬜ | |
| 9 | End-to-End | ⬜ | |
| 10 | Error Handling | ⬜ | |

**Legend:**
- ⬜ Not Started
- 🟡 In Progress
- ✅ Passed
- ❌ Failed

---

## Performance Benchmarks

Record actual performance during testing:

| Workflow | Items Tested | Time Taken | Items/Second |
|----------|--------------|------------|--------------|
| Google Drive | | | |
| Dropbox | | | |
| Google Sheets | | | |
| Data Normalizer | | | |

---

## Issues Found

Document any issues discovered during testing:

| Issue # | Description | Severity | Status | Resolution |
|---------|-------------|----------|--------|------------|
| | | | | |

**Severity Levels:**
- 🔴 Critical - Blocks functionality
- 🟡 Major - Significant impact
- 🟢 Minor - Small issue
- 🔵 Enhancement - Nice to have

---

**Test Plan Version:** 1.0  
**Last Updated:** 2026-03-19  
**Tester:** [Your Name]  
**Phase:** 2 - Tier 1 Scrapers

