# Phase 7: Testing & Validation Plan
## Vendor Product Data Collection System

**Version:** 1.0  
**Date:** 2026-03-19  
**Status:** Ready for Execution (Pending Phase 6 Completion)

---

## 🎯 Testing Objectives

### Primary Goals
1. **Validate Hybrid Schema** - Ensure all 35 fields work correctly (31 existing + 4 new from Phase 6)
2. **Verify Media Asset Collection** - Confirm image_urls and video_urls are properly extracted
3. **Test Both Workflows** - Flow 1 (Weekly Sync) and Flow 2 (Order Fulfillment)
4. **Validate Performance** - Meet <10s cached, <60s on-demand targets
5. **Confirm Cost Targets** - Stay under $300/year budget
6. **Ensure Reliability** - >95% success rate, proper error handling

### Success Criteria
- ✅ Schema Validation: 100% of 35 fields working
- ✅ Test Coverage: >90%
- ✅ Pass Rate: >95%
- ✅ Sync Success Rate: >95%
- ✅ Fulfillment: <10s (cached), <60s (on-demand)
- ✅ Cache Hit Rate: >90%
- ✅ Hybrid Schema: All 4 new fields working
- ✅ Media Assets: Images/videos collected and delivered
- ✅ Cost: <$300/year

---

## 📋 Test Categories

### 1. Schema Validation Tests
**Location:** `tests/schema_tests/`  
**Purpose:** Validate the hybrid schema structure and data integrity

#### Test Cases:
1. **Schema Structure Validation**
   - Verify all 35 fields exist in product_data table
   - Confirm correct data types for each field
   - Test NOT NULL constraints (SKU, Name)
   - Test unique constraints (SKU)
   - Validate auto-increment fields (id, createdAt, updatedAt)

2. **New Fields Validation (Phase 6)**
   - `image_urls` (JSON type, nullable, array format)
   - `video_urls` (JSON type, nullable, array format)
   - `vendor_id` (Text type, nullable, foreign key reference)
   - `source_tier` (Number type, nullable, values 1-3)

3. **Existing Fields Integrity**
   - Verify all 31 Phase 5 fields still work
   - Test board game specific fields (BGG_weight, Mechanics_list, etc.)
   - Test pricing fields (MSRP, MAP, Cost)
   - Test dimension fields (Weight_lbs, Length_in, etc.)

4. **Data Upserter Validation**
   - Test Phase 5 data_upserter populates new fields
   - Verify batch processing (50 products at a time)
   - Test upsert logic (insert new, update existing)
   - Validate timestamp updates (updatedAt)

**Expected Results:**
- All 35 fields accessible and functional
- New fields properly populated by data_upserter
- No data loss from existing 31 fields
- Proper JSON array format for image_urls/video_urls

---

### 2. Enhanced Unit Tests
**Location:** `tests/tier_tests/`  
**Purpose:** Test each tier scraper with media extraction capabilities

#### Tier 1 Tests (Direct Integrations)
**File:** `test_tier1_media_extraction.json`

1. **Google Drive Scraper**
   - Test folder traversal with image/video detection
   - Verify shareable link generation for media files
   - Test image formats: JPG, PNG, GIF, WebP
   - Test video formats: MP4, MOV, AVI
   - Validate vendor_id assignment
   - Confirm source_tier = 1

2. **Dropbox Scraper**
   - Test shared folder media extraction
   - Verify `dl=1` parameter for direct links
   - Test nested folder structures
   - Validate media URL arrays
   - Confirm vendor_id and source_tier

3. **Google Sheets Scraper**
   - Test column detection for image/video URLs
   - Validate URL parsing and cleaning
   - Test multi-sheet structures
   - Verify array formatting for multiple URLs
   - Confirm metadata fields

**Test Data:**
- 10 sample products per tier
- Mix of single and multiple images/videos
- Various file formats and sizes
- Edge cases: no media, 50+ images, broken URLs

#### Tier 2 Tests (Shopify)
**File:** `test_tier2_media_extraction.json`

1. **Shopify JSON Endpoint**
   - Test `/products.json` parsing
   - Extract images from `product.images` array
   - Extract videos from `product.media` array
   - Test pagination (250 products per page)
   - Validate vendor_id from Shopify domain
   - Confirm source_tier = 2

2. **Shopify Fallback to Tier 3**
   - Test fallback trigger conditions
   - Verify Zyte API activation
   - Confirm source_tier updates to 3
   - Test cost tracking for fallback

**Test Data:**
- 5 Shopify stores with varying product counts
- Products with 0, 1, 5, 20+ images
- Products with videos
- Rate limiting scenarios

#### Tier 3 Tests (Zyte + AI)
**File:** `test_tier3_media_extraction.json`

