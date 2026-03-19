# Flow 1: Weekly Product Catalog Sync

## Overview

The Weekly Product Catalog Sync workflow orchestrates the automated collection of product data from 70+ vendors every Sunday at 2:00 AM. It routes each vendor to the appropriate tier scraper, upserts product data, logs results, and sends notifications for any failures.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Weekly Cron Trigger                          │
│                    (Sunday 2:00 AM)                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ├──────────────────┬─────────────────────┐
                         ▼                  ▼                     ▼
              ┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐
              │ Fetch MarketTime │  │ Fetch All    │  │                  │
              │ Items List       │  │ Vendors      │  │                  │
              │ (Google Drive)   │  │ (n8n Table)  │  │                  │
              └────────┬─────────┘  └──────┬───────┘  │                  │
                       │                   │           │                  │
                       └───────┬───────────┘           │                  │
                               ▼                       │                  │
                    ┌──────────────────────┐           │                  │
                    │ Parse & Merge        │           │                  │
                    │ MarketTime SKUs      │           │                  │
                    └──────────┬───────────┘           │                  │
                               │                       │                  │
                               ▼                       │                  │
                    ┌──────────────────────┐           │                  │
                    │ Loop Through Vendors │◄──────────┘                  │
                    └──────────┬───────────┘                              │
                               │                                          │
                ┌──────────────┼──────────────┐                          │
                ▼              ▼              ▼                          │
         ┌──────────┐   ┌──────────┐   ┌──────────┐                    │
         │ Tier 1?  │   │ Tier 2?  │   │ Tier 3?  │                    │
         └────┬─────┘   └────┬─────┘   └────┬─────┘                    │
              │              │              │                            │
    ┌─────────┼─────────┐    │              │                            │
    ▼         ▼         ▼    ▼              ▼                            │
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                 │
│ Drive  │ │Dropbox │ │Sheets  │ │Shopify │ │ Zyte+  │                 │
│Scraper │ │Scraper │ │Scraper │ │Scraper │ │   AI   │                 │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘                 │
    └──────────┴──────────┴───────────┴──────────┘                      │
                           │                                             │
                           ▼                                             │
                ┌──────────────────────┐                                 │
                │   Data Upserter      │                                 │
                │ (Batch 50 products)  │                                 │
                └──────────┬───────────┘                                 │
                           │                                             │
                           ▼                                             │
                ┌──────────────────────┐                                 │
                │    Sync Logger       │                                 │
                │  (Log to sync_log)   │                                 │
                └──────────┬───────────┘                                 │
                           │                                             │
                           ▼                                             │
                ┌──────────────────────┐                                 │
                │   Sync Failed?       │                                 │
                └──────────┬───────────┘                                 │
                           │                                             │
                    ┌──────┴──────┐                                      │
                    ▼             ▼                                      │
         ┌──────────────────┐  ┌──────────────────┐                     │
         │ Error Notifier   │  │ Update Vendor    │                     │
         │ (Slack/Email)    │  │ Sync Date        │                     │
         └──────────────────┘  └──────────────────┘                     │
                    │             │                                      │
                    └──────┬──────┘                                      │
                           │                                             │
                           └─────────────────────────────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │ Aggregate Results    │
                              │ & Send Summary       │
                              └──────────────────────┘
