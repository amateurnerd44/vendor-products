# Flow 1: Weekly Sync Integration Tests
## End-to-End Testing with Hybrid Schema

**Test Suite:** Phase 7 - Flow 1 Integration Tests  
**Date:** 2026-03-19  
**Status:** Ready for Execution

---

## 🎯 Test Objectives

Validate the complete weekly sync workflow (Flow 1) with Phase 6 hybrid schema:
1. MarketTime CSV import from Google Drive
2. Vendor routing to appropriate tiers (1, 2, 3)
3. Media extraction (images/videos) for all tiers
4. Data upserter populates all 35 fields
5. Sync logging with cost tracking
6. Error handling and notifications
7. Tier fallback (Tier 2 → Tier 3)

---

## Test 1: Full Sync - All 70 Vendors

### Test Setup
**Trigger:** Manual (simulating Sunday 2:00 AM cron)  
**Vendors:** All 70 vendors (30 Tier 1, 20 Tier 2, 20 Tier 3)  
**Expected Duration:** <2 hours

### Test Steps
1. Trigger `weekly_sync.json` workflow
2. Monitor execution progress
3. Check sync_log table for results
4. Validate product_data table updates
5. Verify all 35 fields populated
6. Check media URLs extracted

### Expected Results

#### Sync Log Entry
```json
{
  "sync_date": "2026-03-19T02:00:00.000Z",
  "status": "success",
  "vendors_processed": 70,
  "vendors_success": 67,
  "vendors_failed": 3,
  "products_updated": 5234,
  "products_added": 432,
  "products_skipped": 12,
  "duration_seconds": 6840,
  "cost_estimate": 4.26
}
```

#### Product Data Validation
Query random sample of 100 products:
```sql
SELECT 
  SKU, Name, vendor_id, source_tier,
  image_urls, video_urls,
  updatedAt
FROM product_data
WHERE updatedAt >= '2026-03-19T02:00:00.000Z'
LIMIT 100;
```

**Validation Checks:**
- ✅ All 100 products have `vendor_id` populated
- ✅ All 100 products have `source_tier` (1, 2, or 3)
- ✅ >90% have `image_urls` with at least 1 image
- ✅ >50% have `video_urls` with at least 1 video
- ✅ `updatedAt` timestamp is recent (within last 2 hours)
- ✅ All 35 fields accessible

### Success Criteria
- ✅ Sync completes in <2 hours
- ✅ Success rate >95% (67/70 vendors)
- ✅ All products have vendor_id and source_tier
- ✅ Media URLs extracted for >90% of products
- ✅ Cost estimate ~$4.26 for Tier 3 vendors
- ✅ Sync log entry created

---

## Test 2: Partial Failure Handling

### Test Setup
**Trigger:** Manual  
**Scenario:** Simulate 5 vendor failures (API errors, timeouts, invalid configs)

### Test Steps
1. Modify 5 vendor configs to cause failures:
   - Invalid Google Drive folder ID
   - Broken Shopify URL
   - Zyte API timeout
   - Missing credentials
   - Invalid data format
2. Trigger weekly sync
3. Monitor error handling
4. Check notifications sent

### Expected Results

#### Sync Log Entry
```json
{
  "status": "partial",
  "vendors_processed": 70,
  "vendors_success": 65,
  "vendors_failed": 5,
  "errors": [
    {
      "vendor_id": "vendor_001",
      "vendor_name": "Test Vendor 1",
      "error": "Invalid Google Drive folder ID",
      "tier": 1
    },
    {
      "vendor_id": "vendor_015",
      "vendor_name": "Test Vendor 15",
      "error": "Shopify API rate limit exceeded",
      "tier": 2
    },
    {
      "vendor_id": "vendor_045",
      "vendor_name": "Test Vendor 45",
      "error": "Zyte API timeout after 30s",
      "tier": 3
    }
  ]
}
```

#### Notifications
- ✅ Slack message sent with error summary
- ✅ Email sent to admin with detailed error log
- ✅ Failed vendors listed with error messages
- ✅ Successful vendors still processed

### Success Criteria
- ✅ Sync continues despite failures
- ✅ Successful vendors processed normally
- ✅ Errors logged in sync_log table
- ✅ Admin notifications sent
- ✅ Status = "partial" (not "failed")

---

## Test 3: Tier Fallback (Tier 2 → Tier 3)

### Test Setup
**Trigger:** Manual  
**Scenario:** Shopify vendor (Tier 2) fails, triggers fallback to Tier 3

### Test Steps
1. Configure vendor with `fallback_to_tier3 = true`
2. Simulate Shopify API failure (rate limit or 404)
3. Verify fallback to Zyte + Gemini
4. Check source_tier updates from 2 to 3
5. Validate cost tracking

### Expected Results

