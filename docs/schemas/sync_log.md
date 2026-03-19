# Sync Log Table Schema

## Overview
The `sync_log` table records the history of all vendor sync operations. This table is essential for monitoring system health, debugging sync failures, tracking performance, and analyzing cost efficiency.

## Table Name
`sync_log`

## Schema Definition

| Field Name | Data Type | Constraints | Description |
|------------|-----------|-------------|-------------|
| `log_id` | Number | PRIMARY KEY, AUTO_INCREMENT | Unique identifier for each sync log entry |
| `sync_date` | DateTime | NOT NULL | Timestamp when the sync operation started |
| `vendor_id` | Text | NOT NULL | Reference to vendor_config.vendor_id |
| `status` | Text | NOT NULL | Sync result: 'success', 'partial', 'failed' |
| `products_updated` | Number | DEFAULT 0 | Count of products that were updated during sync |
| `products_added` | Number | DEFAULT 0 | Count of new products added during sync |
| `errors` | JSON | NULLABLE | Array of error messages and details if sync failed |
| `duration_seconds` | Number | NULLABLE | Time taken to complete the sync operation in seconds |

## Field Details

### log_id
- **Format**: Auto-incrementing integer
- **Purpose**: Unique identifier for each sync operation
- **Auto-generated**: Automatically assigned by database
- **Usage**: Reference specific sync operations in reports and debugging

### sync_date
- **Format**: ISO 8601 DateTime (e.g., "2026-03-19T02:00:00Z")
- **Purpose**: Record when sync operation started
- **Required**: Cannot be null
- **Timezone**: Store in UTC for consistency
- **Usage**: 
  - Track sync frequency
  - Identify sync patterns
  - Generate time-series reports
  - Detect missed syncs

### vendor_id
- **Format**: `vendor_XXX` matching vendor_config.vendor_id
- **Purpose**: Link sync operation to specific vendor
- **Foreign Key**: References `vendor_config.vendor_id`
- **Required**: Cannot be null
- **Usage**: 
  - Filter logs by vendor
  - Track vendor-specific sync success rates
  - Identify problematic vendors

### status
- **Valid Values**:
  - `success` - All products synced successfully
  - `partial` - Some products synced, some failed
  - `failed` - Sync operation completely failed
- **Purpose**: Quick status indicator for sync result
- **Required**: Cannot be null
- **Usage**:
  - Alert on failed syncs
  - Calculate success rates
  - Identify vendors needing attention

### products_updated
- **Format**: Non-negative integer
- **Purpose**: Count of existing products that were updated
- **Default**: 0
- **Usage**:
  - Track data freshness
  - Measure sync efficiency
  - Identify vendors with frequent updates
  - Calculate sync impact

### products_added
- **Format**: Non-negative integer
- **Purpose**: Count of new products discovered and added
- **Default**: 0
- **Usage**:
  - Track catalog growth
  - Identify vendors adding new products
  - Measure sync completeness
  - Generate growth reports

### errors
- **Format**: JSON array of error objects
- **Structure**:
  ```json
  [
    {
      "timestamp": "2026-03-19T02:15:30Z",
      "error_type": "authentication_failed",
      "message": "OAuth token expired",
      "sku": null,
      "details": {
        "http_status": 401,
        "response": "Unauthorized"
      }
    },
    {
      "timestamp": "2026-03-19T02:16:45Z",
      "error_type": "product_parse_error",
      "message": "Failed to extract SKU from filename",
      "sku": "unknown",
      "details": {
        "filename": "product_image.jpg",
        "expected_format": "SKU-name.jpg"
      }
    }
  ]
  ```
- **Nullable**: Can be null or empty array `[]` for successful syncs
- **Purpose**: Store detailed error information for debugging
- **Error Types**:
  - `authentication_failed` - Credential issues
  - `rate_limit_exceeded` - API rate limiting
  - `network_error` - Connection issues
  - `product_parse_error` - Data extraction failures
  - `invalid_data` - Data validation failures
  - `api_error` - Third-party API errors (Zyte, Gemini)

