# Phase 7: Testing & Validation
## Test Suite for Vendor Product Data Collection System

**Version:** 1.0  
**Date:** 2026-03-19  
**Status:** Ready for Execution

---

## 📁 Directory Structure

```
tests/
├── README.md (this file)
├── schema_tests/
│   └── test_hybrid_schema.json
├── tier_tests/
│   ├── test_tier1_media_extraction.md
│   ├── test_tier2_shopify.md (to be created)
│   └── test_tier3_zyte_ai.md (to be created)
├── integration_tests/
│   ├── test_flow1_weekly_sync.md
│   └── test_flow2_order_fulfillment.md (to be created)
├── edge_cases/
│   └── test_edge_cases.md (to be created)
├── performance_tests/
│   └── test_performance.md (to be created)
└── cost_tests/
    └── test_cost_validation.md (to be created)
```

---

## 🎯 Test Objectives

Phase 7 validates the complete system with special focus on the hybrid schema implementation from Phase 6:

### Critical Validations
1. **Hybrid Schema (35 fields)** - All 31 existing + 4 new fields working
2. **Media Asset Collection** - Images and videos extracted by all tiers
3. **Vendor Routing** - `vendor_id` and `source_tier` correctly assigned
4. **Flow 1 (Weekly Sync)** - Complete sync workflow with 70 vendors
5. **Flow 2 (Order Fulfillment)** - Cached and on-demand fulfillment
6. **Performance Targets** - <2h sync, <10s cached, <60s on-demand
7. **Cost Targets** - <$300/year budget compliance

---

## 📋 Test Categories

### 1. Schema Validation Tests (`schema_tests/`)
**Purpose:** Validate the hybrid schema structure and data integrity

**Key Tests:**
- All 35 fields exist and accessible
- New Phase 6 fields (image_urls, video_urls, vendor_id, source_tier)
- Data types and constraints
- Data upserter populates new fields
- Batch processing with JSON arrays

**Files:**
- `test_hybrid_schema.json` - n8n workflow for automated schema validation

**Expected Results:**
- ✅ All 35 fields present
- ✅ JSON arrays properly formatted
- ✅ vendor_id and source_tier populated
- ✅ No data loss from Phase 5 fields

---

### 2. Enhanced Unit Tests (`tier_tests/`)
**Purpose:** Test each tier scraper with media extraction capabilities

**Key Tests:**

#### Tier 1 (Google Drive, Dropbox, Sheets)
- Image extraction (JPG, PNG, WebP, GIF)
- Video extraction (MP4, MOV, AVI)
- Shareable link generation
- Nested folder structures
- Large media collections (50+ images)
- Empty media arrays

#### Tier 2 (Shopify)
- Shopify JSON endpoint parsing
- Image extraction from `product.images`
- Video extraction from `product.media`
- Pagination handling
- Fallback to Tier 3

#### Tier 3 (Zyte + Gemini AI)
- HTML extraction via Zyte
- AI-powered media URL extraction
- Cost tracking
- Rate limiting compliance

**Files:**
- `test_tier1_media_extraction.md` - Comprehensive Tier 1 test cases
- `test_tier2_shopify.md` - Tier 2 test cases (to be created)
- `test_tier3_zyte_ai.md` - Tier 3 test cases (to be created)

**Expected Results:**
- ✅ All tiers extract media URLs correctly
- ✅ vendor_id assigned from vendor_config
- ✅ source_tier tracks collection method (1, 2, or 3)
- ✅ Media URLs in proper JSON array format

---

### 3. Integration Tests (`integration_tests/`)
**Purpose:** Test complete workflows end-to-end

**Key Tests:**

#### Flow 1: Weekly Sync
- Full sync (70 vendors, <2 hours)
- Partial failure handling
- Tier fallback (Tier 2 → Tier 3)
- MarketTime CSV import
- Performance validation
- Data quality checks

#### Flow 2: Order Fulfillment
- Cached SKU lookup (<10s)
- On-demand scraping (<60s)
- MarketTime email trigger
- Shopify webhook trigger
- Mixed orders (cached + on-demand)
- Delivery methods (Email, Drive, Dropbox, Download)

**Files:**
- `test_flow1_weekly_sync.md` - Flow 1 integration tests
- `test_flow2_order_fulfillment.md` - Flow 2 integration tests (to be created)

**Expected Results:**
- ✅ Sync completes in <2 hours
- ✅ >95% success rate
- ✅ Cached fulfillment <10s
- ✅ On-demand fulfillment <60s
- ✅ All delivery methods functional

---

### 4. Edge Case Tests (`edge_cases/`)
**Purpose:** Test unusual scenarios and error conditions

**Key Tests:**
- Empty media arrays
- Large media arrays (50+ images, 10+ videos)
- Invalid media URLs
- Malformed data (wrong types)
- Duplicate SKUs with vendor_id tracking
- Missing vendor configs
- Rate limiting scenarios
- Large orders (50+ SKUs)
- Concurrent orders
- Database errors

