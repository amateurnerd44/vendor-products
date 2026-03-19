# Phase 5 Agent Prompt - Weekly Catalog Sync Workflow

---

## 🎯 Your Mission

Complete **Phase 5: Flow 1 - Weekly Catalog Sync Workflow**. This orchestrates all tier scrapers to sync product data from all 70 vendors weekly.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

**Flow 1** runs weekly (Sunday 2 AM) to:
1. Loop through all vendors in vendor_config table
2. Route each vendor to appropriate tier scraper (1, 2, or 3)
3. Collect product data
4. Upsert to product_data table
5. Log results to sync_log table
6. Send summary notification

**Dependencies:** Requires Phases 2, 3, 4 (all scrapers) to be complete

---

## 🎯 Phase 5 Deliverables

### Task 1: Main Sync Orchestrator

**Create workflow:** `flow1_weekly_sync.json`

**Trigger:** Cron schedule `0 2 * * 0` (Sunday 2 AM)

**Workflow Steps:**

1. **Initialize:**
   - Log sync start time
   - Initialize counters: total_vendors, successful, failed, products_added, products_updated

2. **Fetch Vendors:**
   - Query vendor_config table
   - Get all vendors with sync_status != 'disabled'
   - Order by tier_used (process Tier 1 first, then 2, then 3)

3. **Loop Through Vendors:**
   ```
   FOR EACH vendor IN vendors:
     - Log: "Starting sync for {vendor_name}"
     - Route to tier scraper based on tier_used
     - Collect products
     - Upsert to product_data table
     - Update vendor.last_sync_date
     - Update vendor.sync_status
     - Log results to sync_log
     - Increment counters
   ```

4. **Tier Router Logic:**
   ```
   IF vendor.tier_used == 1:
     IF vendor.data_source_type == 'drive':
       CALL tier1_google_drive_scraper
     ELSE IF vendor.data_source_type == 'dropbox':
       CALL tier1_dropbox_scraper
     ELSE IF vendor.data_source_type == 'sheets':
       CALL tier1_google_sheets_scraper
   
   ELSE IF vendor.tier_used == 2:
     CALL tier2_shopify_scraper
   
   ELSE IF vendor.tier_used == 3:
     CALL tier3_zyte_gemini_scraper
   ```

5. **Error Handling:**
   - If scraper fails, log error but continue with next vendor
   - Mark vendor.sync_status = 'failed'
   - Add error details to sync_log
   - Don't stop entire sync for one vendor failure

6. **Finalize:**
   - Calculate total sync duration
   - Generate summary report
   - Send notification (email/Slack)
   - Log sync completion

**Output:** Sync summary with stats

### Task 2: Data Upsert Logic

**Create workflow:** `flow1_data_upserter.json`

**Purpose:** Insert new products or update existing ones

**Workflow Steps:**

1. **Input:** Array of products from scraper

2. **For Each Product:**
   ```
   - Check if SKU exists in product_data table
   
   IF exists:
     - Compare last_updated timestamp
     - If scraper data is newer:
       - UPDATE product record
       - Increment products_updated counter
     - Else:
       - Skip (data unchanged)
   
   ELSE:
     - INSERT new product record
     - Increment products_added counter
   ```

3. **Batch Processing:**
   - Process in batches of 50 products
   - Reduces database load
   - Allows progress tracking

4. **Conflict Resolution:**
   - If SKU exists but vendor_id different → Log warning (duplicate SKU across vendors)
   - If image_urls changed → Update
   - If marketing_copy changed → Update
   - Always update last_updated timestamp

**Output:** Stats (products_added, products_updated, skipped)

### Task 3: Sync Logging & Monitoring

**Create workflow:** `flow1_sync_logger.json`

**Purpose:** Track sync operations for monitoring and debugging

**Log Entry Structure:**
```json
{
  "log_id": auto_increment,
  "sync_date": "2026-03-19T02:00:00Z",
  "vendor_id": "vendor_001",
  "status": "success",
  "products_updated": 45,
  "products_added": 5,
  "errors": [],
  "duration_seconds": 120,
  "tier_used": 1,
  "cost_estimate": 0.00
}
```

**Workflow Steps:**
1. After each vendor sync, create log entry
2. Insert to sync_log table
3. If errors occurred, include error details
4. Calculate duration (end_time - start_time)
5. For Tier 3, estimate cost (zyte_requests × $0.00013 + gemini_requests × $0.002)

