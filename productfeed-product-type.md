---
name: productfeed-product-type
description: Generate and fill missing product_type attributes in Google Shopping product feeds. Builds hierarchical category paths from broad to specific. Triggers when user uploads a product feed and wants to populate [product_type] values. Also triggers when user mentions "product type", "producttype", "product categorie", "categoriepad", or asks about product categorization. Use this skill even if the user just says "fill in the product types" or "categorize my products" with an uploaded file.
---

# Feed Product Type Generator

Generate `product_type` values — merchant-defined hierarchical category paths for campaign organization and product understanding.

## Why this matters

The `[product_type]` attribute is recommended and high-impact in Google Shopping:
- Helps Google understand what you're selling and match products to search queries
- Used to organize bidding and reporting in Google Ads Shopping campaigns
- Essential for countries where google_product_category subdivision is not available
- Improves product relevance and discoverability
- More flexible than google_product_category — you define your own hierarchy

## Google's specifications

**Format**: Free text with `>` as level separator (spaces before and after `>`)
- Example: `Home > Women > Dresses > Maxi Dresses`
- Example: `Electronics > Smartphones > Samsung`
- Example: `Wonen > Verlichting > Tafellampen`

**Key rules:**
- Use `>` to separate levels (with a space before and after)
- Go from broad to specific (left to right)
- Max 750 characters
- Submit only 1 value per product (Google recommendation for campaign organization)
- Language should match the feed language
- This is YOUR categorization, not Google's taxonomy — you can define whatever hierarchy makes sense
- Google uses this to understand your products AND for Shopping campaign product groups

**Depth guideline:** Aim for 2-5 levels. Too shallow (1 level) loses specificity. Too deep (6+) creates overly fragmented campaign groups.

**Difference with google_product_category:**
- `google_product_category` = Google's fixed taxonomy (6000+ predefined categories, use numeric ID)
- `product_type` = YOUR own hierarchy (free text, flexible, for campaign organization)
- Both can coexist and complement each other

## Workflow

### Step 1: Load and inspect the feed

```python
import pandas as pd

file_path = "/mnt/user-data/uploads/<filename>"
if file_path.endswith('.zip'):
    import zipfile
    with zipfile.ZipFile(file_path) as z:
        tsv_name = [f for f in z.namelist() if f.endswith('.tsv')][0]
        df = pd.read_csv(z.open(tsv_name), sep='\t', dtype=str)
elif file_path.endswith('.csv'):
    df = pd.read_csv(file_path, dtype=str)
elif file_path.endswith('.tsv'):
    df = pd.read_csv(file_path, sep='\t', dtype=str)
else:
    df = pd.read_excel(file_path, dtype=str)

df.columns = [c.strip().lower() for c in df.columns]
```

Report: total products, how many already have product_type, show the existing product_type distribution to understand the merchant's own categorization pattern.

### Step 2: Product type generation strategy

This is the most feed-specific skill — every webshop has its own logical product hierarchy. The approach:

**Layer 1: Learn from existing product_type values in the feed**
This is by far the most powerful strategy. If 40% of the feed already has product_type, analyze the patterns and apply them to similar uncategorized products.

**Layer 2: Derive from google_product_category**
If the feed has google_product_category but not product_type, convert the Google taxonomy path into a merchant-friendly hierarchy.

**Layer 3: Build from title + other attributes**
Use the product title, brand, and other available data to construct a reasonable category path.

**Layer 4: Build from URL structure**
Many webshops encode their category structure in the product URL (e.g., `/dames/jurken/maxi-jurken/`).

