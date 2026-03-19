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

### Task 1: n8n Data Tables Setup

Create three data tables in n8n Cloud with the following schemas:

#### Table 1: `vendor_config`

Stores configuration for all vendor data sources.

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

Stores product information collected from vendors.

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

#### Table 3: `sync_log` (Optional but Recommended)

Tracks sync operations for monitoring and debugging.

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

**How to Create Tables in n8n:**

1. Log in to n8n Cloud at your instance URL
2. Navigate to **Data** → **Tables**
3. Click **Create Table**
4. Define columns according to schemas above
5. Set primary keys and constraints
6. Enable encryption for the `credentials` field in `vendor_config`

**Deliverable:** Document the table creation process with screenshots or confirmation that all three tables exist and are properly configured.

---

### Task 2: API Credentials Configuration

Set up credentials in n8n for all required integrations. You'll need to create credential entries for:

#### Required Credentials

1. **Zyte API**
   - Type: HTTP Header Auth
   - Name: `Zyte API`
   - Header Name: `Authorization`
   - Header Value: `Bearer [API_KEY]`
   - Note: User will provide API key separately

2. **Google AI Studio (Gemini)**
   - Type: HTTP Header Auth
   - Name: `Google AI Studio`
   - Header Name: `x-goog-api-key`
   - Header Value: `[API_KEY]`
   - Note: User will provide API key separately

3. **Google Drive OAuth2**
   - Type: Google Drive OAuth2 API
   - Name: `Google Drive - Vendor Access`
   - Note: User will complete OAuth flow

4. **Dropbox OAuth2**
   - Type: Dropbox OAuth2 API
   - Name: `Dropbox - Vendor Access`
   - Note: User will complete OAuth flow

5. **Google Sheets Service Account**
   - Type: Google Sheets Service Account
   - Name: `Google Sheets - Vendor Access`
   - Note: User will upload service account JSON key

6. **Email IMAP (MarketTime)**
   - Type: IMAP
   - Name: `MarketTime Email`
   - Host: `imap.gmail.com`
   - Port: `993`
   - SSL: Enabled
   - Note: User will provide email and app password

7. **SMTP (Customer Delivery)**
   - Type: SMTP
   - Name: `Customer Delivery Email`
   - Host: `smtp.gmail.com`
   - Port: `587`
   - Note: User will provide SMTP credentials

**Deliverable:** Create a credential setup checklist document that guides the user through configuring each credential in n8n. Include:
- Step-by-step instructions for each credential type
- Where to obtain API keys/credentials
- How to test each credential after setup

---

### Task 3: Populate Sample Vendor Data

Import sample vendor configurations to the `vendor_config` table for testing purposes.

**Sample Vendors to Create (10 examples across all tiers):**

```json
[
  {
    "vendor_id": "vendor_001",
    "vendor_name": "Acme Products",
    "data_source_type": "drive",
    "source_url": "https://drive.google.com/drive/folders/SAMPLE_FOLDER_ID",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Products organized by SKU in subfolders"
  },
  {
    "vendor_id": "vendor_002",
    "vendor_name": "Beta Supplies",
    "data_source_type": "shopify",
    "source_url": "https://beta-supplies.myshopify.com",
    "tier_used": 2,
    "sync_status": "pending",
    "notes": "Uses public /products.json endpoint"
  },
  {
    "vendor_id": "vendor_003",
    "vendor_name": "Gamma Industries",
    "data_source_type": "sheets",
    "source_url": "https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Product catalog in Sheet1"
  },
  {
    "vendor_id": "vendor_004",
    "vendor_name": "Delta Manufacturing",
    "data_source_type": "dropbox",
    "source_url": "https://www.dropbox.com/sh/SAMPLE_FOLDER",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Folder structure: /products/{SKU}/"
  },
  {
    "vendor_id": "vendor_005",
    "vendor_name": "Epsilon Wholesale",
    "data_source_type": "website",
    "source_url": "https://epsilon-wholesale.com/products",
    "tier_used": 3,
    "sync_status": "pending",
    "notes": "Requires Zyte + AI extraction"
  },
  {
    "vendor_id": "vendor_006",
    "vendor_name": "Zeta Distributors",
    "data_source_type": "shopify",
    "source_url": "https://zeta-dist.myshopify.com",
    "tier_used": 2,
    "sync_status": "pending",
    "notes": "Shopify store with pagination"
  },
  {
    "vendor_id": "vendor_007",
    "vendor_name": "Eta Imports",
    "data_source_type": "drive",
    "source_url": "https://drive.google.com/drive/folders/SAMPLE_FOLDER_ID_2",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Flat folder structure with SKU-named files"
  },
  {
    "vendor_id": "vendor_008",
    "vendor_name": "Theta Goods",
    "data_source_type": "website",
    "source_url": "https://thetagoods.com/catalog",
    "tier_used": 3,
    "sync_status": "pending",
    "notes": "Password-protected catalog"
  },
  {
    "vendor_id": "vendor_009",
    "vendor_name": "Iota Merchandise",
    "data_source_type": "sheets",
    "source_url": "https://docs.google.com/spreadsheets/d/SAMPLE_SHEET_ID_2",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Multiple sheets: Products, Images, Videos"
  },
  {
    "vendor_id": "vendor_010",
    "vendor_name": "Kappa Supplies Co",
    "data_source_type": "dropbox",
    "source_url": "https://www.dropbox.com/sh/SAMPLE_FOLDER_2",
    "tier_used": 1,
    "sync_status": "pending",
    "notes": "Organized by category then SKU"
  }
]
```