### duration_seconds
- **Format**: Positive number (can be decimal for sub-second precision)
- **Purpose**: Track sync performance and identify slow operations
- **Nullable**: Can be null if sync was interrupted
- **Usage**:
  - Performance monitoring
  - Identify slow vendors
  - Optimize sync operations
  - Capacity planning
- **Typical Values**:
  - Tier 1 (Drive/Dropbox/Sheets): 10-60 seconds
  - Tier 2 (Shopify): 30-120 seconds
  - Tier 3 (Zyte + AI): 60-300 seconds

## Relationships

### Foreign Key References
- `vendor_id` references `vendor_config.vendor_id` (many-to-one)

## Indexes

### Recommended Indexes
1. **Primary Key**: `log_id` (automatic)
2. **Vendor Lookup**: `vendor_id` (for vendor-specific logs)
3. **Date Range**: `sync_date` (for time-based queries)
4. **Status Filter**: `status` (for finding failures)
5. **Composite**: `(vendor_id, sync_date)` (for vendor history)

## Usage Examples

### Successful Sync (Tier 1 - Google Drive)
```json
{
  "log_id": 1,
  "sync_date": "2026-03-19T02:00:00Z",
  "vendor_id": "vendor_001",
  "status": "success",
  "products_updated": 45,
  "products_added": 3,
  "errors": [],
  "duration_seconds": 32.5
}
```

### Partial Sync (Tier 2 - Shopify)
```json
{
  "log_id": 2,
  "sync_date": "2026-03-19T02:05:00Z",
  "vendor_id": "vendor_002",
  "status": "partial",
  "products_updated": 78,
  "products_added": 5,
  "errors": [
    {
      "timestamp": "2026-03-19T02:06:30Z",
      "error_type": "rate_limit_exceeded",
      "message": "Shopify rate limit reached, some products skipped",
      "sku": null,
      "details": {
        "products_skipped": 12,
        "retry_after": 60
      }
    }
  ],
  "duration_seconds": 95.2
}
```

### Failed Sync (Tier 3 - Website)
```json
{
  "log_id": 3,
  "sync_date": "2026-03-19T02:10:00Z",
  "vendor_id": "vendor_005",
  "status": "failed",
  "products_updated": 0,
  "products_added": 0,
  "errors": [
    {
      "timestamp": "2026-03-19T02:10:15Z",
      "error_type": "authentication_failed",
      "message": "Zyte API key invalid or expired",
      "sku": null,
      "details": {
        "http_status": 403,
        "api_response": "Invalid API key"
      }
    }
  ],
  "duration_seconds": 15.0
}
```

## n8n Implementation Notes

### Creating the Table in n8n
1. Navigate to **Data** → **Data Tables** in n8n Cloud
2. Click **Create Table**
3. Name: `sync_log`
4. Add fields according to schema above
5. Set `log_id` as Primary Key with AUTO_INCREMENT
6. Create indexes on `vendor_id` and `sync_date`

### Logging a Sync Operation
```javascript
// Start sync
const syncStartTime = Date.now();
const syncDate = new Date().toISOString();
let productsUpdated = 0;
let productsAdded = 0;
const errors = [];

try {
  // Perform sync operations
  // ... sync logic here ...
  
  // Track results
  productsUpdated = 45;
  productsAdded = 3;
  
  // Calculate duration
  const durationSeconds = (Date.now() - syncStartTime) / 1000;
  
  // Log success
  $('Data Table').insert({
    sync_date: syncDate,
    vendor_id: 'vendor_001',
    status: 'success',
    products_updated: productsUpdated,
    products_added: productsAdded,
    errors: [],
    duration_seconds: durationSeconds
  });
  
} catch (error) {
  // Log failure
  const durationSeconds = (Date.now() - syncStartTime) / 1000;
  
  $('Data Table').insert({
    sync_date: syncDate,
    vendor_id: 'vendor_001',
    status: 'failed',
    products_updated: productsUpdated,
    products_added: productsAdded,
    errors: [{
      timestamp: new Date().toISOString(),
      error_type: 'sync_error',
      message: error.message,
      sku: null,
      details: { stack: error.stack }
    }],
    duration_seconds: durationSeconds
  });
  
  // Re-throw or handle error
  throw error;
}
```