1. **Zyte API Scraping**
   - Test HTML extraction
   - Verify JavaScript rendering
   - Test rate limiting (5 requests/second)
   - Validate response caching

2. **Gemini AI Extraction**
   - Test image URL extraction from HTML
   - Test video URL extraction
   - Validate JSON response parsing
   - Test prompt engineering for accuracy
   - Confirm vendor_id and source_tier = 3

3. **Cost Tracking**
   - Verify Zyte cost calculation ($0.00013/request)
   - Verify Gemini cost calculation ($0.002/request)
   - Test cost logging in sync_log table

**Test Data:**
- 10 custom websites with varying structures
- Sites with lazy-loaded images
- Sites with embedded videos
- Sites with image galleries

**Expected Results:**
- All tier scrapers extract media URLs correctly
- vendor_id properly assigned
- source_tier accurately tracked (1, 2, or 3)
- Media URLs in proper JSON array format
- Cost tracking accurate for Tier 3

---

### 3. Integration Tests
**Location:** `tests/integration_tests/`  
**Purpose:** Test complete workflows end-to-end

#### Flow 1: Weekly Sync Tests
**File:** `test_flow1_weekly_sync.json`

1. **Full Sync Test**
   - Trigger weekly sync workflow
   - Test MarketTime CSV import from Google Drive
   - Loop through all 70 vendors
   - Route to appropriate tier (1, 2, or 3)
   - Verify data_upserter batch processing
   - Check sync_log entries
   - Validate all 35 fields populated
   - Confirm media URLs extracted

2. **Partial Failure Test**
   - Simulate 5 vendor failures
   - Verify error logging
   - Test Slack/Email notifications
   - Confirm successful vendors still processed
   - Validate sync_log status = 'partial'

3. **Tier Fallback Test**
   - Simulate Shopify (Tier 2) failure
   - Trigger fallback to Tier 3
   - Verify Zyte API activation
   - Confirm source_tier updates
   - Test cost tracking for fallback

4. **Performance Test**
   - Measure total sync duration
   - Target: <2 hours for 70 vendors
   - Test parallel processing
   - Validate batch upserts (50 products)

**Expected Results:**
- >95% sync success rate
- <2 hours total duration
- Proper error handling and logging
- Tier fallback works correctly
- All media URLs extracted

#### Flow 2: Order Fulfillment Tests
**File:** `test_flow2_order_fulfillment.json`

1. **Cached SKU Test (<10s)**
   - Trigger order with 5 cached SKUs
   - Query product_data table
   - Compile asset package
   - Deliver to customer
   - Measure total time
   - Target: <10 seconds

2. **On-Demand Scraping Test (<60s)**
   - Trigger order with 2 uncached SKUs
   - Activate on-demand scraper
   - Scrape vendor site (Tier 3)
   - Extract media URLs
   - Compile and deliver
   - Measure total time
   - Target: <60 seconds

3. **MarketTime Email Trigger**
   - Send test order confirmation email
   - Verify IMAP monitor detection
   - Parse order details (customer, SKUs, quantities)
   - Test SKU lookup
   - Validate asset compilation
   - Test email delivery

4. **Shopify Webhook Trigger**
   - Send test webhook payload
   - Verify HMAC signature validation
   - Parse order JSON
   - Test SKU lookup
   - Validate asset compilation
   - Test delivery methods

5. **Mixed Order Test**
   - Order with 3 cached + 2 uncached SKUs
   - Verify parallel processing
   - Test asset compiler with mixed sources
   - Validate complete package delivery

6. **Delivery Methods Test**
   - Test Email delivery (with attachments)
   - Test Google Drive delivery (shared folder)
   - Test Dropbox delivery (shared link)
   - Test Download Link delivery (temporary URL)

**Expected Results:**
- <10s for cached SKUs (>90% of orders)
- <60s for on-demand scraping
- Both triggers work correctly
- All delivery methods functional
- Complete asset packages (images, videos, copy)

---

### 4. Edge Case Tests
**Location:** `tests/edge_cases/`  
**Purpose:** Test unusual scenarios and error conditions

#### Test Cases:
**File:** `test_edge_cases.json`

1. **Empty Media Arrays**
   - Product with no images or videos
   - Verify empty JSON arrays: `[]`
   - Test asset compiler handles gracefully
   - Validate customer notification

2. **Large Media Arrays**
   - Product with 50+ images
   - Product with 10+ videos
   - Test JSON array size limits
   - Verify batch processing
   - Test delivery package size

3. **Invalid Media URLs**
   - Broken image links (404)
   - Malformed URLs
   - Non-image/video URLs
   - Test URL validation
   - Verify error handling

4. **Malformed Data**
   - Wrong data types (string in number field)
   - Missing required fields (SKU, Name)
   - Invalid JSON in image_urls/video_urls
   - Test data validation
   - Verify error logging

