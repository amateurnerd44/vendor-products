# Flow 2: Order Fulfillment Test Scenarios

## Test 1: Happy Path - All SKUs Cached

**Objective**: Verify fast fulfillment when all SKUs are in cache

**Setup**:
1. Ensure product_data table has these SKUs:
   - `TEST-001` (with image_urls and video_urls)
   - `TEST-002` (with image_urls and video_urls)
   - `TEST-003` (with image_urls and video_urls)

**Test Steps**:
1. Send test MarketTime email:
```
From: orders@markettime.com
Subject: Order Confirmation #TEST-001

Customer: Test User
Email: test@example.com
Order Date: 2026-03-19

Items:
- SKU: TEST-001, Qty: 1, Vendor: Test Vendor
- SKU: TEST-002, Qty: 2, Vendor: Test Vendor
- SKU: TEST-003, Qty: 1, Vendor: Test Vendor

Total: $99.99
```

**Expected Results**:
- ✅ Order parsed successfully
- ✅ All 3 SKUs found in cache (100% cache hit rate)
- ✅ Asset package compiled with all images/videos
- ✅ Email delivered to test@example.com
- ✅ Total time: <10 seconds

---

## Test 2: Mixed - Some SKUs Missing

**Objective**: Verify on-demand scraping for cache misses

**Setup**:
1. Ensure product_data table has:
   - `TEST-004` (exists in cache)
   - `TEST-005` (exists in cache)
2. Ensure `TEST-006` does NOT exist in cache
3. Ensure vendor_config has entry for vendor that owns `TEST-006`

**Test Steps**:
1. Send test order with SKUs: TEST-004, TEST-005, TEST-006

**Expected Results**:
- ✅ TEST-004 and TEST-005 found in cache (66.67% cache hit rate)
- ✅ TEST-006 triggers on-demand scraping
- ✅ On-demand scraper matches TEST-006 to vendor
- ✅ Appropriate tier scraper called
- ✅ TEST-006 scraped and saved to product_data
- ✅ Asset package compiled with all 3 products
- ✅ Email delivered
- ✅ Total time: <60 seconds

---

## Test 3: All SKUs Missing

**Objective**: Verify system handles all cache misses

**Setup**:
1. Ensure these SKUs do NOT exist in cache:
   - `TEST-007`
   - `TEST-008`
   - `TEST-009`
2. Ensure vendor_config has entries for vendors

**Test Steps**:
1. Send test order with all missing SKUs

**Expected Results**:
- ✅ 0% cache hit rate logged
- ✅ All 3 SKUs trigger on-demand scraping
- ✅ All scrapers complete successfully
- ✅ All products saved to product_data
- ✅ Asset package compiled
- ✅ Email delivered
- ✅ Total time: <60 seconds

---

## Test 4: Vendor Not Found

**Objective**: Verify error handling for unknown vendors

**Setup**:
1. Create SKU `UNKNOWN-001` that doesn't match any vendor pattern

**Test Steps**:
1. Send test order with SKU: UNKNOWN-001

**Expected Results**:
- ✅ SKU not found in cache
- ✅ On-demand scraper cannot match vendor
- ✅ Admin alert email sent
- ✅ Customer email sent: "We're preparing your assets, will send within 24 hours"
- ✅ Error logged with SKU and reason

---

## Test 5: Shopify Webhook

**Objective**: Verify Shopify webhook processing

**Setup**:
1. Configure SHOPIFY_WEBHOOK_SECRET
2. Ensure product_data has SKU: `SHOPIFY-TEST-001`

**Test Steps**:
1. Send POST request to `/webhook/shopify-orders`:
```json
{
  "id": 999999,
  "order_number": 9999,
  "email": "shopify-test@example.com",
  "customer": {
    "first_name": "Shopify",
    "last_name": "Test"
  },
  "line_items": [
    {
      "sku": "SHOPIFY-TEST-001",
      "quantity": 1,
      "vendor": "Test Vendor"
    }
  ],
  "financial_status": "paid",
  "created_at": "2026-03-19T10:00:00Z"
}
```
2. Include valid HMAC-SHA256 signature in header