### Querying Recent Failures
```javascript
// Get failed syncs from last 7 days
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const recentFailures = $('Data Table').all().filter(log => 
  log.status === 'failed' && new Date(log.sync_date) > sevenDaysAgo
);
```

### Calculating Success Rate
```javascript
// Get success rate for a vendor
const vendorId = 'vendor_001';
const logs = $('Data Table').all().filter(log => log.vendor_id === vendorId);
const successCount = logs.filter(log => log.status === 'success').length;
const successRate = (successCount / logs.length) * 100;
console.log(`Success rate for ${vendorId}: ${successRate.toFixed(2)}%`);
```

## Monitoring and Alerting

### Key Metrics to Track

#### Success Rate
```javascript
// Calculate overall success rate
const allLogs = $('Data Table').all();
const successLogs = allLogs.filter(log => log.status === 'success');
const successRate = (successLogs.length / allLogs.length) * 100;
// Alert if success rate < 95%
```

#### Average Sync Duration
```javascript
// Calculate average sync duration by tier
const tier1Logs = $('Data Table').all().filter(log => {
  const vendor = $('vendor_config').get({ vendor_id: log.vendor_id });
  return vendor.tier_used === 1;
});
const avgDuration = tier1Logs.reduce((sum, log) => sum + log.duration_seconds, 0) / tier1Logs.length;
// Alert if duration increases significantly
```

#### Failed Syncs
```javascript
// Get failed syncs from last 24 hours
const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);
const recentFailures = $('Data Table').all().filter(log => 
  log.status === 'failed' && new Date(log.sync_date) > oneDayAgo
);
// Send alert if any failures
```

### Alert Conditions

**Critical Alerts** (immediate notification):
- Any sync with `status: 'failed'`
- Success rate drops below 90%
- Same vendor fails 3 times in a row
- Sync duration exceeds 10 minutes

**Warning Alerts** (daily digest):
- Partial syncs with errors
- Success rate between 90-95%
- Sync duration increases by >50%
- Products updated/added = 0 (possible empty catalog)

## Reporting

### Daily Sync Summary Report
```javascript
// Generate daily summary
const today = new Date();
today.setHours(0, 0, 0, 0);

const todayLogs = $('Data Table').all().filter(log => 
  new Date(log.sync_date) >= today
);

const summary = {
  total_syncs: todayLogs.length,
  successful: todayLogs.filter(l => l.status === 'success').length,
  partial: todayLogs.filter(l => l.status === 'partial').length,
  failed: todayLogs.filter(l => l.status === 'failed').length,
  total_products_updated: todayLogs.reduce((sum, l) => sum + l.products_updated, 0),
  total_products_added: todayLogs.reduce((sum, l) => sum + l.products_added, 0),
  avg_duration: todayLogs.reduce((sum, l) => sum + l.duration_seconds, 0) / todayLogs.length
};

// Email or Slack this summary
```

