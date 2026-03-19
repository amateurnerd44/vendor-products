# Operational Runbook
**Version:** 1.0 (Pre-Deployment)  
**Last Updated:** March 19, 2026

---

## Overview

This runbook provides operational procedures for maintaining the Vendor Product Data Collection System.

### Audience
- Operations team
- On-call engineers
- System administrators

### Scope
- Daily monitoring tasks
- Weekly sync verification
- Error escalation procedures
- Vendor support procedures
- Performance optimization
- Cost monitoring

---

## Daily Monitoring Tasks

### Morning Checklist (9:00 AM)

**1. Check System Health**
```
□ Open n8n Cloud dashboard
□ Verify all workflows are active
□ Check for any failed executions in last 24 hours
□ Review Slack #vendor-sync-alerts channel
□ Check email for system alerts
```

**2. Review Order Fulfillment**
```sql
-- Check yesterday's order fulfillment
SELECT 
  COUNT(*) as total_orders,
  SUM(CASE WHEN fulfillment_status = 'success' THEN 1 ELSE 0 END) as successful,
  AVG(processing_time_seconds) as avg_time
FROM order_fulfillment_log
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR);
```

**Expected Results:**
- Success rate: >98%
- Average time: <15 seconds
- If below targets: Investigate failures

**3. Check for Customer Issues**
```
□ Review customer support tickets
□ Check for delivery failures
□ Verify no missing SKU complaints
□ Respond to urgent issues within 2 hours
```

### Afternoon Checklist (2:00 PM)

**1. Monitor API Usage**
```
□ Check Zyte API usage (should be minimal on non-sync days)
□ Check Gemini API usage
□ Verify costs are within daily budget (~$0.80/day)
□ Alert if usage is abnormal
```

**2. Review Workflow Performance**
```
□ Check n8n execution times
□ Identify any slow-running workflows
□ Review error logs
□ Document any recurring issues
```

### End-of-Day Checklist (5:00 PM)

**1. Daily Summary**
```
□ Total orders fulfilled: ___
□ Success rate: ___%
□ Average fulfillment time: ___ seconds
□ API costs today: $___
□ Issues escalated: ___
□ Issues resolved: ___
```

**2. Prepare for Next Day**
```
□ Review tomorrow's schedule
□ Check if Sunday (weekly sync day)
□ Ensure on-call coverage
□ Document any pending issues
```

---

## Weekly Sync Verification

### Pre-Sync Checklist (Saturday Evening)

**1. System Preparation**
```
□ Verify all workflows are active
□ Check API credentials are valid
□ Ensure sufficient API quotas
□ Review vendor_config for any disabled vendors
□ Check disk space for data storage
```

**2. Notification Setup**
```
□ Ensure Slack notifications are working
□ Verify email alerts are configured
□ Confirm on-call engineer is available
□ Set up monitoring dashboard
```

### During Sync (Sunday 2:00 AM - 10:00 AM)

**1. Real-Time Monitoring**
```
□ Monitor n8n workflow execution
□ Watch for error notifications
□ Check sync progress every hour
□ Log any immediate issues
```

**2. Performance Tracking**
```sql
-- Check sync progress
SELECT 
  COUNT(*) as vendors_synced,
  SUM(products_synced) as total_products,
  AVG(processing_time_seconds) as avg_time
FROM sync_log
WHERE sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR);
```

### Post-Sync Verification (Sunday 10:00 AM)

**1. Success Rate Check**
```sql
-- Verify sync success rate
SELECT 
  COUNT(*) as total_vendors,
  SUM(CASE WHEN sync_status = 'success' THEN 1 ELSE 0 END) as successful,
  ROUND(100.0 * SUM(CASE WHEN sync_status = 'success' THEN 1 ELSE 0 END) / COUNT(*), 2) as success_rate
FROM vendor_config
WHERE last_sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR);
```

**Expected:** Success rate ≥95% (≤3-4 vendors failing)

