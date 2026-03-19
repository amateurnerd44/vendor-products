# Phase 1 Agent Prompt - Foundation & Infrastructure Setup

---

## 🎯 Your Mission

You are tasked with completing **Phase 1: Foundation & Infrastructure** for the Vendor Product Data Collection System. This is a greenfield project that will automate the collection of product data (images, videos, marketing copy) from 70+ vendors and deliver digital asset packages to customers when they place orders.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

### What This System Does

This system has two main workflows:

1. **Flow 1 - Weekly Product Catalog Sync:** Runs weekly to collect product data from all vendors and store it in n8n Data Tables
2. **Flow 2 - Order Fulfillment:** Triggered by customer orders (MarketTime email or Shopify webhook), retrieves product data and delivers digital assets

### Tech Stack

- **Orchestration:** n8n Cloud (workflow automation platform)
- **Web Scraping:** Zyte API (pay-per-success, $0.00013/request)
- **AI Extraction:** Google AI Studio Gemini API (usage-based)
- **Data Storage:** n8n Data Tables (built-in database)
- **Triggers:** Email parsing (MarketTime) and Shopify webhooks

### Tiered Data Collection Strategy

- **Tier 1:** Direct integrations (Google Drive, Dropbox, Google Sheets) - FREE
- **Tier 2:** Shopify public JSON endpoints - FREE  
- **Tier 3:** Zyte API + Gemini AI for custom websites - USAGE-BASED (~$222/year budget)

### Key Constraints

- ✅ Usage-based pricing only (no monthly subscriptions)
- ✅ Target cost: ~$222/year for scraping and AI services
- ✅ Fast order fulfillment: <10 seconds for cached SKUs
- ✅ Sync success rate: >95%

---

## 🎯 Phase 1 Deliverables (Your Tasks)

You are responsible for completing the **foundation and infrastructure** setup. This phase has **no dependencies** and must be completed before Phases 2-4 can begin.

### Task 1: Create n8n Data Tables Documentation

Document three data table schemas that will be created in n8n Cloud:

#### Table 1: `vendor_config`

| Column Name | Type | Constraints | Description |
|------------|------|-------------|-------------|
| vendor_id | Text | PRIMARY KEY | Unique vendor identifier (e.g., "vendor_001") |
| vendor_name | Text | NOT NULL | Human-readable vendor name |
| data_source_type | Text | NOT NULL | One of: 'drive', 'dropbox', 'sheets', 'shopify', 'website' |
| source_url | Text | NULLABLE | URL or path to data source |
| credentials | JSON | NULLABLE, ENCRYPTED | Encrypted credentials for data source access |
| last_sync_date | DateTime | NULLABLE | Last successful sync timestamp |
| sync_status | Text | DEFAULT 'pending' | One of: 'success', 'failed', 'pending' |
| tier_used | Number | NOT NULL | Tier classification: 1, 2, or 3 |
| notes | Long Text | NULLABLE | Additional notes or instructions |

#### Table 2: `product_data`

| Column Name | Type | Constraints | Description |
|------------|------|-------------|-------------|
| sku | Text | PRIMARY KEY | Product SKU (Stock Keeping Unit) |
| vendor_id | Text | NOT NULL | Foreign key to vendor_config.vendor_id |
| product_name | Text | NOT NULL | Product name/title |
| image_urls | JSON | NULLABLE | Array of image URLs |
| video_urls | JSON | NULLABLE | Array of video URLs |
| marketing_copy | Long Text | NULLABLE | Product description/marketing text |
| metadata | JSON | NULLABLE | Additional product metadata (price, variants, etc.) |
| last_updated | DateTime | NOT NULL | Last update timestamp |
| source_tier | Number | NOT NULL | Tier used to collect data: 1, 2, or 3 |

#### Table 3: `sync_log`