**Files:**
- `test_edge_cases.md` - Comprehensive edge case scenarios (to be created)

**Expected Results:**
- ✅ Graceful handling of all edge cases
- ✅ Proper error logging
- ✅ Admin notifications for critical failures
- ✅ No data corruption

---

### 5. Performance Tests (`performance_tests/`)
**Purpose:** Validate system performance meets targets

**Key Tests:**
- Sync performance (70 vendors, <2 hours)
- Fulfillment performance (<10s cached, <60s on-demand)
- Database performance (queries, upserts)
- Media extraction timing
- API response times
- Bottleneck identification

**Files:**
- `test_performance.md` - Performance test scenarios (to be created)

**Expected Results:**
- ✅ All performance targets met
- ✅ No significant bottlenecks
- ✅ Scalable to 100+ vendors

---

### 6. Cost Validation Tests (`cost_tests/`)
**Purpose:** Verify system stays within $300/year budget

**Key Tests:**
- Weekly sync cost (~$4.26)
- Order fulfillment cost (~$0.04/month)
- Annual projection (<$300)
- Cost tracking accuracy
- Cache hit rate (>90%)

**Files:**
- `test_cost_validation.md` - Cost validation scenarios (to be created)

**Expected Results:**
- ✅ Total annual cost <$300
- ✅ Cost tracking accurate
- ✅ Cache hit rate >90%

---

## 🧪 Test Execution

### Prerequisites
1. n8n Cloud account with workflows deployed
2. Test vendor configs in vendor_config table
3. Test data prepared (Drive, Dropbox, Sheets)
4. Monitoring tools configured (n8n logs, Slack)
5. Backup of product_data table

### Execution Order
1. **Schema Validation** (2 hours)
   - Run `test_hybrid_schema.json` workflow
   - Validate all 35 fields
   - Check Phase 6 new fields

2. **Unit Tests** (8 hours)
   - Test Tier 1 scrapers
   - Test Tier 2 scrapers
   - Test Tier 3 scrapers
   - Validate media extraction

3. **Integration Tests** (6 hours)
   - Test Flow 1 (Weekly Sync)
   - Test Flow 2 (Order Fulfillment)
   - Validate end-to-end workflows

4. **Edge Cases** (4 hours)
   - Test unusual scenarios
   - Validate error handling
   - Check recovery mechanisms

5. **Performance Tests** (3 hours)
   - Measure sync performance
   - Measure fulfillment performance
   - Identify bottlenecks

6. **Cost Validation** (2 hours)
   - Calculate actual costs
   - Validate cost tracking
   - Project annual costs

7. **Documentation** (3 hours)
   - Complete TEST_RESULTS.md
   - Complete PERFORMANCE_REPORT.md
   - Update IMPLEMENTATION_PLAN.md

**Total Estimated Time:** 28 hours

---

## ✅ Success Criteria

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

## 🐛 Issue Tracking

### Issue Priority Levels
- **P0 (Blocker):** Prevents deployment, must fix immediately
- **P1 (Critical):** Major functionality broken, fix before deployment
- **P2 (High):** Important feature impacted, fix soon
- **P3 (Medium):** Minor issue, fix when possible
- **P4 (Low):** Nice to have, backlog

### Issue Template
```markdown
**Issue ID:** TEST-XXX
**Priority:** P0/P1/P2/P3/P4
**Category:** Schema/Tier/Integration/Edge/Performance/Cost
**Title:** Brief description
**Description:** Detailed description
**Steps to Reproduce:** 
1. Step 1
2. Step 2
**Expected Result:** What should happen
**Actual Result:** What actually happened
**Impact:** How this affects the system
**Recommendation:** Suggested fix
**Status:** Open/In Progress/Fixed/Closed
```

---

## 📝 Documentation

### Test Documentation Files
1. **TEST_PLAN.md** (`docs/testing/`)
   - Test strategy and approach
   - Test cases and scenarios
   - Execution plan and timeline

2. **TEST_RESULTS.md** (`docs/testing/`)
   - Test execution results
   - Pass/fail status for each test
   - Issues and bugs found
   - Recommendations

3. **PERFORMANCE_REPORT.md** (`docs/testing/`)
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

## 🔄 Next Steps

After Phase 7 completion:
1. Review test results with stakeholders
2. Address any critical issues found
3. Optimize performance bottlenecks
4. Update documentation based on findings
5. Proceed to Phase 8: Documentation & Deployment

---

## 📞 Support

For questions or issues with testing:
- Review test documentation in `docs/testing/`
- Check IMPLEMENTATION_PLAN.md for context
- Consult workflow README files in `workflows/`

---

**Document Status:** ✅ Ready for Execution  
**Last Updated:** 2026-03-19  
**Next Review:** After test execution