**Expected Results**:
- ✅ HMAC signature verified
- ✅ Order data extracted
- ✅ SKU found in cache
- ✅ Asset package compiled
- ✅ Email delivered to shopify-test@example.com
- ✅ 200 OK response returned to Shopify

---

## Test 6: Invalid HMAC Signature

**Objective**: Verify security - reject invalid webhooks

**Test Steps**:
1. Send Shopify webhook with invalid HMAC signature

**Expected Results**:
- ❌ HMAC verification fails
- ❌ Workflow throws error
- ❌ No fulfillment triggered
- ✅ Error logged

---

## Test 7: Email Delivery Failure

**Objective**: Verify retry and error handling

**Setup**:
1. Configure invalid SMTP credentials (to force failure)

**Test Steps**:
1. Send test order

**Expected Results**:
- ✅ Order processed successfully
- ✅ Asset package compiled
- ❌ Email delivery fails
- ✅ Delivery marked as failed
- ✅ Admin alert sent
- ✅ Error logged with order details

---

## Test 8: Performance - Cache Hit Rate

**Objective**: Verify cache hit rate tracking

**Test Steps**:
1. Send 10 test orders with mix of cached and missing SKUs
2. Review SKU Lookup logs

**Expected Results**:
- ✅ Cache hit rate calculated correctly
- ✅ Warning logged if <90%
- ✅ Performance metrics logged

---

## Test 9: On-Demand Scraping Timeout

**Objective**: Verify timeout handling

**Setup**:
1. Configure tier scraper to delay >60 seconds

**Test Steps**:
1. Send order with SKU that triggers slow scraper

**Expected Results**:
- ✅ Scraper times out after 60 seconds
- ✅ SKU marked as failed
- ✅ Admin alert sent
- ✅ Customer notified of delay

---

## Test 10: Multiple Products Per Order

**Objective**: Verify handling of large orders

**Test Steps**:
1. Send order with 10+ SKUs (mix of cached and missing)

**Expected Results**:
- ✅ All SKUs processed
- ✅ Cache hits and misses tracked separately
- ✅ On-demand scraping for missing SKUs
- ✅ Single asset package with all products
- ✅ Single email with all product details
- ✅ Total time: <60 seconds

---

## Performance Benchmarks

| Scenario | Target Time | Acceptable Range |
|----------|-------------|------------------|
| All cached (3 SKUs) | <10s | 5-15s |
| Mixed (2 cached, 1 missing) | <30s | 20-45s |
| All missing (3 SKUs) | <60s | 45-75s |
| Large order (10 SKUs, all cached) | <15s | 10-20s |

---

## Monitoring Checklist

After each test, verify:
- [ ] Cache hit rate logged
- [ ] Scraping duration logged (if applicable)
- [ ] Email delivery status logged
- [ ] No errors in n8n execution logs
- [ ] Customer received email (check inbox)
- [ ] Admin alerts sent when appropriate

---

## Troubleshooting

### Issue: Cache hit rate <90%

**Possible Causes**:
- Phase 5 weekly sync not running
- Vendors not syncing properly
- New products not being added to product_data

**Solution**:
- Run Phase 5 weekly sync manually
- Check vendor_config for active vendors
- Verify data_upserter is populating new fields

### Issue: On-demand scraping fails

**Possible Causes**:
- Vendor not found in vendor_config
- Tier scraper not responding
- Network timeout

**Solution**:
- Add vendor to vendor_config
- Check tier scraper workflows are active
- Increase timeout if needed

### Issue: Email not delivered

**Possible Causes**:
- Invalid SMTP credentials
- Customer email invalid
- Email blocked by spam filter

**Solution**:
- Verify SMTP credentials
- Validate customer email format
- Check spam folder
- Add delivery email to whitelist

