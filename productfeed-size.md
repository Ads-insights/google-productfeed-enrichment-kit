---
name: productfeed-size
description: Extract and fill missing size attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [size] values. Also triggers when user mentions "size attribute", "maat invullen", "grootte invullen", "feed size", "missing sizes in feed", or asks to enrich/complete/fix size data. Use this skill even if the user just says "fill in the sizes" or "fix my feed sizes" with an uploaded file.
---

# Feed Size Extractor

Extract size values from product titles, descriptions, and other available fields to fill empty `size` attributes in a Google Shopping product feed.

## Why this matters

The `[size]` attribute is high-impact in Google Shopping. Missing size data leads to:
- Disapproved products in apparel categories (required for clothing/shoes)
- Missed size-specific searches (e.g., "sneakers maat 42")
- Poor variant grouping and filtering on Shopping tab
- Incorrect product comparisons

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

### Step 2: Size patterns

Size is more complex than other attributes because it has many formats depending on the product category.

```python
import re

# Letter sizes
LETTER_SIZES = {'xxs', 'xs', 's', 'm', 'l', 'xl', 'xxl', 'xxxl', '2xl', '3xl', '4xl', '5xl'}

# Numeric shoe sizes (EU)
SHOE_SIZES_EU = {str(n) for n in range(16, 51)}  # 16-50
# With halves
SHOE_SIZES_EU.update({f"{n}.5" for n in range(16, 51)})
SHOE_SIZES_EU.update({f"{n}½" for n in range(16, 51)})

# Clothing numeric sizes (EU)
CLOTHING_SIZES_EU = {str(n) for n in range(32, 61, 2)}  # 32-60 even numbers
CLOTHING_SIZES_EU.update({str(n) for n in range(34, 58)})  # broader range

# Metric units (PRIORITIZED — always prefer these over imperial)
METRIC_PATTERN = re.compile(r'(\d+(?:[.,]\d+)?)\s*(?:ml|l|liter|cl|dl|g|gr|kg|mg|mcg|cm|mm|m)\b', re.IGNORECASE)

# Imperial units (fallback — only use when no metric equivalent is present)
IMPERIAL_PATTERN = re.compile(r'(\d+(?:[.,]\d+)?)\s*(?:oz|fl\.?\s*oz|lb|lbs|inch|in|")', re.IGNORECASE)

# Dimension sizes
DIMENSION_PATTERN = re.compile(r'(\d+(?:[.,]\d+)?)\s*(?:cm|mm|m|inch|")', re.IGNORECASE)

# Maat/size explicit mention
MAAT_PATTERN = re.compile(r'(?:maat|size|mt\.?)\s*(\w+)', re.IGNORECASE)

# Size ranges
SIZE_RANGE_PATTERN = re.compile(r'(\d+)\s*[-/]\s*(\d+)', re.IGNORECASE)

def _scan_for_size(text):
    """Scan text for size matches. Returns (size, match_type) or (None, None).
    ALWAYS prioritizes metric units over imperial."""
    text_str = str(text) if pd.notna(text) else ''
    text_lower = text_str.lower()
    if not text_lower or text_lower == 'nan':
        return None, None

    # Explicit "maat X" or "size X"
    maat_match = MAAT_PATTERN.search(text_lower)
    if maat_match:
        return maat_match.group(1).upper(), 'explicit'

    # Letter sizes (standalone word)
    words = re.findall(r'\b[a-zA-Z0-9½.]+\b', text_lower)
    for word in words:
        if word.lower() in LETTER_SIZES:
            return word.upper(), 'letter'

    # Volume/weight/capacity — METRIC FIRST
    # Always prefer metric (g, ml, kg, l) over imperial (oz, lb, fl oz)
    # Many titles contain both: "2.5 oz (70 g)" — we want the metric value
    metric_match = METRIC_PATTERN.search(text_str)
    imperial_match = IMPERIAL_PATTERN.search(text_str)

    if metric_match:
        return metric_match.group(0).strip(), 'metric'
    if imperial_match:
        return imperial_match.group(0).strip(), 'imperial'

    # Dimensions
    dim_match = DIMENSION_PATTERN.search(text_str)
    if dim_match:
        return dim_match.group(0).strip(), 'dimension'

    return None, None

def extract_size(title, description='', product_type='', labels='',
                 google_category='', bullet_points=''):
    """Extract size using 4-layer cascade. Returns (size, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    size, _ = _scan_for_size(title)
    if size:
        return size, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    size, _ = _scan_for_size(description)
    if size:
        return size, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        size, _ = _scan_for_size(source_text)
        if size:
            return size, source_name, 'medium'

    # --- LAYER 4: No size found ---
    return None, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
size_col = next((c for c in df.columns if c in ['size', 'grootte', 'maat', 'g:size']), None)
if size_col is None:
    size_col = 'size'
    df[size_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[size_col]).strip() if pd.notna(row[size_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    size, source, confidence = extract_size(
        row.get(title_col, ''), row.get(desc_col, ''), row.get(ptype_col, ''))
    if size:
        df.at[idx, size_col] = size
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'size': df[size_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_size.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with fill rate before/after, sample of filled values, and unresolved list.

## Important guardrails

- **ALWAYS prioritize metric units** (g, ml, kg, l, cm) over imperial (oz, lb, fl oz). If both are present in the title (e.g., "2.5 oz (70 g)"), use the metric value.
- Size format depends on product category — don't force letter sizes on products that use numeric or volume sizing
- Never overwrite existing values
- Be careful with numbers that could be model numbers, not sizes (e.g., "Samsung S24" — S24 is not a size)
- **Dosages (mg, mcg) are valid as size for supplements** — "500 mg" is a meaningful product size for vitamins/supplements
- Volume sizes (ml, L) are valid for non-clothing products (urns, bottles, containers)
- When a title contains both a volume and a letter size, prefer the more product-relevant one
- "One size" / "one size fits all" / "OSFA" → output as `one size`
- Products without any size concept (electronics, digital) — leave blank

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `size`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
