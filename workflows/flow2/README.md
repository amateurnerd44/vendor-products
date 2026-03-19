# Flow 2: Order Fulfillment Workflows

This directory contains all n8n workflows for Phase 6: Order Fulfillment.

## Workflows

### 1. MarketTime Email Trigger (`markettime_trigger.json`)
- **Purpose**: Monitor IMAP inbox for MarketTime order emails
- **Trigger**: IMAP check every 5 minutes
- **Output**: Standardized order object

### 2. Shopify Order Webhook (`shopify_trigger.json`)
- **Purpose**: Receive Shopify order webhooks
- **Trigger**: Webhook endpoint `/webhook/shopify-orders`
- **Security**: HMAC-SHA256 verification
- **Output**: Standardized order object

### 3. SKU Lookup (`sku_lookup.json`)
- **Purpose**: Query product_data table for SKUs
- **Input**: Array of SKUs
- **Output**: Found products + missing SKUs list
- **Performance**: Tracks cache hit rate (target >90%)

### 4. On-Demand Scraper (`on_demand_scraper.json`)
- **Purpose**: Scrape missing SKUs in real-time
- **Input**: Array of missing SKUs
- **Processing**: Routes to Tier 1/2/3 scrapers based on vendor
- **Output**: Scraped products + failed SKUs
- **Timeout**: 30s for Tier 1/2, 60s for Tier 3

### 5. Asset Compiler (`asset_compiler.json`)
- **Purpose**: Compile digital asset packages
- **Input**: Order + product data
- **Output**: Formatted asset package ready for delivery

### 6. Customer Delivery (`customer_delivery.json`)
- **Purpose**: Deliver assets to customers
- **Method**: Email (configurable)
- **Input**: Asset package
- **Output**: Delivery status

## Workflow Dependencies

```
MarketTime Trigger ──┐
                     ├──> SKU Lookup ──> On-Demand Scraper ──> Asset Compiler ──> Customer Delivery
Shopify Trigger ─────┘
```

## Environment Variables Required

```bash
# Email
ADMIN_EMAIL=admin@example.com
DELIVERY_EMAIL=noreply@example.com

# Shopify
SHOPIFY_WEBHOOK_SECRET=your_secret

# n8n
N8N_WEBHOOK_BASE_URL=https://your-n8n-instance.com

# Delivery
DELIVERY_METHOD=email  # Options: email, drive, dropbox, download_link
```

## Import Instructions

1. Open n8n Cloud
2. Go to Workflows
3. Click "Import from File"
4. Select each JSON file in this directory
5. Configure credentials (IMAP, SMTP)
6. Set environment variables
7. Activate workflows

## Testing

See [FLOW2_ORDER_FULFILLMENT.md](../../docs/workflows/FLOW2_ORDER_FULFILLMENT.md) for detailed testing scenarios.

## Performance Targets

- **Cache Hit Rate**: >90%
- **Cached SKU Fulfillment**: <10 seconds
- **On-Demand Scraping**: <60 seconds
- **Email Delivery**: <5 seconds

## Related Documentation

- [Flow 2 Documentation](../../docs/workflows/FLOW2_ORDER_FULFILLMENT.md)
- [Product Data Schema](../../docs/schemas/product_data.md)
- [Implementation Plan](../../IMPLEMENTATION_PLAN.md)