```

## Components

### 1. Main Orchestrator (`weekly_sync.json`)

**Purpose:** Coordinates the entire weekly sync process

**Key Features:**
- Cron trigger: `0 2 * * 0` (Sunday 2:00 AM)
- Fetches MarketTime items list from Google Drive for new item detection
- Fetches all vendors from `vendor_config` table
- Loops through vendors one at a time
- Routes to appropriate tier scraper based on `tier_used` field
- Aggregates results and sends summary notification

**Environment Variables Required:**
- `MARKETTIME_ITEMS_FOLDER_ID`: Google Drive file ID for MarketTime items list

**Workflow Logic:**
1. Trigger on schedule
2. Fetch MarketTime items list (CSV) from Google Drive
3. Parse CSV to create SKU lookup set
4. Fetch all vendors from database
5. Merge MarketTime data with vendor data
6. For each vendor:
   - Route to Tier 1, 2, or 3 scraper
   - Call Data Upserter with results
   - Log sync results
   - Check for failures and notify if needed
   - Update vendor's `last_sync_date`
7. Aggregate all results
8. Send summary notification

### 2. Data Upserter (`data_upserter.json`)

**Purpose:** Efficiently upsert product data into the database

**Key Features:**
- Batch processing (50 products at a time)
- Checks if SKU exists in database
- Inserts new products
- Updates existing products only if data is newer
- Tracks products added, updated, and skipped
- Identifies new MarketTime items

**Logic Flow:**
1. Receive products from scraper
2. Check each product against MarketTime SKU list
3. Batch products into groups of 50
4. For each product:
   - Query database for existing SKU
   - If not found: INSERT new product
   - If found: Compare timestamps
     - If newer: UPDATE product
     - If older: SKIP (no update needed)
5. Tag each result with action taken (inserted/updated/skipped)
6. Aggregate statistics
7. Return summary with counts

**Performance:**
- Processes 50 products per batch
- Typical sync time: <2 hours for 70 vendors (~7,000 products)

### 3. Sync Logger (`sync_logger.json`)

**Purpose:** Log sync results to `sync_log` table

**Key Features:**
- Calculates sync duration
- Determines sync status (success/partial/failed)
- Estimates cost for Tier 3 vendors
- Tracks products added/updated/skipped
- Logs errors for troubleshooting

**Log Entry Fields:**
- `sync_date`: Timestamp of sync completion
- `vendor_id`: Vendor identifier
- `vendor_name`: Vendor name
- `status`: success | partial | failed
- `products_updated`: Count of updated products
- `products_added`: Count of new products
- `products_skipped`: Count of skipped products
- `errors`: Array of error messages
- `duration_seconds`: Sync duration
- `tier_used`: Tier used for this sync
- `cost_estimate`: Estimated cost (Tier 3 only)
- `new_markettime_items`: Count of new MarketTime items found

### 4. Error Notifier (`error_notifier.json`)

**Purpose:** Send alerts for failed vendor syncs

**Key Features:**
- Sends notifications via Slack and Email
- Includes vendor details and error information
- Provides tier-specific suggested actions
- Tracks notification delivery status

**Notification Content:**
- Vendor name and ID
- Tier used
- Data source type
- Error details
- Suggested troubleshooting actions
- Sync statistics

**Suggested Actions by Tier:**
- **Tier 1:** Check permissions, verify folder/file IDs, validate data structure
- **Tier 2:** Verify Shopify URL, check `/products.json` endpoint, review rate limits
- **Tier 3:** Check Zyte/Gemini API status, review website structure, update prompts

**Environment Variables Required:**
- `SLACK_WEBHOOK_URL`: Slack webhook for notifications
- `SLACK_ALERT_CHANNEL`: Slack channel for alerts (default: #alerts)
- `NOTIFICATION_FROM_EMAIL`: Email sender address
- `NOTIFICATION_TO_EMAIL`: Email recipient address

### 5. Tier Fallback Handler (`tier_fallback.json`)

**Purpose:** Automatically fallback to Tier 3 when Tier 2 fails

**Key Features:**
- Only applies to Tier 2 vendors with `fallback_to_tier3 = true`
- Attempts Tier 3 scraping if Tier 2 fails
- Permanently updates vendor config if fallback succeeds
- Sends notifications for both success and failure

**Fallback Logic:**
1. Check if vendor is eligible for fallback (Tier 2 + fallback enabled)
2. If eligible and Tier 2 failed:
   - Log fallback attempt
   - Call Tier 3 scraper (Zyte + AI)
   - Check if Tier 3 succeeded
3. If Tier 3 succeeds:
   - Update `vendor_config` to permanently use Tier 3
   - Send success notification
4. If Tier 3 fails:
   - Send critical failure notification
   - Require manual intervention

**Vendor Config Updates on Success:**
- `tier_used`: 3
- `data_source_type`: "website"
- `fallback_success_date`: Current timestamp
- `notes`: Append fallback success message

## Testing

### Test Workflow (`test_weekly_sync.json`)

**Purpose:** Validate weekly sync functionality with test data

**Test Scenario:**
- 10 test vendors across all tiers
- 2 intentionally broken vendors (1 Tier 1, 1 Tier 2)
- Expected: 8 successful syncs, 2 failures

**Test Vendors:**
1. Test Drive Vendor (Tier 1)
2. Test Dropbox Vendor (Tier 1)
3. Test Sheets Vendor (Tier 1)
4. Test Shopify Vendor 1 (Tier 2, fallback enabled)
5. Test Shopify Vendor 2 (Tier 2, fallback enabled)
6. Test Website Vendor 1 (Tier 3)
7. Test Website Vendor 2 (Tier 3)
8. Test Broken Drive Vendor (Tier 1, should fail)
9. Test Broken Shopify Vendor (Tier 2, should fail)
10. Test Mixed Vendor (Tier 1)

**Validation Checks:**
- Total vendors processed = 10
- Successful syncs = 8
- Failed syncs = 2
- Notifications sent for failures
- Products added/updated counts
- Tier distribution correct

**How to Run:**
1. Import `test_weekly_sync.json` into n8n
2. Click "Execute Workflow" (manual trigger)
3. Review test results
4. Check validation summary

**Expected Output:**
```
✅ Weekly Sync Test PASSED

