# Agent Prompts - Vendor Product Data Collection System

This directory contains comprehensive, self-contained prompts for each phase of the project. Each prompt is designed to be used by a new agent with no prior knowledge or context.

---

## 📋 How to Use These Prompts

1. **Each prompt is standalone:** Copy the entire prompt and provide it to an agent
2. **No context needed:** All necessary background information is included
3. **Sequential execution:** Complete phases in order (1 → 2-4 → 5 → 6 → 7 → 8)
4. **Parallel opportunities:** Phases 2, 3, 4 can run simultaneously

---

## 🗂️ Phase Prompts

### [Phase 1: Foundation & Infrastructure](PHASE_1_PROMPT.md)
**Status:** Can start immediately (no dependencies)  
**Time:** 2-4 hours  
**Deliverables:**
- n8n Data Tables schemas (vendor_config, product_data, sync_log)
- Vendor configuration templates (CSV + JSON)
- Environment variables documentation
- Setup guide
- Repository structure

**Can Run in Parallel With:** Phases 2, 3, 4 (after completion)

---

### [Phase 2: Tier 1 - Direct Integration Scrapers](PHASE_2_PROMPT.md)
**Status:** Requires Phase 1 complete  
**Time:** 6-8 hours  
**Deliverables:**
- Google Drive scraper workflow
- Dropbox scraper workflow
- Google Sheets scraper workflow
- Data normalizer subworkflow
- Test workflows

**Can Run in Parallel With:** Phases 3, 4

---

### [Phase 3: Tier 2 - Shopify Public Endpoint Scraper](PHASE_3_PROMPT.md)
**Status:** Requires Phase 1 complete  
**Time:** 4-6 hours  
**Deliverables:**
- Shopify detector workflow
- Shopify scraper workflow
- SKU mapper workflow
- HTML cleaner subworkflow
- Test workflows

**Can Run in Parallel With:** Phases 2, 4

---

### [Phase 4: Tier 3 - Zyte + AI Scraper](PHASE_4_PROMPT.md)
**Status:** Requires Phase 1 complete  
**Time:** 8-10 hours  
**Deliverables:**
- Zyte API integration workflow
- Gemini AI extraction workflow
- AI prompt templates
- Error handler workflow
- Cost tracker workflow
- Test workflows

**Can Run in Parallel With:** Phases 2, 3

---

### [Phase 5: Flow 1 - Weekly Catalog Sync Workflow](PHASE_5_PROMPT.md)
**Status:** Requires Phases 2, 3, 4 complete  
**Time:** 6-8 hours  
**Deliverables:**
- Main sync orchestrator workflow
- Data upserter workflow
- Sync logger workflow
- Error notifier workflow
- Tier fallback workflow
- Test workflows

**Can Run in Parallel With:** None (sequential after Phases 2-4)

---

### [Phase 6: Flow 2 - Order Fulfillment Workflow](PHASE_6_PROMPT.md)
**Status:** Requires Phase 5 complete  
**Time:** 8-10 hours  
**Deliverables:**
- MarketTime email trigger workflow
- Shopify webhook trigger workflow
- SKU lookup workflow
- On-demand scraper workflow
- Asset compiler workflow
- Customer delivery workflow (4 methods)
- Failure handler workflow
- Test workflows

**Can Run in Parallel With:** None (sequential after Phase 5)

---

### [Phase 7: Testing & Validation](PHASE_7_PROMPT.md)
**Status:** Requires Phases 5, 6 complete  
**Time:** 6-8 hours  
**Deliverables:**
- Unit tests for all tier scrapers
- Integration tests for Flow 1 and Flow 2
- Edge case tests (20+ scenarios)
- Performance tests
- Cost validation tests
- Test documentation
- Performance report

**Can Run in Parallel With:** None (sequential after Phases 5-6)

---