5. **Duplicate SKUs**
   - Same SKU from multiple vendors
   - Test vendor_id differentiation
   - Verify upsert logic (last write wins)
   - Test source_tier tracking

6. **Missing Vendor Config**
   - Order for vendor not in vendor_config
   - Test error handling
   - Verify admin notification
   - Test fallback behavior

7. **Rate Limiting**
   - Shopify API rate limits (2 requests/second)
   - Gemini API rate limits (60 requests/minute)
   - Zyte API rate limits (5 requests/second)
   - Test retry logic
   - Verify exponential backoff

8. **Large Orders**
   - Order with 50+ SKUs
   - Test batch processing
   - Verify timeout handling
   - Test partial delivery

9. **Concurrent Orders**
   - Multiple orders at same time
   - Test n8n workflow concurrency
   - Verify data integrity
   - Test resource limits

10. **Database Errors**
    - Connection failures
    - Query timeouts
    - Table locks
    - Test retry logic
    - Verify error recovery

**Expected Results:**
- Graceful handling of all edge cases
- Proper error logging
- Admin notifications for critical failures
- No data corruption
- System remains operational

---

### 5. Performance Tests
**Location:** `tests/performance_tests/`  
**Purpose:** Validate system performance meets targets

#### Test Cases:
**File:** `test_performance.json`

1. **Sync Performance**
   - Measure sync time for 70 vendors
   - Target: <2 hours total
   - Test parallel processing
   - Measure per-vendor time
   - Identify bottlenecks

2. **Fulfillment Performance**
   - Cached SKU lookup: Target <10s
   - On-demand scraping: Target <60s
   - Measure database query time
   - Measure scraping time
   - Measure asset compilation time

3. **Database Performance**
   - Query performance for SKU lookup
   - Batch upsert performance (50 products)
   - JSON array query performance
   - Index effectiveness
   - Connection pool efficiency

4. **Media Extraction Timing**
   - Tier 1 media extraction time
   - Tier 2 media extraction time
   - Tier 3 media extraction time
   - Compare extraction methods
   - Identify optimization opportunities

5. **API Response Times**
   - Zyte API response time
   - Gemini API response time
   - Shopify API response time
   - Google Drive API response time
   - Dropbox API response time

**Performance Targets:**
- Weekly sync: <2 hours (70 vendors)
- Cached fulfillment: <10 seconds
- On-demand fulfillment: <60 seconds
- SKU lookup: <1 second
- Batch upsert: <5 seconds (50 products)

**Expected Results:**
- All performance targets met
- No significant bottlenecks
- Scalable to 100+ vendors
- Efficient resource usage

---

### 6. Cost Validation Tests
**Location:** `tests/cost_tests/`  
**Purpose:** Verify system stays within $300/year budget

#### Test Cases:
**File:** `test_cost_validation.json`

1. **Weekly Sync Cost**
   - Calculate Tier 3 vendor costs
   - Zyte: 2,000 pages × $0.00013 = $0.26/week
   - Gemini: 2,000 pages × $0.002 = $4.00/week
   - Weekly total: ~$4.26
   - Annual projection: ~$221

2. **Order Fulfillment Cost**
   - Calculate on-demand scraping costs
   - Assume 10% orders need scraping
   - 100 orders/month × 10% × 2 products = 20 scrapes
   - Zyte: 20 × $0.00013 = $0.0026/month
   - Gemini: 20 × $0.002 = $0.04/month
   - Monthly total: ~$0.04
   - Annual projection: ~$0.50

3. **Annual Cost Projection**
   - Weekly sync: $221/year
   - Order fulfillment: $0.50/year
   - Total: ~$222/year
   - Buffer: $78 remaining
   - Target: <$300/year ✅

4. **Cost Tracking Validation**
   - Verify sync_log cost_estimate field
   - Test cost calculation accuracy
   - Validate cost aggregation
   - Test cost alerts (>$25/week)

5. **Cost Optimization Tests**
   - Test cache hit rate (target >90%)
   - Measure Tier 2 → Tier 3 fallback frequency
   - Identify high-cost vendors
   - Test cost reduction strategies

**Expected Results:**
- Total annual cost <$300
- Cost tracking accurate
- Cache hit rate >90%
- Cost alerts functional
- Optimization opportunities identified

---

## 🧪 Test Execution Plan

### Phase 1: Schema Validation (2 hours)
1. Create test data (35 fields, various data types)
2. Run schema structure tests
3. Validate new fields (image_urls, video_urls, vendor_id, source_tier)
4. Test data_upserter with new fields
5. Document results