#### Vendor Config Before
```json
{
  "vendor_id": "shopify_vendor_001",
  "data_source_type": "shopify",
  "tier_used": 2,
  "fallback_to_tier3": true
}
```

#### Vendor Config After
```json
{
  "vendor_id": "shopify_vendor_001",
  "data_source_type": "shopify",
  "tier_used": 3,
  "fallback_to_tier3": true,
  "last_sync_date": "2026-03-19T02:15:00.000Z",
  "sync_status": "success"
}
```

#### Product Data
```json
{
  "SKU": "SHOP-001",
  "vendor_id": "shopify_vendor_001",
  "source_tier": 3,
  "image_urls": ["url1", "url2"],
  "video_urls": ["video1"]
}
```

#### Sync Log
```json
{
  "vendor_id": "shopify_vendor_001",
  "tier_used": 3,
  "cost_estimate": 0.26,
  "notes": "Fallback from Tier 2 to Tier 3 due to Shopify API failure"
}
```

### Success Criteria
- ✅ Fallback triggered automatically
- ✅ Zyte + Gemini scraping successful
- ✅ `source_tier` updated to 3
- ✅ Media URLs extracted via Tier 3
- ✅ Cost tracked correctly
- ✅ Vendor config updated

---

## Test 4: MarketTime CSV Import

### Test Setup
**Trigger:** Manual  
**Scenario:** Import new MarketTime items from Google Drive CSV

### Test Steps
1. Upload test CSV to Google Drive:
```csv
SKU,Vendor,Product Name,MSRP,Cost
NEW-001,Vendor A,New Product 1,29.99,15.00
NEW-002,Vendor B,New Product 2,39.99,20.00
EXISTING-001,Vendor C,Updated Product,49.99,25.00
```
2. Trigger weekly sync
3. Verify new items detected
4. Check sync_log for `new_markettime_items` count

### Expected Results

#### Sync Log
```json
{
  "new_markettime_items": 2,
  "products_added": 2,
  "products_updated": 1,
  "notes": "Found 2 new SKUs in MarketTime CSV"
}
```

#### Product Data
- NEW-001 and NEW-002 added to product_data
- EXISTING-001 updated with new pricing
- All products have vendor_id assigned

### Success Criteria
- ✅ CSV imported successfully
- ✅ New items detected and added
- ✅ Existing items updated
- ✅ Vendor routing works for new items
- ✅ Media extraction works for new items

---

## Test 5: Performance Test

### Test Setup
**Trigger:** Manual  
**Scenario:** Measure sync performance with 70 vendors

### Metrics to Collect

#### Overall Performance
- Total sync duration (target: <2 hours)
- Average time per vendor
- Slowest vendor (identify bottleneck)
- Fastest vendor

#### Tier Performance
- Tier 1 average time per vendor
- Tier 2 average time per vendor
- Tier 3 average time per vendor
- Tier 3 cost per vendor

#### Database Performance
- Batch upsert time (50 products)
- Query time for SKU lookup
- JSON array query performance
- Index effectiveness

#### API Performance
- Google Drive API response time
- Dropbox API response time
- Shopify API response time
- Zyte API response time
- Gemini API response time

### Expected Results

#### Performance Targets
```json
{
  "total_duration_seconds": 6840,
  "total_duration_hours": 1.9,
  "avg_time_per_vendor_seconds": 97.7,
  "tier1_avg_seconds": 45,
  "tier2_avg_seconds": 60,
  "tier3_avg_seconds": 180,
  "batch_upsert_seconds": 4.2,
  "sku_lookup_seconds": 0.8
}
```

### Success Criteria
- ✅ Total sync <2 hours
- ✅ Tier 1 vendors <60s each
- ✅ Tier 2 vendors <90s each
- ✅ Tier 3 vendors <300s each
- ✅ Batch upsert <5s
- ✅ SKU lookup <1s

---

## Test 6: Data Quality Validation

### Test Setup
**Trigger:** After sync completion  
**Scenario:** Validate data quality across all 35 fields

### Validation Queries

#### Check Required Fields
```sql
SELECT COUNT(*) as missing_sku
FROM product_data
WHERE SKU IS NULL OR SKU = '';

SELECT COUNT(*) as missing_name
FROM product_data
WHERE Name IS NULL OR Name = '';
```

**Expected:** Both counts = 0

#### Check Phase 6 Fields
```sql
SELECT 
  COUNT(*) as total_products,
  COUNT(vendor_id) as has_vendor_id,
  COUNT(source_tier) as has_source_tier,
  COUNT(image_urls) as has_images,
  COUNT(video_urls) as has_videos
FROM product_data
WHERE updatedAt >= '2026-03-19T02:00:00.000Z';
```

