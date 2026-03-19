# Flow 1: Weekly Product Catalog Sync

## Quick Start

This directory contains all workflows for the Weekly Product Catalog Sync system (Flow 1).

### Workflows

| Workflow | Purpose | Trigger |
|----------|---------|---------|
| `weekly_sync.json` | Main orchestrator | Cron (Sunday 2 AM) |
| `data_upserter.json` | Upsert products to database | Called by orchestrator |
| `sync_logger.json` | Log sync results | Called by orchestrator |
| `error_notifier.json` | Send failure alerts | Called by orchestrator |
| `tier_fallback.json` | Handle Tier 2→3 fallback | Called by orchestrator |
| `test_weekly_sync.json` | Test suite | Manual trigger |

### Import Order

Import workflows in this order to avoid dependency issues:

1. `data_upserter.json`
2. `sync_logger.json`
3. `error_notifier.json`
4. `tier_fallback.json`
5. `weekly_sync.json` (main orchestrator)
6. `test_weekly_sync.json` (optional, for testing)

### Configuration

Before running, configure these environment variables in n8n:

```bash
# Required
MARKETTIME_ITEMS_FOLDER_ID=your_google_drive_file_id

# Optional (for notifications)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_ALERT_CHANNEL=#alerts
NOTIFICATION_FROM_EMAIL=noreply@yourdomain.com
NOTIFICATION_TO_EMAIL=admin@yourdomain.com
```

### Testing

1. Import `test_weekly_sync.json`
2. Click "Execute Workflow"
3. Review results

Expected: 8 successful syncs, 2 failures (intentional)

### Documentation

See [FLOW1_WEEKLY_SYNC.md](../../docs/workflows/FLOW1_WEEKLY_SYNC.md) for complete documentation.

## Workflow Details

### Main Orchestrator (`weekly_sync.json`)

**Cron Schedule:** `0 2 * * 0` (Sunday 2:00 AM)

**Process:**
1. Fetch MarketTime items list from Google Drive
2. Fetch all vendors from database
3. Loop through each vendor
4. Route to appropriate tier scraper
5. Upsert products to database
6. Log results
7. Send notifications for failures
8. Update vendor sync dates
9. Send summary notification

**Key Features:**
- MarketTime integration for new item detection
- Automatic tier routing
- Batch processing
- Error handling with notifications
- Comprehensive logging

### Data Upserter (`data_upserter.json`)

**Purpose:** Efficiently upsert products to database

**Features:**
- Batch processing (50 products at a time)
- Smart upsert logic (only update if newer)
- MarketTime item detection
- Detailed statistics tracking

**Performance:**
- ~7,000 products in <2 hours
- Minimal database load

### Sync Logger (`sync_logger.json`)

**Purpose:** Log all sync activity

**Logs:**
- Sync status (success/partial/failed)
- Products added/updated/skipped
- Sync duration
- Cost estimates (Tier 3)
- Error details
- New MarketTime items

### Error Notifier (`error_notifier.json`)

**Purpose:** Alert on sync failures

**Notifications:**
- Slack messages
- Email alerts
- Detailed error information
- Suggested troubleshooting actions

### Tier Fallback Handler (`tier_fallback.json`)

**Purpose:** Automatic Tier 2→3 fallback

**Logic:**
- Only for Tier 2 vendors with fallback enabled
- Attempts Tier 3 if Tier 2 fails
- Permanently updates vendor config on success
- Sends notifications for both outcomes

## Success Metrics

- **Sync Success Rate:** >95%
- **Sync Duration:** <2 hours for 70 vendors
- **Cost:** ~$4.26/week (~$221/year)
- **Manual Intervention:** <5% of syncs

## Monitoring

### Check Recent Syncs

Query the `sync_log` table:

```sql
SELECT vendor_name, status, products_added, products_updated, sync_date
FROM sync_log
WHERE sync_date >= NOW() - INTERVAL '7 days'
ORDER BY sync_date DESC;
```

### Identify Problem Vendors

```sql
SELECT vendor_name, COUNT(*) as failures
FROM sync_log
WHERE status = 'failed' AND sync_date >= NOW() - INTERVAL '30 days'
GROUP BY vendor_name
HAVING COUNT(*) > 2
ORDER BY failures DESC;
```

## Troubleshooting

### Sync Fails

1. Check `sync_log` table for error details
2. Verify vendor config (source_url, credentials)
3. Test data source manually
4. Review tier-specific troubleshooting in main docs

### Notifications Not Sending

1. Verify Slack webhook URL
2. Check email SMTP settings
3. Test manually with `error_notifier.json`

### Slow Performance

1. Check which tier is slowest
2. Review product counts per vendor
3. Optimize Tier 3 prompts
4. Consider increasing batch size

## Cost Optimization

1. **Minimize Tier 3 usage** - Prioritize Tier 1 and 2
2. **Optimize AI prompts** - Reduce token usage
3. **Batch processing** - Process multiple products per call
4. **Caching** - Avoid redundant API calls

## Support

- **Documentation:** [FLOW1_WEEKLY_SYNC.md](../../docs/workflows/FLOW1_WEEKLY_SYNC.md)
- **Test Suite:** `test_weekly_sync.json`
- **Logs:** Check `sync_log` table

---

**Version:** 1.0  
**Last Updated:** 2026-03-19