### [Phase 8: Documentation & Deployment](PHASE_8_PROMPT.md)
**Status:** Requires Phase 7 complete  
**Time:** 4-6 hours  
**Deliverables:**
- User guide
- Architecture documentation
- API integration guide
- Runbook
- Deployment checklist
- Vendor onboarding guide
- All 70 vendors onboarded
- System deployed to production

**Can Run in Parallel With:** None (final phase)

---

## 📊 Project Timeline

### Sequential Timeline (No Parallelization)
**Total:** 44-60 hours

```
Phase 1 (2-4h) → Phase 2 (6-8h) → Phase 3 (4-6h) → Phase 4 (8-10h) 
→ Phase 5 (6-8h) → Phase 6 (8-10h) → Phase 7 (6-8h) → Phase 8 (4-6h)
```

### Optimized Timeline (With Parallelization)
**Total:** 25-35 hours

```
Phase 1 (2-4h)
    ↓
┌───┴───┬───────┐
│       │       │
Phase 2 Phase 3 Phase 4
(6-8h)  (4-6h)  (8-10h)
│       │       │
└───┬───┴───────┘
    ↓
Phase 5 (6-8h)
    ↓
Phase 6 (8-10h)
    ↓
Phase 7 (6-8h)
    ↓
Phase 8 (4-6h)
```

**Critical Path:** Phase 1 → Phase 4 → Phase 5 → Phase 6 → Phase 7 → Phase 8

---

## 🎯 Success Criteria

### System Performance
- ✅ Sync success rate: >95%
- ✅ Sync duration: <2 hours for 70 vendors
- ✅ Order fulfillment time: <10 seconds (cached SKUs)
- ✅ Order fulfillment time: <60 seconds (on-demand scraping)
- ✅ Cache hit rate: >90%

### Cost Efficiency
- ✅ Annual cost: <$300/year
- ✅ Weekly sync cost: ~$4.26
- ✅ Order fulfillment cost: ~$0.04/month

### Reliability
- ✅ Manual intervention rate: <5%
- ✅ Data freshness: All products updated within 7 days
- ✅ Error recovery: Automatic fallbacks for all failure modes

---

## 💰 Cost Breakdown

### Weekly Sync (70 vendors, ~100 products each)
- **Tier 1 (30 vendors):** FREE
- **Tier 2 (20 vendors):** FREE
- **Tier 3 (20 vendors):**
  - Zyte: 2,000 pages × $0.00013 = $0.26/week
  - Gemini: 2,000 pages × $0.002 = $4/week
  - **Weekly Total:** ~$4.26
  - **Annual Total:** ~$221

### Order Fulfillment (on-demand scraping)
- Assume 10% of orders need on-demand scraping
- 100 orders/month × 10% × 2 products/order = 20 scrapes/month
- Zyte: 20 × $0.00013 = $0.0026/month
- Gemini: 20 × $0.002 = $0.04/month
- **Monthly Total:** ~$0.04
- **Annual Total:** ~$0.50

### **Grand Total: ~$222/year**

---

## 🔧 Tech Stack

- **Orchestration:** n8n Cloud
- **Web Scraping:** Zyte API ($0.00013/request)
- **AI Extraction:** Google AI Studio Gemini API (usage-based)
- **Data Storage:** n8n Data Tables
- **Tier 1 APIs:** Google Drive, Dropbox, Google Sheets (FREE)
- **Tier 2:** Shopify public JSON endpoints (FREE)
- **Triggers:** Email (IMAP), Shopify webhooks

---

## 📞 Support

For questions or issues:
1. Review the specific phase prompt
2. Check the IMPLEMENTATION_PLAN.md in repository root
3. Refer to documentation in `docs/` directory

---

**Project:** Vendor Product Data Collection System  
**Repository:** amateurnerd44/vendor-products  
**Total Phases:** 8  
**Estimated Timeline:** 25-35 hours (with parallelization)  
**Target Cost:** ~$222/year