**2. Product Count Validation**
```sql
-- Check product counts per vendor
SELECT 
  vendor_id,
  vendor_name,
  products_synced,
  last_sync_date
FROM vendor_config
WHERE last_sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR)
ORDER BY products_synced ASC;
```

**Action:** Investigate vendors with 0 products or significant drops

**3. Error Analysis**
```sql
-- Review sync errors
SELECT 
  vendor_id,
  error_message,
  tier_used,
  processing_time_seconds
FROM sync_log
WHERE sync_status = 'failed'
AND sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR);
```

**4. Tier Performance**
```sql
-- Analyze tier performance
SELECT 
  tier_used,
  COUNT(*) as vendor_count,
  SUM(CASE WHEN sync_status = 'success' THEN 1 ELSE 0 END) as successful,
  AVG(processing_time_seconds) as avg_time
FROM sync_log
WHERE sync_date >= DATE_SUB(NOW(), INTERVAL 12 HOUR)
GROUP BY tier_used;
```

**Expected:**
- Tier 1: 100% success
- Tier 2: ≥98% success
- Tier 3: ≥90% success

**5. Cost Verification**
```
□ Check Zyte API usage for the week
□ Check Gemini API usage for the week
□ Calculate total cost
□ Verify cost is within budget (~$4.50/week)
□ Document any cost anomalies
```

### Weekly Report (Sunday 12:00 PM)

**Generate and send weekly report:**
```
Subject: Weekly Vendor Sync Report - [Date]

Summary:
- Total vendors: 70
- Successful syncs: __ (___%)
- Failed syncs: __
- Total products synced: ____
- Sync duration: __ hours
- Total cost: $__.__

Tier Performance:
- Tier 1: __ vendors, ___% success
- Tier 2: __ vendors, ___% success
- Tier 3: __ vendors, ___% success

Issues:
- [List any vendor failures]
- [List any performance issues]
- [List any cost anomalies]

Actions Taken:
- [List any fixes applied]
- [List any vendors disabled]

Next Steps:
- [List any follow-up actions needed]
```

---

## Error Escalation Procedures

### Severity Levels

**P0 - Critical (Immediate Response)**
- System completely down
- All order fulfillment failing
- Data loss or corruption
- Security breach

**P1 - High (Response within 1 hour)**
- >10% of vendors failing sync
- Order fulfillment success rate <90%
- API credentials expired
- Cost exceeding budget by >50%

**P2 - Medium (Response within 4 hours)**
- Individual vendor consistently failing
- Order fulfillment slow (>60 seconds)
- API rate limiting issues
- Minor data quality issues

**P3 - Low (Response within 24 hours)**
- Single vendor failure
- Minor performance degradation
- Documentation updates needed
- Feature requests

### Escalation Workflow

**Step 1: Identify Issue**
```
1. Determine severity level
2. Gather relevant logs and data
3. Document symptoms and impact
4. Check if issue is known/recurring
```

**Step 2: Initial Response**
```
P0: Page on-call engineer immediately
P1: Notify on-call engineer via Slack
P2: Create ticket and assign to team
P3: Add to backlog for next sprint
```

**Step 3: Investigation**
```
1. Review error logs in n8n
2. Check API status pages
3. Verify credentials and configurations
4. Test affected workflows manually
5. Identify root cause
```

**Step 4: Resolution**
```
1. Apply fix (if known solution)
2. Test fix in staging (if available)
3. Deploy fix to production
4. Verify issue is resolved
5. Monitor for recurrence
```

**Step 5: Post-Mortem (P0/P1 only)**
```
1. Document timeline of events
2. Identify root cause
3. List contributing factors
4. Propose preventive measures
5. Update runbook with learnings
```

### Common Issues and Solutions

**Issue: Vendor Sync Failing**
```
Symptoms: sync_status = 'failed' in vendor_config
Severity: P2 (P1 if >10% vendors)

Investigation:
1. Check error_message in sync_log
2. Verify credentials are valid
3. Test data source manually
4. Check if vendor website changed

Solutions:
- Refresh OAuth tokens
- Update source_url
- Enable Tier 3 fallback
- Contact vendor for data access
```