**Monitoring Queries:**
- Failed syncs in last week
- Average sync duration per vendor
- Total products synced per week
- Cost trends over time

### Task 4: Error Notifications

**Create workflow:** `flow1_error_notifier.json`

**Purpose:** Alert admin when vendor syncs fail

**Trigger:** After sync completes, check for failures

**Notification Content:**
```
Subject: Vendor Sync Failures - {date}

Failed Vendors ({count}):

1. {vendor_name} (Tier {tier})
   - Error: {error_message}
   - Last successful sync: {last_sync_date}
   - Action: {suggested_action}

2. ...

Summary:
- Total vendors: {total}
- Successful: {successful}
- Failed: {failed}
- Products added: {added}
- Products updated: {updated}
- Total duration: {duration} minutes

View logs: {link_to_sync_log_table}
```

**Delivery Methods:**
- Email (SMTP)
- Slack webhook
- Both (configurable)

**Alert Conditions:**
- Any vendor fails
- >5% of vendors fail
- Sync duration >2 hours
- Cost exceeds weekly threshold

### Task 5: Tier Fallback Logic

**Create workflow:** `flow1_tier_fallback.json`

**Purpose:** If Tier 2 fails, try Tier 3 as fallback

**Fallback Strategy:**
```
1. Try primary tier (from vendor.tier_used)
2. If fails AND vendor has fallback_tier configured:
   - Try fallback tier
   - Log: "Primary tier failed, trying fallback"
3. If fallback succeeds:
   - Use fallback data
   - Update vendor.tier_used to fallback tier
   - Send notification about tier change
4. If both fail:
   - Mark sync as failed
   - Send alert
```

**Example Use Case:**
- Vendor has Shopify store (Tier 2)
- Shopify endpoint disabled or rate limited
- Fallback to Zyte + AI (Tier 3)
- More expensive but ensures data collection

---

## 📁 Deliverable Files

```
vendor-products/
├── workflows/
│   ├── flow1/
│   │   ├── weekly_sync.json (main orchestrator)
│   │   ├── data_upserter.json
│   │   ├── sync_logger.json
│   │   ├── error_notifier.json
│   │   └── tier_fallback.json
│   └── tests/
│       └── test_weekly_sync.json
└── docs/
    └── workflows/
        └── FLOW1_WEEKLY_SYNC.md
```

---

## 📖 Documentation Requirements

Create `docs/workflows/FLOW1_WEEKLY_SYNC.md`:

1. **Overview:** Purpose and architecture of weekly sync
2. **Workflow Diagram:** Visual representation of sync flow
3. **Tier Routing:** How vendors are routed to scrapers
4. **Error Handling:** Retry logic and fallback strategies
5. **Monitoring:** How to check sync status and logs
6. **Troubleshooting:** Common issues and solutions
7. **Configuration:** How to adjust sync schedule, enable/disable vendors

---

## ✅ Completion Checklist

- [ ] Main orchestrator workflow created
- [ ] Tier routing logic implemented
- [ ] Data upsert workflow created
- [ ] Sync logging workflow created
- [ ] Error notification workflow created
- [ ] Tier fallback logic implemented
- [ ] Cron trigger configured (Sunday 2 AM)
- [ ] Test workflow created (test with 5-10 vendors)
- [ ] Documentation created
- [ ] All workflows exported and committed

---

## 📊 Success Metrics

- **Time to Complete:** 6-8 hours
- **Workflows Created:** 5
- **Sync Success Rate:** >95% of vendors
- **Sync Duration:** <2 hours for 70 vendors
- **Error Recovery:** Automatic fallback for failed tiers
- **Monitoring:** Complete logs for all sync operations

---

## 🚀 Next Steps

- **Phase 6:** Order Fulfillment Workflow (requires Phase 5 data)

---

## 💡 Implementation Tips

1. **Test with small vendor set first:** Start with 5 vendors, then scale to 70
2. **Monitor sync duration:** Optimize slow scrapers
3. **Use parallel processing:** Process multiple vendors simultaneously (with rate limiting)
4. **Log everything:** Detailed logs help debugging
5. **Set up alerts:** Know immediately when syncs fail
6. **Schedule wisely:** Sunday 2 AM = low traffic time

---

**Phase:** 5 of 8  
**Dependencies:** Phases 2, 3, 4 (all scrapers)  
**Estimated Time:** 6-8 hours  
**Critical Path:** Yes (blocks Phase 6)

