# Tier 1 Media Extraction Tests
## Google Drive, Dropbox, Google Sheets

**Test Suite:** Phase 7 - Tier 1 Scrapers with Media Asset Collection  
**Date:** 2026-03-19  
**Status:** Ready for Execution

---

## 🎯 Test Objectives

Validate that all Tier 1 scrapers (Google Drive, Dropbox, Google Sheets) correctly:
1. Extract product data (all 31 existing fields)
2. Extract image URLs and populate `image_urls` JSON array
3. Extract video URLs and populate `video_urls` JSON array
4. Assign correct `vendor_id` from vendor_config
5. Set `source_tier = 1` for all Tier 1 products
6. Handle various media formats and edge cases

---

## Test 1: Google Drive Scraper - Media Extraction

### Test Setup
**Vendor:** Test Vendor (Google Drive)  
**Source:** Google Drive folder with product images and videos  
**Expected Tier:** 1

### Test Data
Create a test Google Drive folder with:
- 10 products
- Each product has:
  - Product info file (CSV or JSON)
  - 3-5 product images (JPG, PNG, WebP)
  - 1-2 product videos (MP4, MOV)
  - Nested folder structure

### Test Cases

#### TC1.1: Basic Image Extraction
**Input:**
- Product folder with 3 images: `product1.jpg`, `product1_alt.png`, `product1_box.webp`

**Expected Output:**
```json
{
  "SKU": "TEST-001",
  "Name": "Test Product 1",
  "image_urls": [
    "https://drive.google.com/uc?id=FILE_ID_1&export=download",
    "https://drive.google.com/uc?id=FILE_ID_2&export=download",
    "https://drive.google.com/uc?id=FILE_ID_3&export=download"
  ],
  "video_urls": [],
  "vendor_id": "test_vendor_drive",
  "source_tier": 1
}
```

**Validation:**
- ✅ `image_urls` is a JSON array
- ✅ All 3 images extracted
- ✅ URLs are shareable Google Drive links
- ✅ `vendor_id` matches vendor_config entry
- ✅ `source_tier` = 1

#### TC1.2: Video Extraction
**Input:**
- Product folder with 2 videos: `demo.mp4`, `unboxing.mov`

**Expected Output:**
```json
{
  "SKU": "TEST-002",
  "Name": "Test Product 2",
  "image_urls": [],
  "video_urls": [
    "https://drive.google.com/uc?id=VIDEO_ID_1&export=download",
    "https://drive.google.com/uc?id=VIDEO_ID_2&export=download"
  ],
  "vendor_id": "test_vendor_drive",
  "source_tier": 1
}
```

**Validation:**
- ✅ `video_urls` is a JSON array
- ✅ Both videos extracted
- ✅ URLs are shareable Google Drive links
- ✅ Video formats recognized (MP4, MOV)

#### TC1.3: Mixed Media (Images + Videos)
**Input:**
- Product folder with 5 images and 2 videos

**Expected Output:**
```json
{
  "SKU": "TEST-003",
  "Name": "Test Product 3",
  "image_urls": ["url1", "url2", "url3", "url4", "url5"],
  "video_urls": ["video1", "video2"],
  "vendor_id": "test_vendor_drive",
  "source_tier": 1
}
```

**Validation:**
- ✅ Both arrays populated correctly
- ✅ Images and videos separated properly
- ✅ Order preserved (alphabetical or upload order)

#### TC1.4: Nested Folder Structure
**Input:**
```
/Vendor Products/
  /Category A/
    /Product 1/
      product1.jpg
      product1_alt.png
      demo.mp4
  /Category B/
    /Product 2/
      product2.jpg
```

**Expected Output:**
- Both products extracted with correct media URLs
- Folder structure doesn't break extraction
- `vendor_id` consistent across all products

#### TC1.5: No Media Files
**Input:**
- Product folder with only data file (no images/videos)

**Expected Output:**
```json
{
  "SKU": "TEST-005",
  "Name": "Test Product 5",
  "image_urls": [],
  "video_urls": [],
  "vendor_id": "test_vendor_drive",
  "source_tier": 1
}
```

**Validation:**
- ✅ Empty arrays (not null)
- ✅ Product still processed successfully
- ✅ Other fields populated correctly

#### TC1.6: Large Media Collection (50+ Images)
**Input:**
- Product folder with 50 images and 10 videos

**Expected Output:**
- All 50 images in `image_urls` array
- All 10 videos in `video_urls` array
- JSON array size within limits
- No truncation or data loss

**Validation:**
- ✅ Array length = 50 for images
- ✅ Array length = 10 for videos
- ✅ No performance degradation
- ✅ All URLs valid

#### TC1.7: Unsupported File Formats
**Input:**
- Product folder with: `product.jpg`, `document.pdf`, `data.xlsx`, `video.mp4`

