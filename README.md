# 🛒 Google Shopping Feed Enrichment Kit

### 25 AI Skills + Orchestrator for Claude

Upload any product feed. Get a fully enriched supplemental feed back in minutes.

> Built by **[Ads & Insights](https://adsinsights.nl)** — Freelance Google Ads & eCommerce Specialist

---

## Why Feed Quality Matters More Than Ever

Google's AI-driven campaigns (Performance Max, Smart Shopping) are only as smart as the data you feed them. The algorithm decides which searches trigger your products, which audiences see them, and how much you pay — **all based on your feed attributes.**

The more complete and accurate your feed, the better the algorithm can do its job. Missing attributes like `color`, `material`, `size`, or `google_product_category` mean fewer signals for Google to work with — which often leads to less relevant traffic and higher costs.

Most feeds have room for improvement: empty fields, generic titles, missing product types. Fixing this manually takes time — especially with larger catalogs or multiple clients.

**This kit automates that work.** Upload your feed to Claude, type one sentence, get a fully enriched supplemental feed back in minutes.

---

## What This Kit Does

**26 Claude Skills** that analyze any product feed and fill in every missing Google Shopping attribute — automatically.

```
You:    "Enrich this feed with all productfeed skills"
Claude: [analyzes → extracts → classifies → generates → outputs .xlsx]
```

No code. No API setup. No spreadsheet formulas. Just upload and go.

### What gets enriched

| Category | Attributes |
|---|---|
| **Product identity** | `brand` · `color` · `material` · `pattern` · `size` · `gender` · `age_group` |
| **Technical flags** | `condition` · `adult` · `is_bundle` · `multipack` · `identifier_exists` |
| **Sizing** | `size_system` · `size_type` |
| **EU pricing compliance** | `unit_pricing_measure` · `unit_pricing_base_measure` |
| **Classification** | `google_product_category` · `product_type` · `item_group_id` |
| **Content generation** | `title` · `short_title` · `description` · `product_highlight` · `product_detail` |
| **Physical specs** | `product_weight` · dimensions (8 fields) |
| **Energy labels** | `energy_efficiency_class` (+ min/max) |

**35 attributes** across 25 specialized skills + 1 orchestrator.

---

## Key Features

🌍 **Works in any language** — Tested on Dutch, English, and Romanian feeds. Claude understands any input; the skills control the output language.

🔒 **Never overwrites good data** — Existing attributes stay untouched. Only empty or poor-quality fields get enriched.

✍️ **Conservative on generation** — Titles only optimized if they score below 60/100. Descriptions only touched if empty or under 150 characters.

📊 **Tested at scale** — Feeds from 100 to 35,000+ products processed in a single session.

🧠 **Context-aware** — Knows that supplements don't need a `color`, but shoes do. Doesn't hallucinate data that isn't there.

---

## Quick Start

### 1. Install the skills

Go to **[claude.ai](https://claude.ai)** → **Settings** → **Skills**

For each skill folder, create a new Claude skill and paste the contents of the `SKILL.md` file.

**Start with the orchestrator:** [`skills/productfeed-orchestrator.md`](productfeed-orchestrator.md)

### 2. Upload a feed

Upload any product feed to Claude — CSV, TSV, Excel (.xlsx), ZIP, gzip, pipe-delimited.

### 3. Go

**Run everything via the Orchestrator** — this triggers all 25 skills in the correct dependency order (extraction → classification → generation):
```
Enrich this feed with all productfeed skills
```

**Or run specific attributes only:**
```
Fill in the missing colors and materials in this feed
```

---

## How It Works

The orchestrator runs skills in 4 phases — each builds on the previous:

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
```

### Language handling

| Attribute type | Rule | Example |
|---|---|---|
| **Technical** (condition, gender...) | Always English | `new`, `male`, `true` |
| **Extracted** (color, material...) | Echoes the source | `zwart` or `black` |
| **Generated** (title, description...) | Matches feed language | NL feed → Dutch output |

---

## All Skills

### 🎯 Orchestrator

| Skill | Description |
|---|---|
| [`productfeed-orchestrator`](productfeed-orchestrator.md) | Master controller — runs all 25 skills in dependency order |

### Phase 1: Extraction

| Skill | Attribute(s) |
|---|---|
| [`productfeed-brand`](productfeed-brand.md) | `brand` |
| [`productfeed-color`](productfeed-color.md) | `color` |
| [`productfeed-material`](productfeed-material.md) | `material` |
| [`productfeed-gender`](productfeed-gender.md) | `gender` |
| [`productfeed-age-group`](productfeed-age-group.md) | `age_group` |
| [`productfeed-pattern`](productfeed-pattern.md) | `pattern` |
| [`productfeed-condition`](productfeed-condition.md) | `condition` |
| [`productfeed-adult`](productfeed-adult.md) | `adult` |
| [`productfeed-is-bundle`](productfeed-is-bundle.md) | `is_bundle` |
| [`productfeed-multipack`](productfeed-multipack.md) | `multipack` |
| [`productfeed-size`](productfeed-size.md) | `size` |
| [`productfeed-unit-pricing`](productfeed-unit-pricing.md) | `unit_pricing_measure` + `unit_pricing_base_measure` |
| [`productfeed-energy-efficiency`](productfeed-energy-efficiency.md) | `energy_efficiency_class` (+ min/max) |
| [`productfeed-dimensions`](productfeed-dimensions.md) | weight / length / width / height (product + shipping) |

### Phase 2: Dependent

| Skill | Attribute(s) | Depends on |
|---|---|---|
| [`productfeed-size-system`](productfeed-size-system.md) | `size_system` | size |
| [`productfeed-size-type`](productfeed-size-type.md) | `size_type` | size |
| [`productfeed-identifier-exists`](productfeed-identifier-exists.md) | `identifier_exists` | brand, gtin, mpn |

### Phase 3: Classification

| Skill | Attribute(s) |
|---|---|
| [`productfeed-google-product-category`](productfeed-google-product-category.md) | `google_product_category` |
| [`productfeed-product-type`](productfeed-product-type.md) | `product_type` |
| [`productfeed-item-group-id`](productfeed-item-group-id.md) | `item_group_id` |

### Phase 4: Generative

| Skill | Attribute(s) |
|---|---|
| [`productfeed-title`](productfeed-title.md) | `title` |
| [`productfeed-short-title`](productfeed-short-title.md) | `short_title` |
| [`productfeed-product-highlight`](productfeed-product-highlight.md) | `product_highlight` |
| [`productfeed-product-detail`](productfeed-product-detail.md) | `product_detail` |
| [`productfeed-description`](productfeed-description.md) | `description` |

---

## Output

Every run produces a **supplemental feed** in Excel (.xlsx):

- `id` + up to 35 enriched attribute columns
- English Google Merchant Center column names
- Empty cells where enrichment wasn't possible
- All products included

Upload directly to Google Merchant Center as a supplemental feed.

---

## Pro Tip: Automate New Product Enrichment

This kit enriches feeds on demand — but new products get added all the time. You can automate ongoing enrichment by connecting to the **Google Merchant Center API** (`products.list` to detect new items → enrich → `products.insert` to push back). Combine these skills with a Cloud Function on a schedule, and every new product gets enriched automatically.

---

## About

Built by **Tim** at **[Ads & Insights](https://adsinsights.nl)** — Freelance Google Ads & eCommerce Specialist based in the Netherlands.

**Need help with Google Ads or feed management?**
→ [adsinsights.nl](https://adsinsights.nl)
→ [LinkedIn](https://www.linkedin.com/in/timoudejans/)

---

## License

MIT — Free to use, modify, and distribute.
