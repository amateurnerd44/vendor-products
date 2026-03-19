# Tier 3 Workflows

This directory contains n8n workflows for scraping custom vendor websites using Zyte API + Gemini AI.

## Workflows

### Core Workflows

1. **`zyte_fetcher.json`** - Fetches and renders web pages using Zyte API
   - JavaScript rendering enabled
   - Automatic retry with exponential backoff
   - Cost: $0.00013 per request

2. **`gemini_extractor.json`** - Extracts structured data using Gemini AI
   - Supports product, catalog, and pagination extraction
   - JSON validation and error handling
   - Cost: ~$0.002 per request

3. **`error_handler.json`** - Intelligent error handling
   - Retry logic with exponential backoff
   - Fallback to simpler prompts
   - Admin alerts for critical errors

4. **`cost_tracker.json`** - Monitors API usage and costs
   - Tracks Zyte and Gemini costs
   - Budget alerts (weekly/annual)
   - Cost statistics and projections

### Test Workflow

5. **`test_tier3.json`** - End-to-end testing workflow
   - Tests all Tier 3 workflows
   - Validates extraction accuracy
   - Tracks test costs

## Quick Start

### 1. Import Workflows to n8n

1. Open n8n Cloud
2. Go to Workflows → Import from File
3. Import each `.json` file from this directory
4. Activate the workflows

### 2. Configure Credentials

Set up the following credentials in n8n:

- **Zyte API** - Your Zyte API key
- **Google AI Studio** - Your Gemini API key
- **Email (SMTP)** - For error alerts

### 3. Test the Workflows

1. Open `test_tier3.json` workflow
2. Update the test URL in "Set Test Data" node
3. Click "Execute Workflow"
4. Check the output for success/failure

## Usage

### Calling from Other Workflows

```javascript
// Example: Call Zyte Fetcher
const html = await executeWorkflow('Tier 3: Zyte Fetcher', {
  url: 'https://vendor-website.com/product/abc-123',
  maxRetries: 3
});

if (html.success) {
  // Call Gemini Extractor
  const data = await executeWorkflow('Tier 3: Gemini Extractor', {
    html: html.html,
    url: html.url,
    pageType: 'product'
  });
  
  if (data.success) {
    // Use extracted product data
    console.log(data.productData);
    
    // Track costs
    await executeWorkflow('Tier 3: Cost Tracker', {
      service: 'zyte',
      cost: html.cost,
      vendorId: 'vendor-001'
    });
  }
}
```

## Configuration

### AI Prompts

Edit `config/ai_prompts.json` to customize extraction prompts:

- `product` - Single product page extraction
- `catalog` - Multiple products extraction
- `paginated` - Pagination detection
- `simple_fallback` - Simplified extraction

### Cost Budgets

Default budgets (edit in `cost_tracker.json`):

- Weekly: $4.26
- Annual: $222

### Error Handling

Configure retry and fallback behavior in `error_handler.json`:

- Max retries: 3
- Backoff: 5s, 10s, 20s
- Alert email: Update recipient in workflow

## Monitoring

### Cost Tracking

View cost statistics in the `sync_log` table:

```sql
SELECT 
  date,
  SUM(CASE WHEN service = 'zyte' THEN cost ELSE 0 END) as zyte_cost,
  SUM(CASE WHEN service = 'gemini' THEN cost ELSE 0 END) as gemini_cost,
  SUM(cost) as total_cost
FROM sync_log
WHERE date >= DATE('now', '-7 days')
GROUP BY date;
```

### Error Logs

Check error logs in `sync_log` table:

```sql
SELECT *
FROM sync_log
WHERE status = 'failed'
ORDER BY timestamp DESC
LIMIT 10;
```

## Troubleshooting

### Common Issues

**Zyte API Timeouts**
- Increase timeout in `zyte_fetcher.json`
- Check vendor website speed
- Verify Zyte API status

**Gemini Extraction Failures**
- Review AI prompts in `config/ai_prompts.json`
- Check HTML structure of vendor site
- Use fallback prompt

**Cost Overruns**
- Review vendor list
- Increase sync interval
- Optimize prompts

## Documentation

See `docs/workflows/TIER3_ZYTE_AI.md` for comprehensive documentation.

## Support

For issues or questions:
- Check error logs in `sync_log` table
- Review cost tracker reports
- Update AI prompts if extraction quality degrades