Test Summary:
- Total Vendors: 10/10
- Successful Syncs: 8
- Failed Syncs: 2/2 (expected)
- Tier 1 Processed: 5
- Tier 2 Processed: 2
- Tier 3 Processed: 2
- Products Added: [varies]
- Products Updated: [varies]
- Notifications Sent: 2

Status: All systems operational ✅
```

## Configuration

### Environment Variables

Add these to your n8n environment:

```bash
# MarketTime Integration
MARKETTIME_ITEMS_FOLDER_ID=your_google_drive_file_id

# Notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_ALERT_CHANNEL=#alerts
NOTIFICATION_FROM_EMAIL=noreply@yourdomain.com
NOTIFICATION_TO_EMAIL=admin@yourdomain.com
```

### Vendor Config Table

Ensure your `vendor_config` table has these fields:

```sql
CREATE TABLE vendor_config (
  vendor_id VARCHAR(255) PRIMARY KEY,
  vendor_name VARCHAR(255) NOT NULL,
  data_source_type ENUM('drive', 'dropbox', 'sheets', 'shopify', 'website'),
  source_url TEXT,
  credentials JSON,
  last_sync_date TIMESTAMP,
  sync_status ENUM('success', 'failed', 'pending'),
  tier_used INT CHECK (tier_used IN (1, 2, 3)),
  fallback_to_tier3 BOOLEAN DEFAULT FALSE,
  notes TEXT
);
```

### MarketTime Items List Format

The Google Drive file should be a CSV with this structure:

```csv
sku,product_name,vendor_id,date_added
SKU001,Product Name 1,vendor_001,2026-03-15
SKU002,Product Name 2,vendor_002,2026-03-16
...
```

**Purpose:** This list helps identify new items that have been added to MarketTime, allowing the system to prioritize syncing these products.

## Monitoring

### Success Metrics

- **Sync Success Rate:** >95% of vendors sync successfully
- **Sync Duration:** <2 hours for 70 vendors
- **Products Processed:** ~7,000 products per week
- **Error Rate:** <5% of vendors require manual intervention

### Key Metrics to Track

1. **Per-Vendor Metrics:**
   - Sync status (success/failed)
   - Products added/updated
   - Sync duration
   - Tier used
   - Cost estimate (Tier 3)

2. **Aggregate Metrics:**
   - Total vendors synced
   - Success rate by tier
   - Total products added/updated
   - Total sync duration
   - Total cost (Tier 3)
   - New MarketTime items found

3. **Error Metrics:**
   - Failed vendor count
   - Error types by tier
   - Fallback success rate
   - Notification delivery rate

### Monitoring Queries

**Check Recent Sync Status:**
```sql
SELECT 
  vendor_name,
  status,
  products_added,
  products_updated,
  duration_seconds,
  sync_date
FROM sync_log
WHERE sync_date >= NOW() - INTERVAL '7 days'
ORDER BY sync_date DESC;
```

**Identify Problematic Vendors:**
```sql
SELECT 
  vendor_name,
  COUNT(*) as failure_count,
  MAX(sync_date) as last_attempt
FROM sync_log
WHERE status = 'failed'
  AND sync_date >= NOW() - INTERVAL '30 days'
GROUP BY vendor_name
HAVING COUNT(*) > 2
ORDER BY failure_count DESC;
```

**Calculate Weekly Costs:**
```sql
SELECT 
  SUM(cost_estimate) as total_cost,
  COUNT(*) as tier3_syncs,
  AVG(cost_estimate) as avg_cost_per_sync
FROM sync_log
WHERE tier_used = 3
  AND sync_date >= NOW() - INTERVAL '7 days';
