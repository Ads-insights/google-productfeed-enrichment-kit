---
name: productfeed-dimensions
description: Extract and fill missing product and shipping dimension attributes in Google Shopping product feeds. Covers product_weight, product_length, product_width, product_height, shipping_weight, shipping_length, shipping_width, and shipping_height. Triggers when user uploads a product feed and wants to populate dimension/weight values. Also triggers when user mentions "dimensions", "afmetingen", "gewicht", "weight", "lengte", "breedte", "hoogte", or asks about product measurements. Use this skill even if the user just says "fill in the dimensions" or "fill in the weight" with an uploaded file.
---

# Feed Dimensions Extractor

Extract product dimensions (weight, length, width, height) and shipping dimensions from product titles, descriptions, and other fields.

## Why this matters

Product dimensions improve Shopping performance by:
- Enabling accurate shipping cost calculations
- Improving product comparisons and filtering
- Providing better product information to shoppers
- Supporting carrier-calculated shipping rates

Shipping dimensions are typically identical to product dimensions unless the product has special packaging.

## Valid formats

Google requires dimensions in specific formats:

**Weight**: `<number> <unit>` — accepted units: `lb`, `oz`, `g`, `kg`
**Length/Width/Height**: `<number> <unit>` — accepted units: `in`, `cm`

For Dutch/EU webshops, default to metric: `kg` for weight, `cm` for dimensions.

## Workflow

### Step 1: Load the feed

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

### Step 2: Extraction logic