**How to Import:**

Option A: Create an n8n workflow to import from JSON/CSV
Option B: Use n8n's manual data entry interface
Option C: Use n8n's API to bulk insert

**Deliverable:** Confirm that all 10 sample vendors are successfully inserted into the `vendor_config` table.

---

### Task 4: Environment Variables Documentation

Create a comprehensive `.env.example` file that documents all required environment variables for the system. This file should be stored in the repository root.

**Required Variables to Document:**

```bash
# n8n Configuration
N8N_HOST=
N8N_API_KEY=

# Zyte API
ZYTE_API_KEY=
ZYTE_API_URL=https://api.zyte.com/v1/extract

# Google AI Studio
GOOGLE_AI_API_KEY=
GEMINI_MODEL=gemini-1.5-flash
GEMINI_API_URL=https://generativelanguage.googleapis.com/v1beta/models

# Google Drive
GOOGLE_DRIVE_CLIENT_ID=
GOOGLE_DRIVE_CLIENT_SECRET=
GOOGLE_DRIVE_REDIRECT_URI=

# Dropbox
DROPBOX_APP_KEY=
DROPBOX_APP_SECRET=
DROPBOX_ACCESS_TOKEN=

# Google Sheets
GOOGLE_SHEETS_SERVICE_ACCOUNT_EMAIL=
GOOGLE_SHEETS_PRIVATE_KEY=

# Email (MarketTime)
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_IMAP_PORT=993
EMAIL_USERNAME=
EMAIL_PASSWORD=

# Shopify Webhook
SHOPIFY_WEBHOOK_SECRET=

# Notifications
NOTIFICATION_EMAIL=
SLACK_WEBHOOK_URL=

# Customer Delivery
DELIVERY_METHOD=email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=
SMTP_PASSWORD=

# System Configuration
SYNC_SCHEDULE=0 2 * * 0
MAX_RETRIES=3
RETRY_DELAY_SECONDS=5
SHOPIFY_RATE_LIMIT_PER_SECOND=2
GEMINI_RATE_LIMIT_PER_MINUTE=60
LOG_LEVEL=info

# Cost Monitoring
MONTHLY_COST_ALERT_THRESHOLD=50
WEEKLY_COST_ALERT_THRESHOLD=15

# Security
ENCRYPTION_KEY=
API_TIMEOUT=30
```

**Deliverable:** Create `.env.example` file with detailed comments explaining each variable, where to obtain values, and usage notes.

---

### Task 5: Repository Structure Setup

Create a well-organized repository structure for documentation, templates, and scripts.

**Required Directory Structure:**

```
vendor-products/
├── README.md
├── IMPLEMENTATION_PLAN.md
├── .env.example
├── .gitignore
├── docs/
│   ├── schemas/
│   │   └── n8n-data-tables.md
│   ├── workflows/
│   │   └── (workflow exports will go here)
│   └── guides/
│       ├── SETUP_GUIDE.md
│       ├── VENDOR_ONBOARDING.md
│       └── TROUBLESHOOTING.md
├── templates/
│   ├── vendor-config-template.csv
│   ├── vendor-config-template.json
│   └── product-data-sample.json
├── scripts/
│   └── (utility scripts will go here)
└── config/
    └── (configuration files will go here)
```

