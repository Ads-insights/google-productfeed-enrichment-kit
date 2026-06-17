# 🛒 Google Shopping Feed Enrichment Kit

### 31 AI Skills + Orchestrator for Claude

Upload any product feed. Get a fully enriched feed with 41+ key attributes back in minutes — including the new attributes for Google's **conversational AI / AI Mode** shopping experience.

> Built by **[Ads & Insights](https://adsinsights.nl)** — Freelance Google Ads & eCommerce Specialist

---

## Why Feed Quality Matters More Than Ever

Google's AI-driven campaigns (Performance Max, Smart Shopping) are only as smart as the data you feed them. The algorithm decides which searches trigger your products, which audiences see them, and how much you pay — **all based on your feed attributes.**

The more complete and accurate your feed, the better the algorithm can do its job. Missing attributes like `color`, `material`, `size`, or `google_product_category` mean fewer signals for Google to work with — which often leads to less relevant traffic and higher costs.

And with the rise of **AI Mode in Google Search**, a new layer of attributes now feeds the conversational shopping experience — Q&A, product relationships, variant grouping, and popularity signals. Feeds that supply this data get to shape how AI talks about their products; feeds that don't are left to inference.

Most feeds have room for improvement: empty fields, generic titles, missing product types. Fixing this manually takes time — especially with larger catalogs or multiple clients.

**This kit automates that work.** Upload your feed to Claude, type one sentence, get a fully enriched supplemental feed back in minutes.

---

## What This Kit Does

**31 Claude Skills** that analyze any product feed and fill in every missing Google Shopping attribute — automatically.

```
You:    "Enrich this feed with all productfeed skills"
Claude: [analyzes → extracts → classifies → generates → AI Mode → outputs .xlsx]
```

No code. No API setup. No spreadsheet formulas. Just upload and go.

### What gets enriched

| Category | Attributes |
| --- | --- |
| **Product identity** | `brand` · `color` · `material` · `pattern` · `size` · `gender` · `age_group` |
| **Technical flags** | `condition` · `adult` · `is_bundle` · `multipack` · `identifier_exists` |
| **Sizing** | `size_system` · `size_type` |
| **EU pricing compliance** | `unit_pricing_measure` · `unit_pricing_base_measure` |
| **Classification** | `google_product_category` · `product_type` · `item_group_id` |
| **Content generation** | `title` · `short_title` · `description` · `product_highlight` · `product_detail` |
| **Physical specs** | `product_weight` · dimensions (8 fields) |
| **Energy labels** | `energy_efficiency_class` (+ min/max) |
| **AI Mode (conversational)** | `question_and_answer` · `item_group_title` · `variant_option` · `related_product` · `document_link` · `popularity_rank` |

**41 attributes** across 31 specialized skills + 1 orchestrator.

---

## Key Features

🌍 **Works in any language** — Claude understands any input; the skills control the output language.

🔒 **Never overwrites good data** — Existing attributes stay untouched. Only empty or poor-quality fields get enriched.

✍️ **Conservative on generation** — Titles only optimized if they score below 60/100. Descriptions only touched if empty or under 150 characters.

🤖 **Never fabricates AI Mode data** — `document_link` and `popularity_rank` only map, validate, or compute from real source data and report honestly when it's missing. `question_and_answer` draws only from hard facts and never duplicates other attributes.

📊 **Tested at scale** — Feeds from 100 to 35,000+ products processed in a single session.

🧠 **Context-aware** — Knows that supplements don't need a `color`, but shoes do. Doesn't hallucinate data that isn't there.

---

## Quick Start

> **Requires a Claude Pro, Team, or Enterprise subscription.** Skills are not available on the free plan.

### 1. Install the skills

Go to **[claude.ai](https://claude.ai)** → **Customize** → **Skills**

For each of the 32 `.md` files in this repository:

1. Click the **+** button → **Upload a skill**
2. Open the `.md` file from this repo (or copy-paste its contents)
3. Save

**Install all 32 for the best results.** The orchestrator coordinates them, but each skill handles a specific attribute. You can also pick and choose — if you only need colors and materials, just install those two skills.

**The orchestrator is key:** [`productfeed-orchestrator.md`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-orchestrator.md) — install this one first, it runs all other installed skills in the correct order.

### 2. Upload a feed

Upload any product feed to Claude — CSV, TSV, Excel (.xlsx), ZIP, gzip, pipe-delimited.

### 3. Go

**Run everything via the Orchestrator** — this triggers all 31 skills in the correct dependency order (extraction → classification → generation → AI Mode):

```
Enrich this feed with all productfeed skills
```

**Or run specific attributes only:**

```
Fill in the missing colors and materials in this feed
```

---

## How It Works

The orchestrator runs skills in 5 phases — each builds on the previous:

```
Phase 1 — EXTRACTION (14 skills)
  brand · color · material · gender · age_group · pattern
  condition · adult · is_bundle · multipack · size
  unit_pricing · energy_efficiency · dimensions

Phase 2 — DEPENDENT (3 skills, need Phase 1)
  size_system · size_type · identifier_exists

Phase 3 — CLASSIFICATION (3 skills, improved by Phase 1)
  google_product_category · product_type · item_group_id

Phase 4 — GENERATIVE (5 skills, need all previous)
  title → short_title → product_highlight → product_detail → description

Phase 5 — AI MODE (6 skills, conversational experience)
  item_group_title · variant_option · question_and_answer
  related_product · document_link · popularity_rank
```

Phase 5 runs last on purpose: it needs `item_group_id` from Phase 3 and the generated `title` / `description` / `product_detail` / `product_highlight` from Phase 4, so `question_and_answer` can avoid duplicating them.

### Language handling

| Attribute type | Rule | Example |
| --- | --- | --- |
| **Technical** (condition, gender...) | Always English | `new`, `male`, `true` |
| **Extracted** (color, material...) | Echoes the source | `zwart` or `black` |
| **Generated** (title, description, Q&A...) | Matches feed language | NL feed → Dutch output |
| **Identifiers / numbers** (related_product, document_link, popularity_rank) | Language-independent | ids, URLs, `0.0–100.0` |

---

## All Skills

### 🎯 Orchestrator

| Skill | Description |
| --- | --- |
| [`productfeed-orchestrator`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-orchestrator.md) | Master controller — runs all 31 skills in dependency order |

### Phase 1: Extraction

| Skill | Attribute(s) |
| --- | --- |
| [`productfeed-brand`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-brand.md) | `brand` |
| [`productfeed-color`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-color.md) | `color` |
| [`productfeed-material`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-material.md) | `material` |
| [`productfeed-gender`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-gender.md) | `gender` |
| [`productfeed-age-group`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-age-group.md) | `age_group` |
| [`productfeed-pattern`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-pattern.md) | `pattern` |
| [`productfeed-condition`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-condition.md) | `condition` |
| [`productfeed-adult`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-adult.md) | `adult` |
| [`productfeed-is-bundle`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-is-bundle.md) | `is_bundle` |
| [`productfeed-multipack`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-multipack.md) | `multipack` |
| [`productfeed-size`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-size.md) | `size` |
| [`productfeed-unit-pricing`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-unit-pricing.md) | `unit_pricing_measure` + `unit_pricing_base_measure` |
| [`productfeed-energy-efficiency`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-energy-efficiency.md) | `energy_efficiency_class` (+ min/max) |
| [`productfeed-dimensions`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-dimensions.md) | weight / length / width / height (product + shipping) |

### Phase 2: Dependent

| Skill | Attribute(s) | Depends on |
| --- | --- | --- |
| [`productfeed-size-system`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-size-system.md) | `size_system` | size |
| [`productfeed-size-type`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-size-type.md) | `size_type` | size |
| [`productfeed-identifier-exists`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-identifier-exists.md) | `identifier_exists` | brand, gtin, mpn |

### Phase 3: Classification

| Skill | Attribute(s) |
| --- | --- |
| [`productfeed-google-product-category`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-google-product-category.md) | `google_product_category` |
| [`productfeed-product-type`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-product-type.md) | `product_type` |
| [`productfeed-item-group-id`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-item-group-id.md) | `item_group_id` |

### Phase 4: Generative

| Skill | Attribute(s) |
| --- | --- |
| [`productfeed-title`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-title.md) | `title` |
| [`productfeed-short-title`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-short-title.md) | `short_title` |
| [`productfeed-product-highlight`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-product-highlight.md) | `product_highlight` |
| [`productfeed-product-detail`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-product-detail.md) | `product_detail` |
| [`productfeed-description`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-description.md) | `description` |

### Phase 5: AI Mode (conversational experience)

These attributes power Google's conversational / AI Mode shopping experience. They behave differently from the rest of the kit: some are group attributes, some reason across the whole feed, and some require merchant performance data — so by design they never fabricate values.

| Skill | Attribute(s) | Approach |
| --- | --- | --- |
| [`productfeed-question-and-answer`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-question-and-answer.md) | `question_and_answer` | Generative, conservative — factual Q&A from hard feed data only |
| [`productfeed-item-group-title`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-item-group-title.md) | `item_group_title` | Derived — one shared title per variant group, stripped of variant tokens |
| [`productfeed-variant-option`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-variant-option.md) | `variant_option` | Extraction — varying variant dimensions as `name:value` pairs |
| [`productfeed-related-product`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-related-product.md) | `related_product` | Cross-product — explicit columns preferred over heuristics |
| [`productfeed-document-link`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-document-link.md) | `document_link` | Map-and-validate PDF URLs — never invents links |
| [`productfeed-popularity-rank`](https://github.com/Ads-insights/google-productfeed-enrichment-kit/blob/main/productfeed-popularity-rank.md) | `popularity_rank` | Validate or compute a 0–100 percentile from a sales/quantity column |

> **Note on AI Mode attributes:** `item_group_title` and `variant_option` only apply to grouped variants (they need `item_group_id`). `document_link` and `popularity_rank` need merchant-supplied data (PDF URLs / sales figures) — without it, the skills tell you it's not possible rather than guessing.

---

## Output

Every run produces a **supplemental feed** in Excel (.xlsx):

- `id` + up to 41 enriched attribute columns
- English Google Merchant Center column names
- Group attributes (`question_and_answer`, `variant_option`, `related_product`) correctly escaped for Google Sheets
- Empty cells where enrichment wasn't possible
- All products included

Upload directly to Google Merchant Center as a supplemental feed.

---

## Pro Tip: Automate New Product Enrichment

This kit enriches feeds on demand — but new products get added all the time. You can automate ongoing enrichment by connecting to the **Google Merchant Center API** (`products.list` to detect new items → enrich → `products.insert` to push back). Combine these skills with a Cloud Function on a schedule, and every new product gets enriched automatically.

---

## Requirements

- **Claude Pro, Team, or Enterprise subscription** — Skills are not available on the free Claude plan.

---

## Disclaimer

This kit is provided as-is. While tested on multiple feeds across different verticals and languages, **edge cases may occur** — especially with unusual product titles, non-standard feed formats, or niche product categories. Always spot-check the output before uploading to Google Merchant Center. The skills are conservative by design (they won't overwrite good data or hallucinate attributes), but a manual review of a sample of enriched products is recommended.

The AI Mode attributes (`question_and_answer`, `related_product`, etc.) are relatively new in Google Merchant Center and primarily intended for conversational experiences such as AI Mode in Google Search. Availability and behavior may evolve — check the [Google Merchant Center Help](https://support.google.com/merchants) for the latest specifications.

---

## About

Built by **Tim** at **[Ads & Insights](https://adsinsights.nl)** — Freelance Google Ads & eCommerce Specialist based in the Netherlands.

**Need help with Google Ads or feed management?** → [adsinsights.nl](https://adsinsights.nl) → [LinkedIn](https://www.linkedin.com/in/timoudejans/)

---

## License

MIT — Free to use, modify, and distribute.