```python
import re

# === WEIGHT PATTERNS ===
WEIGHT_PATTERNS = [
    # "gewicht: 2.5 kg", "weight: 500g"
    re.compile(r'(?:gewicht|weight|wt\.?)\s*[:=]?\s*(\d+(?:[.,]\d+)?)\s*(kg|g|gram|kilogram|lb|oz)\b', re.I),
    # "weegt 2.5 kg", "weighs 500g"
    re.compile(r'(?:weegt|weighs)\s+(\d+(?:[.,]\d+)?)\s*(kg|g|gram|kilogram|lb|oz)\b', re.I),
    # Standalone weight with clear context: "2.5 kg" near weight-related words
    re.compile(r'(\d+(?:[.,]\d+)?)\s*(kg|kilogram)\b', re.I),
]

# Fallback: standalone gram (but be careful — gram can be false positive)
WEIGHT_FALLBACK = re.compile(r'(\d+(?:[.,]\d+)?)\s*(g|gram)\b', re.I)

# === DIMENSION PATTERNS ===
# "L x B x H: 30 x 20 x 15 cm" or "30x20x15cm"
DIMENSION_LxWxH = re.compile(
    r'(?:(?:l\s*x\s*b\s*x\s*h|l\s*x\s*w\s*x\s*h|afmetingen|dimensions?|maten)\s*[:=]?\s*)?'
    r'(\d+(?:[.,]\d+)?)\s*[x×X]\s*(\d+(?:[.,]\d+)?)\s*[x×X]\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in|")',
    re.I
)

# "L x B: 30 x 20 cm" (2D, no height)
DIMENSION_LxW = re.compile(
    r'(\d+(?:[.,]\d+)?)\s*[x×X]\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in|")',
    re.I
)

# Explicit individual dimensions
LENGTH_PATTERNS = [
    re.compile(r'(?:lengte|length|lang|diepte|depth)\s*[:=]?\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in)', re.I),
]
WIDTH_PATTERNS = [
    re.compile(r'(?:breedte|width|breed|wijd)\s*[:=]?\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in)', re.I),
]
HEIGHT_PATTERNS = [
    re.compile(r'(?:hoogte|height|hoog|hgt)\s*[:=]?\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in)', re.I),
]
DIAMETER_PATTERNS = [
    re.compile(r'(?:diameter|ø|doorsnede)\s*[:=]?\s*(\d+(?:[.,]\d+)?)\s*(cm|mm|m|inch|in)', re.I),
]

def normalize_to_cm(value, unit):
    """Convert any length unit to cm for consistency."""
    value = float(str(value).replace(',', '.'))
    unit = unit.lower().strip().rstrip('"')
    if unit in ['mm']: return round(value / 10, 1)
    if unit in ['cm']: return round(value, 1)
    if unit in ['m', 'meter']: return round(value * 100, 1)
    if unit in ['in', 'inch', '"']: return round(value * 2.54, 1)
    return round(value, 1)

def normalize_weight(value, unit):
    """Normalize weight to Google-accepted format."""
    value = float(str(value).replace(',', '.'))
    unit = unit.lower().strip()
    if unit in ['kilogram']: unit = 'kg'
    if unit in ['gram']: unit = 'g'
    # Convert grams > 1000 to kg for cleanliness
    if unit == 'g' and value >= 1000:
        return f"{round(value/1000, 2)} kg"
    return f"{round(value, 2)} {unit}"

def _scan_for_dimensions(text):
    """Scan text for dimension data.
    Returns dict with keys: weight, length, width, height (values as strings or None)."""
    result = {'weight': None, 'length': None, 'width': None, 'height': None}
    text_str = str(text) if pd.notna(text) else ''
    if not text_str or text_str.lower() == 'nan':
        return result

    # Weight
    for pat in WEIGHT_PATTERNS:
        m = pat.search(text_str)
        if m:
            result['weight'] = normalize_weight(m.group(1), m.group(2))
            break

    # LxWxH (3D dimensions in one pattern)
    m = DIMENSION_LxWxH.search(text_str)
    if m:
        unit = m.group(4)
        result['length'] = f"{normalize_to_cm(m.group(1), unit)} cm"
        result['width'] = f"{normalize_to_cm(m.group(2), unit)} cm"
        result['height'] = f"{normalize_to_cm(m.group(3), unit)} cm"
        return result

    # LxW (2D dimensions)
    m = DIMENSION_LxW.search(text_str)
    if m:
        unit = m.group(3)
        result['length'] = f"{normalize_to_cm(m.group(1), unit)} cm"
        result['width'] = f"{normalize_to_cm(m.group(2), unit)} cm"

    # Individual dimensions (fill in what's missing)
    for patterns, key in [
        (LENGTH_PATTERNS, 'length'),
        (WIDTH_PATTERNS, 'width'),
        (HEIGHT_PATTERNS, 'height'),
    ]:
        if not result[key]:
            for pat in patterns:
                m = pat.search(text_str)
                if m:
                    result[key] = f"{normalize_to_cm(m.group(1), m.group(2))} cm"
                    break

    # Diameter → treat as both length and width
    if not result['length'] and not result['width']:
        for pat in DIAMETER_PATTERNS:
            m = pat.search(text_str)
            if m:
                dim = f"{normalize_to_cm(m.group(1), m.group(2))} cm"
                result['length'] = dim
                result['width'] = dim
                break

    return result

def extract_dimensions(title, description='', product_type='', labels='',
                       google_category='', bullet_points=''):
    """Extract dimensions using 4-layer cascade.
    Returns (dimensions_dict, source, confidence)."""

    # --- LAYER 1: Title ---
    dims = _scan_for_dimensions(title)
    if any(dims.values()):
        return dims, 'title', 'high'

    # --- LAYER 2: Description ---
    dims = _scan_for_dimensions(description)
    if any(dims.values()):
        return dims, 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        dims = _scan_for_dimensions(source_text)
        if any(dims.values()):
            return dims, source_name, 'medium'

    # --- LAYER 4: Nothing found ---
    return {'weight': None, 'length': None, 'width': None, 'height': None}, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

# Also check for existing weight/dimension columns
weight_col = next((c for c in df.columns if c in ['product weight', 'product_weight', 'gewicht', 'verzendgewicht', 'shipping weight', 'shipping_weight']), None)

weights, lengths, widths, heights = [], [], [], []
results = {'filled': 0, 'unresolved': 0}

for idx, row in df.iterrows():
    # If feed already has a weight column, use it as high-confidence source
    existing_weight = None
    if weight_col:
        w = str(row[weight_col]).strip() if pd.notna(row[weight_col]) else ''
        if w and w.lower() not in ['', 'nan', 'none']:
            existing_weight = w

    dims, source, confidence = extract_dimensions(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))

    # Use existing weight if available, otherwise extracted
    final_weight = existing_weight if existing_weight else (dims['weight'] or '')

    weights.append(final_weight)
    lengths.append(dims['length'] or '')
    widths.append(dims['width'] or '')
    heights.append(dims['height'] or '')

    if final_weight or dims['length'] or dims['width'] or dims['height']:
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'product_weight': weights,
    'product_length': lengths,
    'product_width': widths,
    'product_height': heights,
    'shipping_weight': weights,   # Same as product dimensions
    'shipping_length': lengths,
    'shipping_width': widths,
    'shipping_height': heights,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_dimensions.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with fill rates per dimension type. Flag products where only partial dimensions were found (e.g., weight but no LxWxH).

## Important guardrails

- Never overwrite existing values
- Shipping dimensions default to product dimensions unless separate shipping packaging is specified
- Don't confuse product volume (500ml) with product dimensions — a 500ml bottle has dimensions in cm, not ml
- Screen sizes (e.g., "27 inch monitor") are NOT product dimensions — that's a display diagonal
- Battery capacity (mAh, Wh) is NOT weight
- "30x20cm" in a product name could be the product itself or packaging — treat as product dimensions by default
- Convert all lengths to cm and all weights to kg/g for consistency
- mm values should be converted to cm (e.g., 200mm → 20 cm)

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- This skill outputs 9 columns: `id` + 4 product dimensions + 4 shipping dimensions
- Column names: `product_weight`, `product_length`, `product_width`, `product_height`, `shipping_weight`, `shipping_length`, `shipping_width`, `shipping_height`
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