**Deliverable:** Create all directories and placeholder files. Ensure the structure is committed to the repository.

---

### Task 6: Documentation Creation

Create comprehensive documentation for Phase 1 setup.

#### Document 1: `docs/schemas/n8n-data-tables.md`

Should include:
- Complete schema definitions for all three tables
- Example records for each table
- n8n implementation notes
- Data type mappings
- Indexing recommendations
- Security considerations

#### Document 2: `docs/guides/SETUP_GUIDE.md`

Should include:
- Prerequisites checklist
- Step-by-step n8n Data Tables setup
- API credentials configuration guide
- Email and webhook setup instructions
- Vendor data import process
- Testing procedures for each integration
- Security checklist
- Monitoring setup
- Backup and recovery procedures
- Troubleshooting common issues

#### Document 3: `docs/guides/VENDOR_ONBOARDING.md`

Should include:
- How to add a new vendor to the system
- Vendor data source requirements for each tier
- Credential setup for vendor-specific access
- Testing new vendor configurations
- Troubleshooting vendor-specific issues

**Deliverable:** All three documentation files created and committed to the repository.

---

## ✅ Phase 1 Completion Checklist

Before marking Phase 1 as complete, verify:

- [ ] All three n8n Data Tables created (`vendor_config`, `product_data`, `sync_log`)
- [ ] Table schemas match specifications exactly
- [ ] Encryption enabled for sensitive fields
- [ ] All 7 credential types documented in setup guide
- [ ] 10 sample vendors imported to `vendor_config` table
- [ ] `.env.example` file created with all required variables
- [ ] Repository directory structure created
- [ ] All documentation files created and complete
- [ ] `.gitignore` file includes `.env` and sensitive files
- [ ] All changes committed to repository
- [ ] Phase 1 completion report created

---

## 📊 Success Metrics for Phase 1

- **Time to Complete:** 2-4 hours
- **Tables Created:** 3/3
- **Sample Vendors:** 10/10
- **Documentation Files:** 3/3 minimum
- **Credential Types Documented:** 7/7

---

## 🚀 Next Steps After Phase 1

Once Phase 1 is complete:

1. **Phase 2, 3, 4 can begin in parallel** (Tier 1, 2, 3 scrapers)
2. User will configure actual API credentials in n8n
3. User will replace sample vendor data with real vendor information
4. Development of scraper workflows can commence

---

## 📝 Deliverable Format

Please provide a completion report with:

1. **Summary:** Brief overview of what was accomplished
2. **Tables Created:** Confirmation of all three tables with schema details
3. **Sample Data:** Confirmation of 10 vendors imported
4. **Documentation:** Links to all created documentation files
5. **Repository Structure:** Directory tree showing all created folders/files
6. **Issues Encountered:** Any problems and how they were resolved
7. **Next Steps:** Recommendations for Phase 2-4 development

---

## 🔧 Tools and Access

You have access to:
- ✅ GitHub repository: `amateurnerd44/vendor-products`
- ✅ File creation and editing tools
- ✅ Command-line tools for repository management
- ⚠️ n8n Cloud access (user will need to create tables manually or provide API access)

**Note:** Since you may not have direct n8n Cloud access, create detailed instructions and SQL-like schema definitions that the user can execute in their n8n instance.

---

## 💡 Tips for Success

1. **Be thorough:** This is the foundation for the entire system
2. **Document everything:** Future developers will rely on this documentation
3. **Use examples:** Provide concrete examples in all documentation
4. **Think ahead:** Consider what Phases 2-4 will need from this foundation
5. **Test instructions:** Ensure setup steps are clear and actionable
6. **Security first:** Emphasize encryption and credential management

---

## ❓ Questions to Consider

If you encounter ambiguity, make reasonable assumptions and document them:

- What should the default `sync_status` be for new vendors?
- Should `credentials` field be required or optional?
- What date format should be used for timestamps?
- Should there be validation rules for `data_source_type` values?
- What should happen if a vendor has no `source_url`?

Document your decisions in the completion report.

---

**Good luck! This foundation will enable the entire system to function smoothly. Take your time and be thorough.**

---

**Prompt Version:** 1.0  
**Created:** 2026-03-19  
**For Phase:** 1 - Foundation & Infrastructure  
**Estimated Time:** 2-4 hours