### Weekly Performance Report
```javascript
// Generate weekly performance report
const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
const weekLogs = $('Data Table').all().filter(log => 
  new Date(log.sync_date) > oneWeekAgo
);

// Group by vendor
const vendorStats = {};
weekLogs.forEach(log => {
  if (!vendorStats[log.vendor_id]) {
    vendorStats[log.vendor_id] = {
      total: 0,
      success: 0,
      failed: 0,
      products_updated: 0,
      products_added: 0
    };
  }
  vendorStats[log.vendor_id].total++;
  if (log.status === 'success') vendorStats[log.vendor_id].success++;
  if (log.status === 'failed') vendorStats[log.vendor_id].failed++;
  vendorStats[log.vendor_id].products_updated += log.products_updated;
  vendorStats[log.vendor_id].products_added += log.products_added;
});

// Identify problematic vendors
const problematicVendors = Object.entries(vendorStats)
  .filter(([id, stats]) => stats.failed / stats.total > 0.1)
  .map(([id, stats]) => ({ vendor_id: id, ...stats }));
```

## Cost Analysis

### Tier 3 Usage Tracking
```javascript
// Calculate Tier 3 costs from sync logs
const tier3Logs = $('Data Table').all().filter(log => {
  const vendor = $('vendor_config').get({ vendor_id: log.vendor_id });
  return vendor.tier_used === 3;
});

const totalProducts = tier3Logs.reduce((sum, log) => 
  sum + log.products_updated + log.products_added, 0
);

// Estimate costs (assuming 1 product = 1 Zyte request + 1 Gemini request)
const zyteCost = totalProducts * 0.00013; // $0.00013 per request
const geminiCost = totalProducts * 0.002; // $0.002 per request
const totalCost = zyteCost + geminiCost;

console.log(`Tier 3 costs: $${totalCost.toFixed(2)}`);
```

## Troubleshooting

### Common Issues

**Issue**: Sync logs show repeated failures for same vendor
- Review `errors` field for specific error messages
- Check vendor_config for correct credentials
- Verify vendor's data source is still accessible
- Test vendor connection manually
- Check if vendor changed their API or data structure

**Issue**: Sync duration increasing over time
- Check if vendor's catalog is growing
- Look for network latency issues
- Verify API rate limits aren't being hit
- Consider optimizing sync logic
- Check if Tier 3 AI extraction is slowing down

**Issue**: Products updated/added = 0 for successful sync
- Verify vendor actually has products
- Check if data source format changed
- Review sync logic for parsing errors
- Manually inspect vendor's data source
- Check if SKU matching logic is working

**Issue**: Partial syncs with many errors
- Review specific errors in `errors` field
- Check if rate limits are being exceeded
- Verify data quality from vendor
- Consider implementing retry logic
- May need to adjust sync batch size

## Data Retention

### Retention Policy Recommendations
- **Keep all logs for 90 days** for detailed analysis
- **Archive logs older than 90 days** to separate table
- **Keep summary statistics indefinitely** for long-term trends
- **Delete archived logs after 1 year** unless needed for compliance

### Archival Process
```javascript
// Archive logs older than 90 days
const ninetyDaysAgo = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000);
const oldLogs = $('Data Table').all().filter(log => 
  new Date(log.sync_date) < ninetyDaysAgo
);

// Move to archive table or export to file
// Then delete from sync_log table
```

## Performance Optimization

### Batch Logging
- Log multiple sync operations in a single batch insert
- Reduces database overhead
- Improves sync performance

### Async Logging
- Don't block sync operations waiting for log writes
- Use fire-and-forget pattern for non-critical logs
- Ensure critical errors are logged synchronously

### Log Rotation
- Implement automatic log rotation
- Archive old logs to reduce table size
- Maintain indexes for fast queries

## Security Considerations

### Sensitive Data
- Don't log sensitive credentials in `errors` field
- Sanitize error messages before logging
- Redact API keys, tokens, passwords
- Be careful with PII in error details

### Access Control
- Limit access to sync_log table
- Use read-only access for reporting
- Restrict delete operations
- Audit log access

## Change Log

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-19 | Initial schema definition |

---

**Related Documentation:**
- [Vendor Config Schema](./vendor_config.md)
- [Product Data Schema](./product_data.md)
- [Monitoring Guide](../guides/monitoring.md)
- [Setup Guide](../guides/SETUP_GUIDE.md)