**Issue: Order Fulfillment Slow**
```
Symptoms: processing_time_seconds >60
Severity: P2

Investigation:
1. Check cache hit rate
2. Review on-demand scraping frequency
3. Check API response times
4. Verify network connectivity

Solutions:
- Improve cache hit rate (run sync more frequently)
- Optimize scraping logic
- Add caching for frequently ordered SKUs
- Scale up n8n resources
```

**Issue: High API Costs**
```
Symptoms: Monthly cost >$25
Severity: P1

Investigation:
1. Check Zyte API usage
2. Check Gemini API usage
3. Identify high-usage vendors
4. Review scraping frequency

Solutions:
- Move vendors to lower tiers
- Reduce sync frequency
- Optimize AI extraction prompts
- Negotiate with vendors for data exports
```

---

## Vendor Support Procedures

### New Vendor Onboarding

**1. Initial Contact**
```
□ Receive vendor information
□ Determine data source type
□ Assess tier assignment
□ Estimate product count
□ Set expectations for timeline
```

**2. Data Source Setup**
```
Tier 1:
□ Request Drive/Dropbox/Sheets access
□ Verify data format
□ Test data parsing
□ Configure credentials

Tier 2:
□ Verify Shopify store URL
□ Test products.json endpoint
□ Check product count
□ Configure in vendor_config

Tier 3:
□ Verify website accessibility
□ Test Zyte rendering
□ Test AI extraction
□ Estimate costs
```

**3. Testing**
```
□ Run manual sync test
□ Verify product data quality
□ Check all 35 fields populated
□ Validate images/videos accessible
□ Confirm pricing data accurate
```

**4. Production Deployment**
```
□ Add to vendor_config table
□ Set sync_status = 'pending'
□ Wait for next weekly sync
□ Monitor first sync closely
□ Verify success
□ Update vendor with results
```

### Vendor Issue Resolution

**Issue: Vendor Data Quality Poor**
```
1. Contact vendor about missing/incorrect data
2. Request data format improvements
3. If Tier 1: Ask for more complete exports
4. If Tier 2: Check if metafields available
5. If Tier 3: Improve AI extraction prompts
6. Consider tier upgrade/downgrade
```

**Issue: Vendor Website Changed**
```
1. Verify new website structure
2. Update source_url if needed
3. If Tier 3: Update Zyte selectors
4. Test scraping with new structure
5. Deploy updated configuration
6. Monitor next sync
```

**Issue: Vendor Credentials Expired**
```
1. Contact vendor for new credentials
2. Update credentials in n8n
3. Test authentication
4. Run manual sync test
5. Update vendor_config
6. Monitor next sync
```

---

## Performance Optimization

### Database Optimization

**1. Index Management**
```sql
-- Ensure critical indexes exist
CREATE INDEX idx_sku ON product_data(SKU);
CREATE INDEX idx_vendor_id ON product_data(vendor_id);
CREATE INDEX idx_sync_date ON sync_log(sync_date);
```

**2. Data Cleanup**
```sql
-- Delete old sync logs (>90 days)
DELETE FROM sync_log
WHERE sync_date < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- Archive old order fulfillment logs
-- (Move to separate archive table)
```

### Workflow Optimization

**1. Parallel Processing**
```
- Increase concurrent vendor processing
- Current: 5 concurrent vendors
- Recommended: 10 concurrent (if no rate limit issues)
- Monitor API rate limits
```

**2. Caching Strategy**
```
- Cache vendor credentials in memory
- Cache MarketTime items list
- Reuse HTTP connections
- Implement smart cache invalidation
```

**3. Batch Operations**
```
- Batch product upserts (100 products at a time)
- Batch image downloads
- Batch API requests where possible
```

### Cost Optimization

**1. Tier Migration**
```
- Review Tier 3 vendors quarterly
- Identify candidates for Tier 1/2
- Negotiate data exports with vendors
- Target: Move 5 vendors per quarter
```

