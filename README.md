# Vendor Product Data Collection System

🎯 **Automated system for collecting board game and toy product data from vendors**

[![Phase 1](https://img.shields.io/badge/Phase%201-Complete-success)](docs/guides/SETUP_GUIDE.md)
[![n8n](https://img.shields.io/badge/Built%20with-n8n-FF6D5A)](https://n8n.io)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **Note**: This README describes the general system architecture. The actual implementation uses a **31-field product schema** optimized for board games and toys, with **Shopify vendor management** instead of a separate vendor_config table. See [Product Data Schema](docs/schemas/product_data.md) for the real schema.

---

## 📋 Overview

This system automates the collection of board game and toy product data from multiple vendors using a tiered approach:

- **Tier 1** (FREE): Direct integrations with Google Drive, Dropbox, and Google Sheets
- **Tier 2** (FREE): Shopify public JSON endpoints
- **Tier 3** (Usage-based): Zyte API + Google AI Studio for web scraping
- **Tier 4** (FREE): BoardGameGeek (BGG) API for game-specific enrichment

### Key Features

✅ **Weekly Automated Sync** - Keeps product catalogs up-to-date  
✅ **On-Demand Scraping** - Fallback for missing SKUs during orders  
✅ **Fast Order Fulfillment** - <10 seconds for cached products  
✅ **Cost Efficient** - ~$222/year for 70 vendors  
✅ **Usage-Based Pricing** - No monthly subscriptions  
✅ **BGG Integration** - Enrich board games with mechanics, categories, complexity ratings  
✅ **Shopify Vendor Management** - Leverage existing Shopify vendor system  

---

## 🚀 Quick Start

### Prerequisites

- [n8n Cloud](https://n8n.io/cloud/) account (Starter or Pro plan)
- [Zyte API](https://www.zyte.com/zyte-api/) account
- [Google AI Studio](https://aistudio.google.com/app/apikey) API key
- Google Cloud project (for Drive/Sheets integration)

### Setup Steps

1. **Clone this repository**
   ```bash
   git clone https://github.com/amateurnerd44/vendor-products.git
   cd vendor-products
   ```

2. **Follow the Setup Guide**
   - Read [SETUP_GUIDE.md](SETUP_GUIDE.md) for detailed instructions
   - Estimated time: 2-4 hours

3. **Configure Environment**
   ```bash
   cp .env.example .env
   # Edit .env with your API keys and credentials
   ```

4. **Import Vendor Configuration**
   - Use templates in `config/` folder
   - Import to n8n Data Tables

5. **Test Integrations**
   - Follow testing steps in [SETUP_GUIDE.md](SETUP_GUIDE.md#testing)

---

## 📁 Repository Structure

```
vendor-products/
├── docs/
│   ├── schemas/              # n8n Data Table schemas
│   │   ├── vendor_config.md
│   │   ├── product_data.md
│   │   └── sync_log.md
│   └── guides/               # Setup and usage guides
│       └── SETUP_GUIDE.md
├── templates/                # Configuration templates
│   ├── vendor_config_template.json
│   └── vendor_config_template.csv
├── config/                   # Sample configurations
│   ├── sample_vendors.json
│   └── sample_vendors.csv
├── scripts/                  # Utility scripts (future)
├── .env.example              # Environment variables template
├── IMPLEMENTATION_PLAN.md    # Full 8-phase implementation plan
├── DOCUMENTATION_INDEX.md    # Central documentation hub
└── README.md                 # This file
```

---

## 📚 Documentation

### Core Documentation
- **[Setup Guide](SETUP_GUIDE.md)** - Complete setup instructions
- **[Implementation Plan](IMPLEMENTATION_PLAN.md)** - 8-phase development roadmap
- **[Documentation Index](DOCUMENTATION_INDEX.md)** - All documentation links

### Data Schemas
- **[Vendor Config Schema](docs/schemas/vendor_config.md)** - Vendor configuration table
- **[Product Data Schema](docs/schemas/product_data.md)** - Product information table
- **[Sync Log Schema](docs/schemas/sync_log.md)** - Sync operation logs

### Templates
- **[Vendor Config Template (JSON)](templates/vendor_config_template.json)**
- **[Vendor Config Template (CSV)](templates/vendor_config_template.csv)**
- **[Sample Vendors (JSON)](config/sample_vendors.json)** - 10 example vendors
- **[Sample Vendors (CSV)](config/sample_vendors.csv)** - 10 example vendors

---

## 🏗️ System Architecture

### Flow 1: Weekly Product Catalog Sync
```
Cron Trigger (Sunday 2 AM)
    ↓
Loop Through Vendors
    ↓
Route by Tier (1, 2, or 3)
    ↓
Collect Product Data
    ↓
Update product_data Table
    ↓
Log Results to sync_log
```

### Flow 2: Order Fulfillment
```
Order Trigger (Email/Webhook)
    ↓
Extract SKUs
    ↓
Query product_data Table
    ↓
If SKU Missing → On-Demand Scrape
    ↓
Compile Digital Asset Package
    ↓
Deliver to Customer
```

### Tiered Data Collection

| Tier | Method | Cost | Vendors |
|------|--------|------|---------|
| 1 | Google Drive, Dropbox, Sheets | FREE | 30 vendors |
| 2 | Shopify JSON endpoints | FREE | 20 vendors |
| 3 | Zyte API + Gemini AI | ~$0.002/product | 20 vendors |

---

## 💰 Cost Breakdown

### Annual Costs
- **Weekly Sync**: ~$221/year
  - Tier 1 (30 vendors): $0
  - Tier 2 (20 vendors): $0
  - Tier 3 (20 vendors): $4.26/week × 52 = $221
- **On-Demand Scraping**: ~$0.50/year
- **Total**: ~$222/year

### Cost Per Sync (Tier 3)
- Zyte API: 2,000 pages × $0.00013 = $0.26
- Gemini AI: 2,000 pages × $0.002 = $4.00
- **Total**: $4.26 per weekly sync

---

## 🎯 Implementation Phases

### ✅ Phase 1: Foundation & Infrastructure (COMPLETE)
- [x] Repository structure
- [x] n8n Data Tables documentation
- [x] Vendor configuration templates
- [x] Environment variables setup
- [x] Setup guide

### 🔄 Phase 2: Tier 1 Scrapers (Next)
- [ ] Google Drive integration
- [ ] Dropbox integration
- [ ] Google Sheets integration
- [ ] Data normalization module

### 📅 Upcoming Phases
- **Phase 3**: Tier 2 - Shopify scrapers
- **Phase 4**: Tier 3 - Zyte + AI scrapers
- **Phase 5**: Weekly sync workflow
- **Phase 6**: Order fulfillment workflow
- **Phase 7**: Testing & validation
- **Phase 8**: Documentation & deployment

See [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) for full details.

---

## 🔧 Tech Stack

- **Orchestration**: [n8n Cloud](https://n8n.io/cloud/)
- **Web Scraping**: [Zyte API](https://www.zyte.com/zyte-api/)
- **AI Extraction**: [Google AI Studio (Gemini)](https://aistudio.google.com/)
- **Data Storage**: n8n Data Tables
- **Integrations**: Google Drive, Dropbox, Google Sheets, Shopify

---

## 📊 Success Metrics

| Metric | Target | Current |
|--------|--------|---------|
| Sync Success Rate | >95% | TBD |
| Order Fulfillment Speed | <10s (cached) | TBD |
| Data Freshness | <7 days | TBD |
| Annual Cost | <$300 | ~$222 |
| Manual Intervention Rate | <5% | TBD |

---

## 🛠️ Development

### Prerequisites
- n8n Cloud account
- API keys for Zyte, Google AI Studio
- OAuth2 credentials for Google Drive/Sheets
- Dropbox access token

### Local Development
```bash
# Clone repository
git clone https://github.com/amateurnerd44/vendor-products.git
cd vendor-products

# Copy environment template
cp .env.example .env

# Edit .env with your credentials
nano .env

# Follow setup guide
open SETUP_GUIDE.md
```

### Testing
See [SETUP_GUIDE.md - Testing Section](SETUP_GUIDE.md#testing) for:
- API connection tests
- Data table tests
- Integration tests for each tier

---

## 🤝 Contributing

This is a private project, but contributions are welcome:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 🆘 Support

### Documentation
- [Setup Guide](SETUP_GUIDE.md)
- [Implementation Plan](IMPLEMENTATION_PLAN.md)
- [Schema Documentation](docs/schemas/)

### External Resources
- [n8n Documentation](https://docs.n8n.io/)
- [Zyte API Docs](https://docs.zyte.com/)
- [Google AI Studio](https://ai.google.dev/)

### Issues
If you encounter any issues:
1. Check the [Troubleshooting section](SETUP_GUIDE.md#troubleshooting)
2. Review the [Schema Documentation](docs/schemas/)
3. Open an issue on GitHub

---

## 🎉 Acknowledgments

- Built with [n8n](https://n8n.io/) - Workflow automation platform
- Powered by [Zyte API](https://www.zyte.com/) - Web scraping service
- AI extraction by [Google Gemini](https://ai.google.dev/) - Generative AI

---

## 📈 Roadmap

- [x] **Phase 1**: Foundation & Infrastructure (Complete)
- [ ] **Phase 2**: Tier 1 Scrapers (In Progress)
- [ ] **Phase 3**: Tier 2 Scrapers
- [ ] **Phase 4**: Tier 3 Scrapers
- [ ] **Phase 5**: Weekly Sync Workflow
- [ ] **Phase 6**: Order Fulfillment Workflow
- [ ] **Phase 7**: Testing & Validation
- [ ] **Phase 8**: Documentation & Deployment

**Estimated Completion**: 25-35 hours with parallel development

---

**Project Status**: 🟢 Active Development  
**Current Phase**: Phase 1 Complete, Phase 2 Starting  
**Last Updated**: 2026-03-19

---

Made with ❤️ using n8n, Zyte, and Google AI
