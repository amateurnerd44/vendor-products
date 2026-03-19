# Phase 7 Agent Prompt - Testing & Validation

---

## 🎯 Your Mission

Complete **Phase 7: Testing & Validation**. Thoroughly test both workflows (Flow 1 and Flow 2) to ensure system reliability before production deployment.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

**Testing Scope:**
- All tier scrapers (1, 2, 3)
- Weekly sync workflow (Flow 1)
- Order fulfillment workflow (Flow 2)
- Error handling and edge cases
- Performance and cost validation

**Goal:** >95% sync success rate, <10s fulfillment time, <$300/year cost

---

## 🎯 Phase 7 Deliverables

### Task 1: Unit Tests for Each Tier

**Create test suite:** `tests/tier_tests/`

**Tier 1 Tests:**

1. **Google Drive Scraper:**
   - Test with folder structure Pattern A (`/products/{SKU}/images/`)
   - Test with folder structure Pattern B (flat with `{SKU}_image1.jpg`)
   - Test with missing folders (should skip gracefully)
   - Test with empty folders (should log warning)
   - Test with invalid file types (should filter)
   - Verify shareable links work

2. **Dropbox Scraper:**
   - Test with shared folder
   - Test with nested folder structure
   - Test download link generation (`dl=1` parameter)
   - Test rate limiting (600 req/hour)
   - Verify all files accessible

3. **Google Sheets Scraper:**
   - Test single-sheet structure
   - Test multi-sheet structure (Products, Images, Videos)
   - Test column auto-detection
   - Test with missing columns (should use defaults)
   - Test with empty rows (should skip)
   - Test with malformed URLs (should validate)

**Tier 2 Tests:**

1. **Shopify Scraper:**
   - Test with small store (<50 products)
   - Test with large store (>250 products, pagination)
   - Test with products that have variants
   - Test with products that have videos in description
   - Test rate limiting (2 req/sec)
   - Test with custom domain Shopify store
   - Verify all image URLs accessible

**Tier 3 Tests:**

1. **Zyte + Gemini Scraper:**
   - Test with simple product page
   - Test with complex product page (JavaScript-heavy)
   - Test with product catalog page
   - Test AI extraction accuracy
   - Test cost tracking
   - Test retry logic on failures
   - Verify extracted data quality

**Test Data:**
- Create 5-10 sample vendors per tier
- Use real public websites where possible
- Document test vendor URLs and expected results

### Task 2: Integration Tests

**Create test suite:** `tests/integration_tests/`

**Flow 1 Integration Test:**

1. **Full Sync Test:**
   - Set up 10 test vendors (3 Tier 1, 3 Tier 2, 4 Tier 3)
   - Run weekly sync workflow
   - Verify all vendors processed
   - Check sync_log entries created
   - Verify product_data table populated
   - Check vendor.last_sync_date updated
   - Measure total sync duration

2. **Partial Failure Test:**
   - Intentionally break 2 vendors (invalid credentials)
   - Run sync
   - Verify other 8 vendors succeed
   - Check error notifications sent
   - Verify failed vendors marked in sync_log

3. **Tier Fallback Test:**
   - Configure vendor with Tier 2 primary, Tier 3 fallback
   - Disable Tier 2 endpoint
   - Run sync
   - Verify fallback to Tier 3 works
   - Check tier_used updated in vendor_config

**Flow 2 Integration Test:**

1. **Order Fulfillment Test (Cached SKUs):**
   - Create test order with 3 SKUs (all in product_data table)
   - Trigger fulfillment workflow
   - Verify SKU lookup succeeds (100% cache hit)
   - Verify asset package compiled
   - Verify delivery email sent
   - Measure fulfillment time (should be <10s)

2. **Order Fulfillment Test (On-Demand Scraping):**
   - Create test order with 2 cached SKUs + 1 missing SKU
   - Trigger fulfillment workflow
   - Verify on-demand scraping triggered for missing SKU
   - Verify scraped data saved to product_data table
   - Verify all 3 products delivered
   - Measure fulfillment time (should be <60s)

3. **Order Fulfillment Test (Complete Failure):**
   - Create test order with invalid SKU (vendor not found)
   - Trigger fulfillment workflow
   - Verify failure handler triggered
   - Verify admin alert sent
   - Verify customer notified of delay

### Task 3: Edge Case Testing

**Create test suite:** `tests/edge_cases/`

**Edge Cases to Test:**

1. **Missing SKUs:**
   - Order with SKU not in any vendor catalog
   - Verify graceful failure and admin alert

2. **Failed Scrapes:**
   - Vendor website down (Tier 3)
   - Shopify endpoint disabled (Tier 2)
   - Drive folder deleted (Tier 1)
   - Verify retry logic and error logging

3. **Invalid Vendor Configs:**
   - Missing source_url
   - Invalid credentials
   - Wrong data_source_type
   - Verify validation and error messages

4. **Rate Limiting Scenarios:**
   - Shopify rate limit hit (429 error)
   - Gemini quota exceeded
   - Verify backoff and retry logic

5. **Malformed Data:**
   - Products with no images
   - Products with no SKU
   - Products with invalid URLs
   - Verify data validation and cleaning

6. **Large Orders:**
   - Order with 50+ SKUs
   - Verify batch processing works
   - Verify delivery doesn't timeout

7. **Duplicate SKUs:**
   - Same SKU in multiple vendor catalogs
   - Verify conflict resolution (use most recent)

8. **Empty Catalogs:**
   - Vendor with no products
   - Verify sync completes without errors

### Task 4: Performance Testing