**2. Sync Frequency Optimization**
```
- Analyze product change frequency per vendor
- Reduce sync frequency for stable catalogs
- Implement change detection
- Target: 20% reduction in Tier 3 scraping
```

**3. AI Prompt Optimization**
```
- Reduce token usage in Gemini prompts
- Extract only relevant HTML sections
- Optimize prompt structure
- Target: 30% reduction in token usage
```

---

## Cost Monitoring

### Daily Cost Tracking

**Check API Usage:**
```
□ Zyte API: __ requests today
□ Gemini API: __ tokens today
□ Estimated cost: $__.__
□ Budget remaining: $__.__
```

### Weekly Cost Report

```
Week of [Date]:

Zyte API:
- Requests: ____
- Cost: $__.__

Gemini API:
- Tokens: ____
- Cost: $__.__

Total Cost: $__.__
Budget: $4.50/week
Variance: $__.__ (___%)

Trends:
- [Increasing/Decreasing/Stable]
- [Any anomalies]

Actions:
- [Any cost optimization steps taken]
```

### Monthly Cost Review

**Generate monthly report:**
```
Month of [Month]:

Total Cost: $__.__
Budget: $19/month
Variance: $__.__ (___%)

Breakdown by Tier:
- Tier 1: $0 (30 vendors)
- Tier 2: $0 (20 vendors)
- Tier 3: $__.__ (20 vendors)

Cost per Vendor (Tier 3):
- Average: $__.__
- Highest: $__.__ (vendor: ___)
- Lowest: $__.__ (vendor: ___)

Recommendations:
- [List cost optimization opportunities]
- [List vendors to migrate to lower tiers]
```

---

## Maintenance Windows

### Scheduled Maintenance

**Monthly Maintenance (First Saturday of Month)**
```
Time: 10:00 PM - 12:00 AM (2 hours)

Tasks:
□ Update n8n workflows if needed
□ Rotate API credentials
□ Clean up old logs
□ Review and update documentation
□ Test disaster recovery procedures
□ Update monitoring dashboards
```

**Quarterly Maintenance (First Saturday of Quarter)**
```
Time: 10:00 PM - 2:00 AM (4 hours)

Tasks:
□ All monthly tasks
□ Review vendor tier assignments
□ Audit API usage and costs
□ Performance optimization review
□ Security audit
□ Update runbook and documentation
□ Team training on new features
```

---

## Disaster Recovery

### Backup Procedures

**Daily Backups:**
```
□ n8n workflow exports
□ vendor_config table export
□ product_data table export (incremental)
□ Credentials backup (encrypted)
```

**Weekly Backups:**
```
□ Full database export
□ Configuration files
□ Documentation
□ Monitoring dashboards
```

### Recovery Procedures

**Scenario 1: n8n Workflow Corruption**
```
1. Stop affected workflow
2. Restore from latest backup
3. Test workflow in staging
4. Deploy to production
5. Verify functionality
6. Monitor for issues
```

**Scenario 2: Database Corruption**
```
1. Identify affected tables
2. Restore from latest backup
3. Verify data integrity
4. Run sync to update missing data
5. Monitor for issues
```

**Scenario 3: Complete System Failure**
```
1. Assess scope of failure
2. Restore n8n workflows from backup
3. Restore database from backup
4. Restore credentials
5. Test all workflows
6. Run full sync
7. Verify order fulfillment
8. Monitor closely for 48 hours
```

---

## Contact Information

### On-Call Rotation
- **Primary:** [Name] - [Phone] - [Email]
- **Secondary:** [Name] - [Phone] - [Email]
- **Escalation:** [Manager] - [Phone] - [Email]

### Vendor Contacts
- See `vendor_config` table for vendor-specific contacts
- General vendor support: vendor-ops@[domain].com

### External Support
- **n8n Support:** support@n8n.io
- **Zyte Support:** support@zyte.com
- **Google Cloud Support:** [Support Portal]

---

**Document Version:** 1.0 (Pre-Deployment)  
**Next Review:** Post-deployment validation  
**Maintained By:** Operations Team  
**Last Updated:** March 19, 2026

