---
name: productfeed-brand
description: Extract and fill missing brand attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [brand] values. Also triggers when user mentions "brand attribute", "merk invullen", "feed brand", "missing brands in feed", or asks to enrich/complete/fix brand data. Use this skill even if the user just says "fill in the brands" or "fix my feed brands" with an uploaded file.
---

# Feed Brand Extractor

Extract brand values from product titles, descriptions, and other available fields to fill empty `brand` attributes in a Google Shopping product feed.

## Why this matters

The `[brand]` attribute is high-impact in Google Shopping. Missing brand data leads to:
- Disapproved products (brand is required for most categories)
- Missed branded search queries (e.g., "Nike sneakers", "Samsung tv")
- Lower ad relevance and Quality Score
- Poor product matching and comparison

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

### Step 2: Brand extraction strategy

Brand extraction is different from other attributes because there is no fixed vocabulary of "all brands". The extraction follows a 4-layer cascade.

**Layer 1: Manufacturer/vendor column** — If the feed has a `manufacturer`, `vendor`, or `merk` column with data, use that directly. This is the most reliable source.

**Layer 2: Learn from existing brand data + match in title** — Look at products that already have a brand. Build a brand list, then check titles of products without a brand.

**Layer 3: Learn from existing brand data + match in other columns** — Same brand list, but scan description, product_type, labels, bullet points.

**Layer 4: First word heuristic** — The first word of a title is often the brand. Low confidence fallback.

```python
import re
from collections import Counter

def build_brand_list_from_feed(df, brand_col, title_col):
    """Extract known brands from products that already have brand filled."""
    known_brands = set()
    brand_filled = df[brand_col].notna() & (df[brand_col].str.strip() != '') & (df[brand_col].str.lower() != 'nan')
    for brand in df.loc[brand_filled, brand_col].unique():
        brand_clean = str(brand).strip()
        if brand_clean and len(brand_clean) > 1:
            known_brands.add(brand_clean)
    return known_brands

def _scan_for_known_brand(text, known_brands):
    """Check if any known brand appears in text. Returns brand or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    for brand in known_brands:
        if brand.lower() in text_lower:
            return brand
    return None

def extract_brand(title, description='', known_brands=set(), manufacturer='',
                  vendor='', product_type='', labels='', bullet_points=''):
    """Extract brand using 4-layer cascade. Returns (brand, source, confidence)."""

    # --- LAYER 1: Manufacturer/vendor column (highest confidence) ---
    for source_name, source_val in [('manufacturer', manufacturer), ('vendor', vendor)]:
        val = str(source_val).strip() if pd.notna(source_val) else ''
        if val and val.lower() not in ['', 'nan', 'none']:
            return val, source_name, 'high'

    # --- LAYER 2: Known brands matched in title (high confidence) ---
    brand = _scan_for_known_brand(title, known_brands)
    if brand:
        return brand, 'title (known brand)', 'high'

    # --- LAYER 3: Known brands matched in other columns (medium confidence) ---
    for source_name, source_text in [
        ('description', description),
        ('product_type', product_type),
        ('labels', labels),
        ('bullet_points', bullet_points),
    ]:
        brand = _scan_for_known_brand(source_text, known_brands)
        if brand:
            return brand, source_name, 'medium'

    # --- LAYER 4: First word heuristic (low confidence) ---
    GENERIC_FIRST_WORDS = {
        # Dutch
        'de', 'het', 'een', 'set', 'pack', 'paar', 'stuks',
        'grote', 'kleine', 'nieuwe', 'luxe', 'professionele', 'handgemaakte',
        'klassieke', 'moderne', 'originele', 'exclusieve', 'vintage',
        # English
        'the', 'a', 'an', 'new', 'large', 'small', 'big', 'best',
        'professional', 'handmade', 'classic', 'modern', 'original',
        'exclusive', 'official', 'genuine', 'authentic', 'deluxe',
        # Shared / language-neutral
        'premium', 'item', 'product', 'custom', 'personalized', 'personalised',
        'super', 'ultra', 'pro', 'mini', 'mega', 'extra',
    }

    title_str = str(title).strip() if pd.notna(title) else ''
    if title_str:
        parts = re.split(r'\s*[\|–—-]\s*', title_str)
        first_part = parts[0].strip()
        first_words = first_part.split()
        if first_words:
            candidate = first_words[0]
            if (candidate[0].isupper() and
                candidate.lower() not in GENERIC_FIRST_WORDS and
                len(candidate) > 1):
                return candidate, 'title (first word heuristic)', 'low'

    return None, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
brand_col = next((c for c in df.columns if c in ['brand', 'merk', 'g:brand']), None)
if brand_col is None:
    brand_col = 'brand'
    df[brand_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)

# Build brand list from existing data
known_brands = build_brand_list_from_feed(df, brand_col, title_col)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[brand_col]).strip() if pd.notna(row[brand_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    brand, source, confidence = extract_brand(
        row.get(title_col, ''), row.get(desc_col, ''), known_brands)
    if brand:
        df.at[idx, brand_col] = brand
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'brand': df[brand_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_brand.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with fill rate before/after. Specifically highlight:
- Brands found via known brand matching (high confidence)
- Brands found via first-word heuristic (low confidence — user should verify)
- Show a sample of each for spot-checking

## Important guardrails

- Never overwrite existing brand values
- The first-word heuristic has the lowest confidence — always flag these for manual review
- For private label / white label products, the brand is often the shop name itself — ask the user if you detect many unresolved products
- Brand values should preserve original casing (e.g., "Nike" not "nike", "IKEA" not "Ikea")
- If the feed has a `vendor` or `manufacturer` column, use that as a high-confidence source before falling back to title extraction

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `brand`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