**Create test suite:** `tests/performance_tests/`

**Performance Metrics to Measure:**

1. **Sync Performance:**
   - Measure sync time for 10, 25, 50, 70 vendors
   - Target: <2 hours for 70 vendors
   - Identify bottlenecks (slow scrapers)
   - Optimize if needed

2. **Fulfillment Performance:**
   - Measure fulfillment time for 1, 5, 10, 20 SKUs
   - Target: <10s for cached, <60s with on-demand
   - Test with different delivery methods
   - Optimize slow steps

3. **Database Performance:**
   - Measure query time for SKU lookup (1, 10, 100 SKUs)
   - Measure upsert time for 100, 500, 1000 products
   - Verify indexes are effective
   - Optimize if queries >1s

4. **API Performance:**
   - Measure Zyte API response time
   - Measure Gemini API response time
   - Measure Shopify endpoint response time
   - Track rate limit compliance

**Performance Optimization:**
- If sync >2 hours: Parallelize vendor processing
- If fulfillment >10s: Optimize SKU lookup query
- If database slow: Add indexes, optimize queries

### Task 5: Cost Validation

**Create test suite:** `tests/cost_tests/`

**Cost Tracking:**

1. **Weekly Sync Cost:**
   - Run full sync with all 70 vendors
   - Track Tier 3 usage:
     - Zyte requests: count × $0.00013
     - Gemini requests: count × $0.002
   - Calculate weekly cost
   - Verify <$5/week ($260/year)

2. **Order Fulfillment Cost:**
   - Simulate 100 orders/month
   - Assume 10% need on-demand scraping
   - Track Tier 3 usage for on-demand scrapes
   - Calculate monthly cost
   - Verify <$1/month ($12/year)

3. **Total Annual Cost:**
   - Weekly sync: ~$221/year
   - Order fulfillment: ~$12/year
   - Total: ~$233/year
   - Verify <$300/year budget

**Cost Optimization:**
- If over budget: Reduce Tier 3 vendors or scraping frequency
- Cache aggressively to reduce on-demand scraping
- Use Gemini Flash (cheaper) instead of Pro

---

## 📁 Deliverable Files

```
vendor-products/
├── tests/
│   ├── tier_tests/
│   │   ├── test_tier1_drive.json
│   │   ├── test_tier1_dropbox.json
│   │   ├── test_tier1_sheets.json
│   │   ├── test_tier2_shopify.json
│   │   └── test_tier3_zyte_ai.json
│   ├── integration_tests/
│   │   ├── test_flow1_full_sync.json
│   │   ├── test_flow2_fulfillment.json
│   │   └── test_tier_fallback.json
│   ├── edge_cases/
│   │   ├── test_missing_skus.json
│   │   ├── test_failed_scrapes.json
│   │   ├── test_rate_limiting.json
│   │   └── test_malformed_data.json
│   ├── performance_tests/
│   │   ├── test_sync_performance.json
│   │   ├── test_fulfillment_performance.json
│   │   └── test_database_performance.json
│   └── cost_tests/
│       └── test_cost_tracking.json
└── docs/
    └── testing/
        ├── TEST_PLAN.md
        ├── TEST_RESULTS.md
        └── PERFORMANCE_REPORT.md
```

---

## 📖 Documentation Requirements

**Create 3 documents:**

1. **TEST_PLAN.md:**
   - Test strategy and scope
   - Test environment setup
   - Test data and fixtures
   - Test execution schedule

2. **TEST_RESULTS.md:**
   - Test execution summary
   - Pass/fail rates for each test suite
   - Issues found and resolutions
   - Performance metrics
   - Cost validation results

3. **PERFORMANCE_REPORT.md:**
   - Sync performance (time per vendor, total time)
   - Fulfillment performance (cache hit rate, avg time)
   - Database performance (query times)
   - API performance (response times)
   - Bottlenecks identified
   - Optimization recommendations

---

## ✅ Completion Checklist

- [ ] All tier scrapers tested (unit tests)
- [ ] Flow 1 tested (integration tests)
- [ ] Flow 2 tested (integration tests)
- [ ] Edge cases tested (20+ scenarios)
- [ ] Performance tested (sync, fulfillment, database)
- [ ] Cost validated (<$300/year)
- [ ] All tests documented
- [ ] Test results documented
- [ ] Performance report created
- [ ] Issues found and fixed
- [ ] System ready for production

---

## 📊 Success Metrics

- **Time to Complete:** 6-8 hours
- **Test Coverage:** >90% of workflows
- **Sync Success Rate:** >95%
- **Fulfillment Success Rate:** >99%
- **Sync Duration:** <2 hours for 70 vendors
- **Fulfillment Time:** <10s (cached), <60s (on-demand)
- **Cache Hit Rate:** >90%
- **Annual Cost:** <$300
- **Manual Intervention Rate:** <5%

---

## 🚀 Next Steps

- **Phase 8:** Documentation & Deployment

---

## 💡 Testing Tips

1. **Start with unit tests:** Test each component in isolation
2. **Use real data:** Test with actual vendor websites when possible
3. **Automate tests:** Create n8n workflows that can be re-run
4. **Document failures:** Track all issues found and how they were fixed
5. **Test edge cases:** The weird scenarios are where bugs hide
6. **Measure everything:** Performance, cost, success rates
7. **Optimize iteratively:** Fix bottlenecks one at a time

---

**Phase:** 7 of 8  
**Dependencies:** Phases 5, 6 (both flows complete)  
**Estimated Time:** 6-8 hours  
**Critical Path:** Yes (blocks Phase 8)