**Expected:**
- `has_vendor_id` = `total_products` (100%)
- `has_source_tier` = `total_products` (100%)
- `has_images` >= 90% of `total_products`
- `has_videos` >= 50% of `total_products`

#### Check Media URL Format
```sql
SELECT SKU, image_urls, video_urls
FROM product_data
WHERE 
  (image_urls IS NOT NULL AND image_urls NOT LIKE '[%]')
  OR
  (video_urls IS NOT NULL AND video_urls NOT LIKE '[%]')
LIMIT 10;
```

**Expected:** 0 results (all JSON arrays properly formatted)

### Success Criteria
- ✅ No missing required fields
- ✅ All products have vendor_id and source_tier
- ✅ >90% have image URLs
- ✅ >50% have video URLs
- ✅ All JSON arrays properly formatted

---

## Test 7: Error Recovery

### Test Setup
**Trigger:** Manual  
**Scenario:** Test error recovery mechanisms

### Test Cases

#### TC7.1: Database Connection Failure
**Scenario:** Simulate database timeout during upsert

**Expected Behavior:**
- Retry logic activates (3 attempts)
- Exponential backoff (1s, 2s, 4s)
- Error logged if all retries fail
- Admin notification sent

#### TC7.2: API Rate Limiting
**Scenario:** Hit Shopify rate limit (2 req/s)

**Expected Behavior:**
- Workflow pauses for rate limit window
- Resumes after cooldown
- No data loss
- Sync continues

#### TC7.3: Partial Data Loss
**Scenario:** Vendor provides incomplete data (missing fields)

**Expected Behavior:**
- Product still created/updated
- Missing fields set to null
- Warning logged
- Data quality report generated

### Success Criteria
- ✅ All error recovery mechanisms work
- ✅ No data corruption
- ✅ Sync completes despite errors
- ✅ Errors logged and reported

---

## 📊 Test Execution Checklist

### Pre-Test Setup
- [ ] Backup product_data table
- [ ] Configure test vendor configs
- [ ] Prepare test data (Drive, Dropbox, Sheets)
- [ ] Set up monitoring (n8n execution logs)
- [ ] Configure Slack/Email notifications

### Test Execution
- [ ] Test 1: Full Sync (70 vendors)
- [ ] Test 2: Partial Failure Handling
- [ ] Test 3: Tier Fallback
- [ ] Test 4: MarketTime CSV Import
- [ ] Test 5: Performance Test
- [ ] Test 6: Data Quality Validation
- [ ] Test 7: Error Recovery

### Post-Test Validation
- [ ] Review sync_log entries
- [ ] Validate product_data updates
- [ ] Check media URLs
- [ ] Verify cost estimates
- [ ] Review error logs
- [ ] Document results

---

## 📝 Test Results Template

```markdown
### Flow 1 Integration Test Results
**Date:** YYYY-MM-DD  
**Tester:** Name  
**Environment:** n8n Cloud Production

#### Test 1: Full Sync
- Duration: X hours Y minutes
- Vendors Processed: 70
- Success Rate: X%
- Products Updated: XXXX
- Products Added: XXX
- Cost Estimate: $X.XX
- Status: ✅ PASS / ❌ FAIL

#### Test 2: Partial Failure
- Failures Simulated: 5
- Failures Handled: 5
- Notifications Sent: ✅ Yes / ❌ No
- Status: ✅ PASS / ❌ FAIL

#### Test 3: Tier Fallback
- Fallback Triggered: ✅ Yes / ❌ No
- Tier 3 Successful: ✅ Yes / ❌ No
- Cost Tracked: ✅ Yes / ❌ No
- Status: ✅ PASS / ❌ FAIL

#### Test 4: MarketTime Import
- New Items Detected: X
- Items Added: X
- Items Updated: X
- Status: ✅ PASS / ❌ FAIL

#### Test 5: Performance
- Total Duration: X hours
- Avg Time/Vendor: X seconds
- Tier 1 Avg: X seconds
- Tier 2 Avg: X seconds
- Tier 3 Avg: X seconds
- Status: ✅ PASS / ❌ FAIL

#### Test 6: Data Quality
- Missing Required Fields: X
- Products with vendor_id: X%
- Products with images: X%
- Products with videos: X%
- Status: ✅ PASS / ❌ FAIL

#### Test 7: Error Recovery
- Recovery Mechanisms Tested: X
- Successful Recoveries: X
- Status: ✅ PASS / ❌ FAIL

**Overall Result:** ✅ PASS / ❌ FAIL  
**Pass Rate:** X/7 tests passed  
**Issues Found:** List any bugs  
**Recommendations:** Improvements needed
```

---

**Next Steps:**
1. Execute all integration tests
2. Document detailed results
3. Fix any critical issues
4. Proceed to Flow 2 tests