### Phase 2: Unit Tests (8 hours)
1. Test Tier 1 scrapers (Google Drive, Dropbox, Sheets)
2. Test Tier 2 scrapers (Shopify)
3. Test Tier 3 scrapers (Zyte + Gemini)
4. Validate media extraction for all tiers
5. Test vendor_id and source_tier tracking
6. Document results

### Phase 3: Integration Tests (6 hours)
1. Test Flow 1 (Weekly Sync)
   - Full sync
   - Partial failure
   - Tier fallback
   - Performance
2. Test Flow 2 (Order Fulfillment)
   - Cached SKUs
   - On-demand scraping
   - MarketTime trigger
   - Shopify webhook
   - Delivery methods
3. Document results

### Phase 4: Edge Cases (4 hours)
1. Test empty/large media arrays
2. Test invalid URLs and malformed data
3. Test duplicate SKUs and missing configs
4. Test rate limiting and concurrent orders
5. Test database errors
6. Document results

### Phase 5: Performance Tests (3 hours)
1. Measure sync performance
2. Measure fulfillment performance
3. Measure database performance
4. Measure API response times
5. Document results

### Phase 6: Cost Validation (2 hours)
1. Calculate weekly sync costs
2. Calculate order fulfillment costs
3. Project annual costs
4. Validate cost tracking
5. Document results

### Phase 7: Documentation (3 hours)
1. Compile TEST_RESULTS.md
2. Create PERFORMANCE_REPORT.md
3. Update IMPLEMENTATION_PLAN.md
4. Create test summary dashboard
5. Final review

**Total Estimated Time:** 28 hours

---

## 📊 Test Metrics

### Coverage Metrics
- **Unit Test Coverage:** >90%
- **Integration Test Coverage:** 100% of workflows
- **Edge Case Coverage:** 20+ scenarios
- **Performance Test Coverage:** All critical paths

### Quality Metrics
- **Pass Rate:** >95%
- **Sync Success Rate:** >95%
- **Cache Hit Rate:** >90%
- **Error Recovery Rate:** 100%

### Performance Metrics
- **Sync Duration:** <2 hours (70 vendors)
- **Cached Fulfillment:** <10 seconds
- **On-demand Fulfillment:** <60 seconds
- **Database Query Time:** <1 second

### Cost Metrics
- **Weekly Sync Cost:** ~$4.26
- **Monthly Fulfillment Cost:** ~$0.04
- **Annual Total Cost:** ~$222
- **Budget Compliance:** <$300/year ✅

---

## 🚨 Critical Test Scenarios

### Must-Pass Tests (Blockers)
1. ✅ All 35 fields accessible and functional
2. ✅ Media URLs extracted for all tiers
3. ✅ vendor_id and source_tier tracked correctly
4. ✅ Flow 1 sync completes successfully
5. ✅ Flow 2 fulfillment <10s for cached SKUs
6. ✅ Annual cost <$300

### High-Priority Tests
1. ✅ Tier fallback works (Tier 2 → Tier 3)
2. ✅ On-demand scraping <60s
3. ✅ Error handling and notifications
4. ✅ All delivery methods functional
5. ✅ Cache hit rate >90%

### Medium-Priority Tests
1. Edge cases handled gracefully
2. Rate limiting respected
3. Concurrent orders processed
4. Large orders completed
5. Performance optimizations identified

---

## 📝 Test Documentation

### Required Documents
1. **TEST_PLAN.md** (this document)
   - Test strategy and approach
   - Test cases and scenarios
   - Execution plan and timeline

2. **TEST_RESULTS.md**
   - Test execution results
   - Pass/fail status for each test
   - Issues and bugs found
   - Recommendations

3. **PERFORMANCE_REPORT.md**
   - Performance metrics
   - Bottleneck analysis
   - Optimization recommendations
   - Scalability assessment

### Test Artifacts
- Test workflows (JSON files)
- Test data (sample products, orders)
- Test logs (execution logs, error logs)
- Screenshots (n8n workflow executions)
- Performance graphs (timing, costs)

---

## ✅ Definition of Done

Phase 7 is complete when:
- [ ] All test categories executed (6 categories)
- [ ] Test coverage >90%
- [ ] Pass rate >95%
- [ ] All critical tests pass
- [ ] Performance targets met
- [ ] Cost targets met (<$300/year)
- [ ] All documentation complete
- [ ] Test results reviewed and approved
- [ ] Issues logged and prioritized
- [ ] System ready for Phase 8 (deployment)

---

## 🔄 Next Steps

After Phase 7 completion:
1. Review test results with stakeholders
2. Address any critical issues found
3. Optimize performance bottlenecks
4. Update documentation based on findings
5. Proceed to Phase 8: Documentation & Deployment

---

**Document Status:** ✅ Ready for Execution  
**Last Updated:** 2026-03-19  
**Next Review:** After Phase 6 completion