```

## Troubleshooting

### Common Issues

#### 1. Vendor Sync Fails Repeatedly

**Symptoms:** Same vendor fails multiple syncs in a row

**Diagnosis:**
1. Check `sync_log` table for error details
2. Review vendor's `data_source_type` and `source_url`
3. Verify credentials are valid
4. Test data source manually

**Solutions:**
- **Tier 1:** Check Google Drive/Dropbox/Sheets permissions
- **Tier 2:** Verify Shopify store is accessible, check rate limits
- **Tier 3:** Review Zyte/Gemini API status, update extraction prompts

#### 2. Sync Takes Too Long

**Symptoms:** Sync duration >2 hours for 70 vendors

**Diagnosis:**
1. Check which tier is slowest
2. Review product counts per vendor
3. Check for rate limiting issues

**Solutions:**
- Optimize Tier 3 prompts for faster extraction
- Increase batch size in Data Upserter (if database can handle it)
- Consider running syncs in parallel (advanced)

#### 3. MarketTime Items Not Detected

**Symptoms:** `new_markettime_items` count is always 0

**Diagnosis:**
1. Verify `MARKETTIME_ITEMS_FOLDER_ID` is correct
2. Check CSV format matches expected structure
3. Ensure SKUs in CSV match product SKUs

**Solutions:**
- Update Google Drive file ID
- Standardize SKU format across systems
- Review CSV parsing logic in `weekly_sync.json`

#### 4. Notifications Not Sending

**Symptoms:** Failed syncs but no alerts received

**Diagnosis:**
1. Check Slack webhook URL is valid
2. Verify email SMTP credentials
3. Review error notifier logs

**Solutions:**
- Test Slack webhook manually
- Verify SMTP settings in n8n
- Check firewall/network restrictions

### Debug Mode

To enable detailed logging:

1. Add this to the start of any Code node:
```javascript
console.log('DEBUG:', JSON.stringify($input.all(), null, 2));
```

2. Check n8n execution logs for output

3. Review `sync_log` table for detailed error information

## Maintenance

### Weekly Tasks

- [ ] Review sync success rate
- [ ] Check for new failed vendors
- [ ] Monitor Tier 3 costs
- [ ] Verify MarketTime items list is up to date

### Monthly Tasks

- [ ] Analyze sync performance trends
- [ ] Review and optimize slow scrapers
- [ ] Update vendor configs as needed
- [ ] Audit fallback success rate

### Quarterly Tasks

- [ ] Review and update AI extraction prompts
- [ ] Optimize database queries
- [ ] Update documentation
- [ ] Evaluate new scraping tools/APIs

## Cost Estimates

### Weekly Sync (70 vendors)

**Tier 1 (30 vendors):** FREE
**Tier 2 (20 vendors):** FREE
**Tier 3 (20 vendors):**
- Zyte API: 2,000 pages × $0.00013 = $0.26
- Gemini AI: 2,000 pages × $0.002 = $4.00
- **Weekly Total: ~$4.26**

**Annual Total: ~$221**

### Optimization Tips

1. **Minimize Tier 3 Usage:**
   - Prioritize Tier 1 and Tier 2 vendors
   - Only use Tier 3 when necessary

2. **Optimize AI Prompts:**
   - Reduce token usage
   - Improve extraction accuracy to avoid retries

3. **Batch Processing:**
   - Process multiple products per API call when possible

4. **Caching:**
   - Cache frequently accessed data
   - Avoid redundant API calls

## Future Enhancements

### Planned Features

1. **Parallel Vendor Processing:**
   - Process multiple vendors simultaneously
   - Reduce total sync time

2. **Incremental Syncs:**
   - Only sync products that have changed
   - Reduce API usage and costs

3. **Smart Scheduling:**
   - Sync high-priority vendors more frequently
   - Adjust schedule based on vendor update patterns

4. **Advanced Fallback:**
   - Multi-tier fallback strategies
   - Automatic tier optimization based on success rates

5. **Real-time Monitoring Dashboard:**
   - Live sync progress tracking
   - Visual analytics and alerts

## Support

For issues or questions:
1. Check this documentation
2. Review `sync_log` table for error details
3. Test with `test_weekly_sync.json`
4. Contact system administrator

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-19  
**Author:** Codegen AI  
**Related Workflows:** Flow 2 (Order Fulfillment)