| Column Name | Type | Constraints | Description |
|------------|------|-------------|-------------|
| log_id | Number | PRIMARY KEY, AUTO_INCREMENT | Unique log entry ID |
| sync_date | DateTime | NOT NULL | Sync operation timestamp |
| vendor_id | Text | NOT NULL | Vendor being synced |
| status | Text | NOT NULL | One of: 'success', 'partial', 'failed' |
| products_updated | Number | DEFAULT 0 | Number of products updated |
| products_added | Number | DEFAULT 0 | Number of new products added |
| errors | JSON | NULLABLE | Array of error messages if any |
| duration_seconds | Number | NULLABLE | Sync duration in seconds |

### Task 2: Create Vendor Configuration Templates

Create CSV and JSON templates with 10 sample vendors across all tiers:

**Sample Vendors:**
- vendor_001: Acme Products (Google Drive, Tier 1)
- vendor_002: Beta Supplies (Shopify, Tier 2)
- vendor_003: Gamma Industries (Google Sheets, Tier 1)
- vendor_004: Delta Manufacturing (Dropbox, Tier 1)
- vendor_005: Epsilon Wholesale (Website, Tier 3)
- vendor_006: Zeta Distributors (Shopify, Tier 2)
- vendor_007: Eta Imports (Google Drive, Tier 1)
- vendor_008: Theta Goods (Website, Tier 3)
- vendor_009: Iota Merchandise (Google Sheets, Tier 1)
- vendor_010: Kappa Supplies Co (Dropbox, Tier 1)

### Task 3: Create Environment Variables Documentation

Create `.env.example` file documenting all required environment variables:

**Required Sections:**
- n8n Configuration (host, API key)
- Zyte API (key, URL)
- Google AI Studio (API key, model, URL)
- Google Drive OAuth2 (client ID, secret, redirect URI)
- Dropbox (app key, secret, access token)
- Google Sheets (service account email, private key)
- Email IMAP (host, port, username, password)
- Shopify Webhook (secret)
- Notifications (email, Slack webhook)
- Customer Delivery (method, SMTP settings, Drive/Dropbox paths)
- System Configuration (sync schedule, retries, rate limits, logging)
- Cost Monitoring (alert thresholds)
- Security (encryption key, API timeout)

### Task 4: Create Setup Guide

Create comprehensive `SETUP_GUIDE.md` with:
- Prerequisites checklist
- Step-by-step n8n Data Tables creation
- API credentials configuration for all 7 integrations
- Email and webhook setup
- Vendor data import procedures
- Testing procedures for each tier
- Security checklist
- Monitoring and backup setup
- Troubleshooting guide

### Task 5: Create Repository Structure

Organize files into:
```
vendor-products/
├── .env.example
├── docs/
│   ├── schemas/
│   │   └── n8n-data-tables.md
│   └── guides/
│       └── SETUP_GUIDE.md
└── templates/
    ├── vendor-config-template.csv
    └── vendor-config-template.json
```

---

## ✅ Completion Checklist

- [ ] n8n Data Tables schemas documented (3 tables)
- [ ] Vendor configuration templates created (CSV + JSON)
- [ ] 10 sample vendors defined across all tiers
- [ ] Environment variables documented (30+ variables)
- [ ] Setup guide created with step-by-step instructions
- [ ] Repository structure organized
- [ ] All files committed to repository

---

## 📊 Success Metrics

- **Time to Complete:** 2-4 hours
- **Tables Documented:** 3/3
- **Sample Vendors:** 10/10
- **Documentation Files:** 3+ files
- **Credential Types:** 7/7

---

## 🚀 Next Steps After Phase 1

Once complete, Phases 2-4 can begin in parallel:
- **Phase 2:** Tier 1 Scrapers (Drive, Dropbox, Sheets)
- **Phase 3:** Tier 2 Scrapers (Shopify)
- **Phase 4:** Tier 3 Scrapers (Zyte + AI)

---

**Phase:** 1 of 8  
**Dependencies:** None  
**Enables:** Phases 2, 3, 4  
**Estimated Time:** 2-4 hours