**Expected Output:**
```json
{
  "image_urls": ["product.jpg"],
  "video_urls": ["video.mp4"]
}
```

**Validation:**
- ✅ Only image/video formats extracted
- ✅ PDF and Excel files ignored
- ✅ No errors thrown

---

## Test 2: Dropbox Scraper - Media Extraction

### Test Setup
**Vendor:** Test Vendor (Dropbox)  
**Source:** Dropbox shared folder  
**Expected Tier:** 1

### Test Cases

#### TC2.1: Dropbox Shared Links
**Input:**
- Dropbox shared folder with 3 images

**Expected Output:**
```json
{
  "image_urls": [
    "https://www.dropbox.com/s/SHARE_ID_1/image1.jpg?dl=1",
    "https://www.dropbox.com/s/SHARE_ID_2/image2.jpg?dl=1",
    "https://www.dropbox.com/s/SHARE_ID_3/image3.jpg?dl=1"
  ],
  "vendor_id": "test_vendor_dropbox",
  "source_tier": 1
}
```

**Validation:**
- ✅ URLs have `?dl=1` parameter for direct download
- ✅ Shareable links generated correctly
- ✅ `vendor_id` and `source_tier` correct

#### TC2.2: Dropbox Video Links
**Input:**
- Dropbox folder with 2 videos

**Expected Output:**
```json
{
  "video_urls": [
    "https://www.dropbox.com/s/SHARE_ID/video1.mp4?dl=1",
    "https://www.dropbox.com/s/SHARE_ID/video2.mov?dl=1"
  ],
  "vendor_id": "test_vendor_dropbox",
  "source_tier": 1
}
```

**Validation:**
- ✅ Video URLs with `?dl=1` parameter
- ✅ Both MP4 and MOV formats supported

#### TC2.3: Dropbox Nested Folders
**Input:**
- Nested folder structure similar to Google Drive test

**Expected Output:**
- All products extracted with correct media
- Folder hierarchy traversed correctly
- Links generated for all files

---

## Test 3: Google Sheets Scraper - Media Extraction

### Test Setup
**Vendor:** Test Vendor (Google Sheets)  
**Source:** Google Sheet with product data and media URLs  
**Expected Tier:** 1

### Test Cases

#### TC3.1: Image URLs in Columns
**Input:**
Google Sheet with columns:
```
SKU | Name | Image1 | Image2 | Image3
TEST-001 | Product 1 | url1 | url2 | url3
```

**Expected Output:**
```json
{
  "SKU": "TEST-001",
  "Name": "Product 1",
  "image_urls": ["url1", "url2", "url3"],
  "vendor_id": "test_vendor_sheets",
  "source_tier": 1
}
```

**Validation:**
- ✅ Multiple image columns detected
- ✅ URLs combined into single array
- ✅ Empty cells handled (not added to array)

#### TC3.2: Video URLs in Columns
**Input:**
```
SKU | Name | Video1 | Video2
TEST-002 | Product 2 | video_url1 | video_url2
```

**Expected Output:**
```json
{
  "video_urls": ["video_url1", "video_url2"],
  "vendor_id": "test_vendor_sheets",
  "source_tier": 1
}
```

#### TC3.3: Comma-Separated URLs in Single Cell
**Input:**
```
SKU | Name | Images
TEST-003 | Product 3 | url1, url2, url3
```

**Expected Output:**
```json
{
  "image_urls": ["url1", "url2", "url3"]
}
```

**Validation:**
- ✅ Comma-separated values parsed correctly
- ✅ Whitespace trimmed
- ✅ Array format maintained

#### TC3.4: Multi-Sheet Structure
**Input:**
- Sheet 1: Product data
- Sheet 2: Image URLs (SKU mapping)
- Sheet 3: Video URLs (SKU mapping)

**Expected Output:**
- Data from all sheets combined correctly
- SKU used as join key
- All media URLs associated with correct products

---

## Test 4: Data Upserter Validation

### Test Setup
Verify that `data_upserter.json` correctly processes Tier 1 data with new fields.

### Test Cases

#### TC4.1: Batch Processing with Media
**Input:**
- 50 products from Tier 1 scrapers
- Each with image_urls and video_urls

**Expected Output:**
- All 50 products upserted successfully
- Batch size = 50 (as configured)
- All media URLs preserved
- `vendor_id` and `source_tier` set correctly

**Validation:**
- ✅ Batch processing works with JSON fields
- ✅ No data loss during upsert
- ✅ `updatedAt` timestamp updated

#### TC4.2: Upsert Logic (Update Existing)
**Input:**
- Product TEST-001 already exists with 3 images
- New scrape finds 5 images

