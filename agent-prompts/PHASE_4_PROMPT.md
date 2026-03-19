# Phase 4 Agent Prompt - Tier 3 Zyte + AI Scraper

---

## 🎯 Your Mission

Complete **Phase 4: Tier 3 - Zyte + AI Scraper** for vendors with custom websites that require web scraping and AI extraction.

**Repository:** `amateurnerd44/vendor-products`

---

## 📋 Project Context

**Tier 3 vendors** have custom websites without public APIs. Solution:
- **Zyte API:** Renders JavaScript, handles anti-bot protection ($0.00013/request)
- **Gemini AI:** Extracts structured data from HTML (usage-based, ~$0.002/request)

**Target Cost:** ~$222/year for 20 Tier 3 vendors

---

## 🎯 Phase 4 Deliverables

### Task 1: Zyte API Integration

**Create workflow:** `tier3_zyte_fetcher.json`

**Steps:**
1. Input: URL to scrape
2. HTTP POST to Zyte API:
   ```json
   {
     "url": "https://vendor-site.com/products",
     "httpResponseBody": true,
     "javascript": true,
     "screenshot": false
   }
   ```
3. Handle response: Extract HTML body
4. Error handling: Retry on failure (max 3 attempts)
5. Output: Raw HTML

**Zyte Configuration:**
- Endpoint: `https://api.zyte.com/v1/extract`
- Auth: Bearer token in header
- Enable JavaScript rendering for dynamic sites
- Request screenshots only for debugging

### Task 2: Gemini AI Extraction Module

**Create workflow:** `tier3_gemini_extractor.json`

**AI Prompt Template:**
```
Extract product data from this HTML. Return JSON with this structure:
{
  "products": [
    {
      "sku": "string (required)",
      "product_name": "string (required)",
      "image_urls": ["array of image URLs"],
      "video_urls": ["array of video URLs"],
      "marketing_copy": "string (product description)",
      "price": "string (optional)",
      "availability": "string (optional)"
    }
  ]
}

Rules:
- Extract ALL products found on the page
- SKU must be unique identifier
- Image URLs must be absolute (not relative)
- Marketing copy should be clean text (no HTML)
- If data is missing, use null

HTML:
{html_content}
```

**Workflow Steps:**
1. Input: HTML from Zyte
2. Prepare prompt with HTML
3. HTTP POST to Gemini API:
   ```json
   {
     "contents": [{
       "parts": [{"text": "{prompt}"}]
     }],
     "generationConfig": {
       "temperature": 0.1,
       "maxOutputTokens": 8192
     }
   }
   ```
4. Parse JSON response
5. Validate extracted data
6. Output: Array of product objects

**Gemini Configuration:**
- Model: `gemini-1.5-flash` (fast and cheap)
- Temperature: 0.1 (deterministic)
- Max tokens: 8192 (handles large product lists)

### Task 3: AI Prompt Templates

**Create file:** `config/ai_prompts.json`

**Templates for different scenarios:**

1. **Product Page Extraction:**
   - Single product with details
   - Extract: SKU, name, images, videos, description, specs

2. **Catalog Page Extraction:**
   - Multiple products in grid/list
   - Extract: Basic info for each product
   - May need pagination

3. **Product List with Pagination:**
   - Detect "Next" button or page numbers
   - Extract current page products
   - Return next page URL

**Prompt Versioning:**
- Track prompt versions in git
- A/B test prompts for accuracy
- Document which prompts work best for which site types

### Task 4: Error Handling & Fallbacks

**Create workflow:** `tier3_error_handler.json`

**Error Scenarios:**

1. **Zyte API Failure:**
   - Retry with exponential backoff (1s, 2s, 4s)
   - If still fails, log error and skip vendor
   - Send alert to admin

2. **Gemini Extraction Failure:**
   - Try simpler prompt (extract just SKU and name)
   - If still fails, use regex fallback for basic extraction
   - Log partial data for manual review

3. **Invalid Data:**
   - Validate SKU format
   - Validate URLs are accessible
   - Flag products with missing required fields

4. **Rate Limiting:**
   - Gemini: Max 60 requests/minute
   - Implement queue with delays
   - Track usage to stay within budget

**Fallback Strategy:**
```
1. Try Zyte + Gemini (full extraction)
2. If Gemini fails → Try regex patterns
3. If regex fails → Log for manual extraction
4. If Zyte fails → Try direct HTTP request (may fail on protected sites)
```

### Task 5: Cost Tracking

**Create workflow:** `tier3_cost_tracker.json`

**Track:**
- Zyte requests: count × $0.00013
- Gemini requests: count × $0.002
- Daily/weekly/monthly totals
- Alert if approaching budget threshold

**Store in sync_log table:**
```json
{
  "vendor_id": "vendor_005",
  "tier_used": 3,
  "zyte_requests": 15,
  "gemini_requests": 15,
  "estimated_cost": 0.03225,
  "sync_date": "2026-03-19T10:00:00Z"
}
```

---

## 📁 Deliverable Files

```
vendor-products/
├── workflows/
│   ├── tier3/
│   │   ├── zyte_fetcher.json
│   │   ├── gemini_extractor.json
│   │   ├── error_handler.json
│   │   └── cost_tracker.json
│   └── tests/
│       ├── test_zyte_api.json
│       └── test_gemini_extraction.json
├── config/
│   └── ai_prompts.json
└── docs/
    └── workflows/
        └── TIER3_ZYTE_AI.md
```

---

## 📖 Documentation Requirements

Create `docs/workflows/TIER3_ZYTE_AI.md`:

1. **Overview:** How Zyte + Gemini work together
2. **Cost Analysis:** Per-request costs and budget management
3. **AI Prompts:** How to write effective extraction prompts
4. **Error Handling:** Retry logic and fallback strategies
5. **Testing:** How to test with sample websites
6. **Optimization:** Tips to reduce costs (caching, selective scraping)

---

## ✅ Completion Checklist

- [ ] Zyte API integration workflow created
- [ ] Gemini extraction workflow created
- [ ] AI prompt templates created (3+ scenarios)
- [ ] Error handling workflow created
- [ ] Cost tracking workflow created
- [ ] Retry logic implemented (max 3 attempts)
- [ ] Rate limiting implemented (60 req/min for Gemini)
- [ ] Test workflows created
- [ ] Documentation created
- [ ] All files committed

---

## 📊 Success Metrics

- **Time to Complete:** 8-10 hours
- **Workflows Created:** 4
- **AI Prompts:** 3+ templates
- **Extraction Accuracy:** >90% for structured sites
- **Cost per Vendor:** <$0.50/week
- **Error Recovery:** Graceful fallbacks for all failure modes

---

## 🚀 Next Steps

- **Phase 5:** Weekly Sync Workflow (requires Phases 2, 3, 4 complete)

---

## 💡 Implementation Tips

1. **Test with free sites first:** Use public product sites to test extraction
2. **Start with simple prompts:** Iterate to improve accuracy
3. **Monitor costs closely:** Track every API call
4. **Cache aggressively:** Don't re-scrape unchanged pages
5. **Use Gemini Flash:** Cheaper and faster than Pro
6. **Validate AI output:** Always check extracted data makes sense

---

**Phase:** 4 of 8  
**Dependencies:** Phase 1  
**Can Run in Parallel With:** Phases 2, 3  
**Estimated Time:** 8-10 hours  
**Cost Impact:** ~$222/year for 20 vendors