```python
import re
from collections import Counter
from urllib.parse import urlparse

def analyze_existing_product_types(df, ptype_col):
    """Analyze existing product_type values to understand the merchant's hierarchy.
    Returns: hierarchy patterns, level counts, and keyword-to-type mapping."""

    analysis = {
        'patterns': {},       # product_type → count
        'level_counts': [],   # distribution of hierarchy depth
        'level1_values': {},  # top-level categories → count
        'keyword_map': {},    # title keywords → product_type
    }

    if not ptype_col or ptype_col not in df.columns:
        return analysis

    has_ptype = df[ptype_col].notna() & (df[ptype_col].str.strip() != '') & (df[ptype_col].str.lower() != 'nan')

    for idx, row in df[has_ptype].iterrows():
        ptype = str(row[ptype_col]).strip()
        levels = [l.strip() for l in ptype.split('>')]
        num_levels = len(levels)

        analysis['patterns'][ptype] = analysis['patterns'].get(ptype, 0) + 1
        analysis['level_counts'].append(num_levels)

        if levels[0]:
            analysis['level1_values'][levels[0]] = analysis['level1_values'].get(levels[0], 0) + 1

    return analysis


def build_type_mapping_from_feed(df, ptype_col, title_col, brand_col, gpc_col):
    """Build a mapping to assign product_type to uncategorized products.
    Uses existing categorized products as training data."""

    mapping = {
        'by_brand': {},        # brand → most common product_type
        'by_gpc': {},          # google_product_category → most common product_type
        'by_title_words': {},  # significant title words → product_type
    }

    if not ptype_col or ptype_col not in df.columns:
        return mapping

    has_ptype = df[ptype_col].notna() & (df[ptype_col].str.strip() != '') & (df[ptype_col].str.lower() != 'nan')
    if has_ptype.sum() == 0:
        return mapping

    # Map brand → product_type
    if brand_col and brand_col in df.columns:
        for brand in df.loc[has_ptype, brand_col].dropna().unique():
            brand_clean = str(brand).strip().lower()
            if brand_clean and brand_clean not in ['', 'nan']:
                mask = has_ptype & (df[brand_col].str.lower().str.strip() == brand_clean)
                if mask.sum() > 0:
                    most_common = df.loc[mask, ptype_col].mode().iloc[0]
                    mapping['by_brand'][brand_clean] = most_common

    # Map google_product_category → product_type
    if gpc_col and gpc_col in df.columns:
        for gpc in df.loc[has_ptype, gpc_col].dropna().unique():
            gpc_clean = str(gpc).strip()
            if gpc_clean and gpc_clean.lower() not in ['', 'nan']:
                mask = has_ptype & (df[gpc_col] == gpc)
                if mask.sum() > 0:
                    most_common = df.loc[mask, ptype_col].mode().iloc[0]
                    mapping['by_gpc'][gpc_clean] = most_common

    # Map significant title words → product_type
    # Find words that strongly correlate with specific product types
    word_type_counts = {}
    for idx, row in df[has_ptype].iterrows():
        title = str(row.get(title_col, '')).lower()
        ptype = str(row[ptype_col]).strip()
        words = re.findall(r'[a-zà-ÿ-]+', title)
        for word in words:
            if len(word) > 3:  # Skip short words
                if word not in word_type_counts:
                    word_type_counts[word] = {}
                word_type_counts[word][ptype] = word_type_counts[word].get(ptype, 0) + 1

    # Keep words that strongly predict a single product_type (>70% of occurrences)
    for word, type_counts in word_type_counts.items():
        total = sum(type_counts.values())
        if total >= 3:  # Need at least 3 occurrences
            most_common_type = max(type_counts, key=type_counts.get)
            ratio = type_counts[most_common_type] / total
            if ratio > 0.7:
                mapping['by_title_words'][word] = most_common_type

    return mapping


def extract_category_from_url(url):
    """Extract category hierarchy from product URL structure."""
    if not url or str(url).lower() in ['', 'nan', 'none']:
        return None
    try:
        parsed = urlparse(str(url))
        path = parsed.path.strip('/')
        parts = [p for p in path.split('/') if p and p not in [
            'products', 'product', 'shop', 'collections', 'categorie',
            'category', 'p', 'item', 'detail'
        ]]
        # Filter out parts that look like product slugs (usually the last one)
        if len(parts) > 1:
            category_parts = parts[:-1]  # Remove last part (usually product slug)
            # Clean up: replace hyphens with spaces, title case
            clean_parts = [p.replace('-', ' ').replace('_', ' ').title() for p in category_parts]
            return ' > '.join(clean_parts)
    except:
        pass
    return None


def generate_product_type(title, description='', brand='', google_category='',
                          url='', labels='', mapping=None):
    """Generate product_type using the 4-layer cascade.
    Returns (product_type, source, confidence)."""

    title_lower = str(title).lower() if pd.notna(title) else ''
    brand_lower = str(brand).lower().strip() if pd.notna(brand) else ''
    gpc = str(google_category).strip() if pd.notna(google_category) else ''
    url_str = str(url).strip() if pd.notna(url) else ''

    if not mapping:
        mapping = {'by_brand': {}, 'by_gpc': {}, 'by_title_words': {}}

    # --- LAYER 1: Match from learned mappings ---

    # Try google_product_category → product_type mapping
    if gpc and gpc.lower() not in ['', 'nan'] and gpc in mapping.get('by_gpc', {}):
        return mapping['by_gpc'][gpc], 'learned from feed (google_category)', 'high'

    # Try brand → product_type mapping
    if brand_lower and brand_lower in mapping.get('by_brand', {}):
        return mapping['by_brand'][brand_lower], 'learned from feed (brand)', 'medium'

    # Try title keyword → product_type mapping
    title_words = re.findall(r'[a-zà-ÿ-]+', title_lower)
    for word in title_words:
        if word in mapping.get('by_title_words', {}):
            return mapping['by_title_words'][word], f'learned from feed (keyword: {word})', 'medium'

    # --- LAYER 2: Derive from google_product_category ---
    if gpc and gpc.lower() not in ['', 'nan']:
        # If GPC is a text path, use it directly as product_type
        if '>' in gpc:
            return gpc, 'derived from google_product_category', 'medium'
        # If GPC is a numeric ID, we can't easily convert without the full taxonomy
        # Leave for layer 3

    # --- LAYER 3: Extract from URL structure ---
    url_category = extract_category_from_url(url_str)
    if url_category:
        return url_category, 'extracted from URL', 'medium'

    # --- LAYER 4: No product type determinable ---
    return '', 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
brand_col = next((c for c in df.columns if c in ['brand', 'merk', 'manufacturer']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
gpc_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
url_col = next((c for c in df.columns if c in ['link', 'url', 'product url']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)

# Step 1: Analyze existing product_types
analysis = analyze_existing_product_types(df, ptype_col)

# Step 2: Build mapping from existing data
mapping = build_type_mapping_from_feed(df, ptype_col, title_col, brand_col, gpc_col)

# Step 3: Apply to uncategorized products
results = {'generated': 0, 'unresolved': 0, 'already_had': 0}
ptype_values = []

for idx, row in df.iterrows():
    if ptype_col:
        current = str(row[ptype_col]).strip() if pd.notna(row[ptype_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            ptype_values.append(current)
            continue

    ptype, source, confidence = generate_product_type(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(brand_col, ''), row.get(gpc_col, ''),
        row.get(url_col, ''), row.get(label_col, ''),
        mapping)

    ptype_values.append(ptype)
    if ptype:
        results['generated'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'product_type': ptype_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_product_type.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with:
- Fill rate before/after
- Distribution of assigned product types (top 15)
- Hierarchy depth distribution (how many levels per product)
- Sample of generated product types for spot-checking
- Show the learned mapping so the user can verify the pattern recognition
- Products that couldn't be categorized

Also show the existing product_type analysis:
- Top-level categories
- Average depth
- Consistency of naming

## Important guardrails

- Never overwrite existing product_type values
- Use ` > ` (space-greater than-space) as the separator — this is Google's required format
- Language must match the feed language
- Aim for 2-5 levels of depth (not too shallow, not too deep)
- Keep naming consistent within the feed (don't mix "Kleding" and "Clothing" in the same feed)
- Product type should reflect the merchant's own categorization logic, not Google's taxonomy
- The "learn from existing data" strategy is the most powerful — existing product_types in the feed define the style and language for new ones
- If a feed has zero existing product_types AND no google_product_category, URL extraction is the best fallback
- When nothing works, leave empty — a wrong product_type disrupts campaign organization
- Submit only 1 product_type per product (Google recommendation)
- Don't use product_type as a keyword dump — it should be a genuine category hierarchy

## Consistency principles

When generating product_types for a feed, maintain consistency with:
1. **Same vocabulary**: If existing types use "Kleding", don't introduce "Clothing"
2. **Same depth pattern**: If existing types are mostly 3 levels, aim for 3 levels
3. **Same top-level categories**: If existing types start with "Accessoires", group similar new products there too
4. **Same separator format**: Always ` > ` (space, greater-than, space)

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `product_type`
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