**Expected Output:**
```json
{
  "SKU": "TEST-001",
  "image_urls": ["new1", "new2", "new3", "new4", "new5"],
  "updatedAt": "2026-03-19T17:45:00.000Z"
}
```

**Validation:**
- ✅ Old image URLs replaced (not appended)
- ✅ `updatedAt` timestamp updated
- ✅ Other fields preserved

---

## Test 5: Edge Cases

### TC5.1: Malformed URLs
**Input:**
- Image URL: `not-a-valid-url`
- Video URL: `http://broken-link.com/video.mp4`

**Expected Behavior:**
- Invalid URLs logged as warnings
- Product still processed
- Invalid URLs excluded from arrays OR included with validation flag

### TC5.2: Duplicate URLs
**Input:**
- Same image URL appears 3 times in folder

**Expected Output:**
- Deduplicated array (only 1 instance)
- OR all instances preserved (depending on business logic)

### TC5.3: Very Long URLs
**Input:**
- Image URL with 500+ characters

**Expected Behavior:**
- URL stored successfully (JSON supports long strings)
- No truncation

### TC5.4: Special Characters in Filenames
**Input:**
- `product #1 (special).jpg`
- `video & demo.mp4`

**Expected Behavior:**
- URLs properly encoded
- Special characters handled correctly

---

## 📊 Success Criteria

### Must Pass (Blockers)
- ✅ All 3 Tier 1 scrapers extract media URLs
- ✅ `image_urls` and `video_urls` are valid JSON arrays
- ✅ `vendor_id` correctly assigned
- ✅ `source_tier = 1` for all Tier 1 products
- ✅ Data upserter processes new fields correctly

### Should Pass (High Priority)
- ✅ Large media collections (50+ images) handled
- ✅ Nested folder structures work
- ✅ Empty media arrays handled gracefully
- ✅ Multiple file formats supported

### Nice to Have (Medium Priority)
- ✅ Malformed URLs handled gracefully
- ✅ Duplicate URLs deduplicated
- ✅ Special characters in filenames work

---

## 🧪 Test Execution

### Manual Testing Steps
1. Create test Google Drive folder with sample products
2. Create test Dropbox folder with sample products
3. Create test Google Sheet with sample data
4. Configure vendor_config entries for test vendors
5. Run each Tier 1 scraper workflow
6. Query product_data table for test products
7. Validate `image_urls`, `video_urls`, `vendor_id`, `source_tier`
8. Document results in TEST_RESULTS.md

### Automated Testing (n8n Workflow)
1. Import `test_tier1_media_extraction.json` workflow
2. Configure test data sources
3. Run workflow
4. Review execution logs
5. Validate output against expected results

---

## 📝 Test Results Template

```markdown
### Test Execution: Tier 1 Media Extraction
**Date:** YYYY-MM-DD  
**Tester:** Name  
**Environment:** n8n Cloud

#### Google Drive Scraper
- TC1.1: ✅ PASS / ❌ FAIL - Notes
- TC1.2: ✅ PASS / ❌ FAIL - Notes
- TC1.3: ✅ PASS / ❌ FAIL - Notes
- TC1.4: ✅ PASS / ❌ FAIL - Notes
- TC1.5: ✅ PASS / ❌ FAIL - Notes
- TC1.6: ✅ PASS / ❌ FAIL - Notes
- TC1.7: ✅ PASS / ❌ FAIL - Notes

#### Dropbox Scraper
- TC2.1: ✅ PASS / ❌ FAIL - Notes
- TC2.2: ✅ PASS / ❌ FAIL - Notes
- TC2.3: ✅ PASS / ❌ FAIL - Notes

#### Google Sheets Scraper
- TC3.1: ✅ PASS / ❌ FAIL - Notes
- TC3.2: ✅ PASS / ❌ FAIL - Notes
- TC3.3: ✅ PASS / ❌ FAIL - Notes
- TC3.4: ✅ PASS / ❌ FAIL - Notes

#### Data Upserter
- TC4.1: ✅ PASS / ❌ FAIL - Notes
- TC4.2: ✅ PASS / ❌ FAIL - Notes

#### Edge Cases
- TC5.1: ✅ PASS / ❌ FAIL - Notes
- TC5.2: ✅ PASS / ❌ FAIL - Notes
- TC5.3: ✅ PASS / ❌ FAIL - Notes
- TC5.4: ✅ PASS / ❌ FAIL - Notes

**Overall Result:** ✅ PASS / ❌ FAIL  
**Pass Rate:** X/Y tests passed (Z%)  
**Issues Found:** List any bugs or issues  
**Recommendations:** Suggestions for improvements
```

---

**Next Steps:**
1. Execute all test cases
2. Document results
3. Fix any issues found
4. Proceed to Tier 2 and Tier 3 tests

