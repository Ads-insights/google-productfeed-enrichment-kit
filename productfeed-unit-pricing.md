---
name: productfeed-unit-pricing
description: Extract and fill missing unit_pricing_measure and unit_pricing_base_measure attributes in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [unit_pricing_measure] or [unit_pricing_base_measure] values. Also triggers when user mentions "unit pricing", "eenheidsprijs", "prijs per eenheid", "unit measure", or asks about unit pricing data. Use this skill even if the user just says "fill in unit pricing" with an uploaded file.
---

# Feed Unit Pricing Extractor

Extract unit pricing measure and base measure values from product titles, descriptions, and other fields to fill `unit_pricing_measure` and `unit_pricing_base_measure` attributes.

## Why this matters

The `[unit_pricing_measure]` and `[unit_pricing_base_measure]` attributes are required in many EU countries (including the Netherlands) for products sold by weight, volume, length, or area. Missing unit pricing leads to:
- Legal non-compliance with EU price indication directives
- Product disapprovals in Merchant Center
- Missing price-per-unit comparisons for shoppers

## Valid formats

**unit_pricing_measure**: The quantity + unit of the product.
Format: `<number> <unit>`, e.g., `500ml`, `1.5kg`, `250g`, `2l`, `100cm`, `10m2`

**unit_pricing_base_measure**: The standard unit for price comparison.
Format: `<number> <unit>`, e.g., `100ml`, `1kg`, `100g`, `1l`, `1m`, `1m2`

### Accepted units by Google:

**Weight**: `oz`, `lb`, `mg`, `g`, `kg`
**Volume**: `floz`, `pt`, `qt`, `gal`, `ml`, `cl`, `l`, `cbm`
**Length**: `in`, `ft`, `yd`, `cm`, `m`
**Area**: `sqft`, `sqm` / `m2`
**Per unit**: `ct` (count)

### Standard base measures per unit type:
| Product unit | Base measure |
|---|---|
| ml, cl | 100ml |
| l | 1l |
| g, mg | 100g |
| kg | 1kg |
| cm | 1m |
| m | 1m |
| m2, sqm | 1m2 |
| ct (count) | 1ct |

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

# Patterns to find quantity + unit combinations
# Order matters: check longer/more specific patterns first

UNIT_PATTERNS = [
    # Volume patterns
    re.compile(r'(\d+(?:[.,]\d+)?)\s*(?:milli)?liter', re.I),  # "500 milliliter", "1.5 liter"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*ml\b', re.I),             # "500ml"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*cl\b', re.I),             # "75cl"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*l\b', re.I),              # "1.5l", "2l"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*fl\.?\s*oz', re.I),       # "12 fl oz"

    # Weight patterns
    re.compile(r'(\d+(?:[.,]\d+)?)\s*kilogram', re.I),         # "2.5 kilogram"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*kg\b', re.I),             # "2.5kg"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*gram\b', re.I),           # "500 gram"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*gr?\b', re.I),            # "500g", "250gr"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*mg\b', re.I),             # "100mg"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*oz\b', re.I),             # "16oz"
    re.compile(r'(\d+(?:[.,]\d+)?)\s*lb\b', re.I),             # "5lb"

    # ⚠️ Length units (cm, mm, m) and area units (m², sqm) are EXCLUDED from unit pricing.
    # These are product DIMENSIONS, not consumable measures. Unit pricing is only for products
    # sold by weight, volume, or count — e.g., food, beverages, cosmetics, cleaning products.
    # A bedsheet that is "140 x 200 cm" does not have a unit price per meter.
    # A bottle of shampoo that is "500 ml" DOES have a unit price per 100ml.

    # Count patterns
    re.compile(r'(\d+)\s*(?:stuks?|pieces?|tabs?|capsules?|tabletten?)', re.I),  # "30 stuks"
]

# Map matched unit text to Google-accepted unit code
def normalize_unit(text_after_number):
    """Convert matched unit text to Google-accepted unit code."""
    t = text_after_number.lower().strip()
    if t in ['ml', 'milliliter']: return 'ml'
    if t in ['cl']: return 'cl'
    if t in ['l', 'liter']: return 'l'
    if t in ['fl oz', 'fl. oz', 'floz']: return 'floz'
    if t in ['kg', 'kilogram']: return 'kg'
    if t in ['g', 'gr', 'gram']: return 'g'
    if t in ['mg']: return 'mg'
    if t in ['oz']: return 'oz'
    if t in ['lb']: return 'lb'
    # ⚠️ Length units (cm, mm, m) and area units (m², sqm) are NOT valid for unit pricing.
    # They represent product dimensions, not consumable measures. Returning None here
    # ensures these are never used as unit_pricing_measure values.
    if t in ['cm', 'mm', 'm', 'meter', 'm²', 'm2', 'sqm']: return None
    if t in ['stuks', 'stuk', 'pieces', 'piece', 'tabs', 'tab',
             'capsules', 'capsule', 'tabletten', 'tablet']: return 'ct'
    return None

# Map unit to standard base measure
BASE_MEASURE_MAP = {
    'ml': '100ml', 'cl': '100ml', 'l': '1l', 'floz': '1floz',
    'g': '100g', 'mg': '100g', 'kg': '1kg', 'oz': '1oz', 'lb': '1lb',
    # ⚠️ No length/area units — these are product dimensions, not unit pricing
    'ct': '1ct',
}

FULL_MEASURE_PATTERN = re.compile(
    r'(\d+(?:[.,]\d+)?)\s*'
    r'(milliliter|liter|kilogram|gram|'
    r'ml|cl|l|fl\.?\s*oz|'
    r'kg|gr?|mg|oz|lb'
    r'|stuks?|pieces?|tabs?|capsules?|tabletten?)',
    re.I
)

METRIC_MEASURE_PATTERN = re.compile(
    r'(\d+(?:[.,]\d+)?)\s*'
    r'(milliliter|liter|kilogram|gram|ml|cl|l|kg|gr?|mg'
    r'|stuks?|pieces?|tabs?|capsules?|tabletten?)',
    re.I
)

IMPERIAL_MEASURE_PATTERN = re.compile(
    r'(\d+(?:[.,]\d+)?)\s*'
    r'(fl\.?\s*oz|oz|lb)',
    re.I
)

def _scan_for_unit_pricing(text):
    """Scan text for unit pricing. Returns (measure, base_measure) or (None, None).
    ALWAYS prioritizes metric units (g, ml, kg, l) over imperial (oz, lb, fl oz).
    Many titles contain both: '2.5 oz (70 g)' — we want the metric value."""
    text_str = str(text) if pd.notna(text) else ''
    if not text_str or text_str.lower() == 'nan':
        return None, None

    # Try metric FIRST
    match = METRIC_MEASURE_PATTERN.search(text_str)
    if match:
        number = match.group(1).replace(',', '.')
        unit_text = match.group(2)
        unit = normalize_unit(unit_text)
        if unit:
            measure = f"{number}{unit}"
            base = BASE_MEASURE_MAP.get(unit)
            return measure, base

    # Fallback to imperial only if no metric found
    match = IMPERIAL_MEASURE_PATTERN.search(text_str)
    if match:
        number = match.group(1).replace(',', '.')
        unit_text = match.group(2)
        unit = normalize_unit(unit_text)
        if unit:
            measure = f"{number}{unit}"
            base = BASE_MEASURE_MAP.get(unit)
            return measure, base

    return None, None

def extract_unit_pricing(title, description='', product_type='', labels='',
                         google_category='', bullet_points=''):
    """Extract unit pricing using 4-layer cascade.
    Returns (measure, base_measure, source, confidence)."""

    # --- LAYER 1: Title ---
    measure, base = _scan_for_unit_pricing(title)
    if measure:
        return measure, base, 'title', 'high'

    # --- LAYER 2: Description ---
    measure, base = _scan_for_unit_pricing(description)
    if measure:
        return measure, base, 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        measure, base = _scan_for_unit_pricing(source_text)
        if measure:
            return measure, base, source_name, 'medium'

    # --- LAYER 4: No unit pricing found ---
    return None, None, 'none', 'unresolved'
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

upm_col = next((c for c in df.columns if c in ['unit_pricing_measure', 'eenheidsprijs hoeveelheid']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}
measures = []
bases = []

for idx, row in df.iterrows():
    if upm_col:
        current = str(row[upm_col]).strip() if pd.notna(row[upm_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            measures.append(current)
            bases.append('')  # Keep existing, don't override base
            continue

    measure, base, source, confidence = extract_unit_pricing(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))

    measures.append(measure if measure else '')
    bases.append(base if base else '')
    if measure:
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'unit_pricing_measure': measures,
    'unit_pricing_base_measure': bases,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_unit_pricing.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with fill rate, sample values, and highlight any products where the unit might be ambiguous (e.g., "m" could be meter or minutes in some contexts).

## Important guardrails

- **NEVER use length or area units (cm, mm, m, m², sqm) as unit_pricing_measure.** These are product dimensions, not consumable measures. A "140 x 200 cm" bedsheet does not have a price per meter. Only weight (g, mg, kg, oz, lb), volume (ml, cl, l, floz), and count (ct/stuks/pieces) are valid unit pricing measures. This is the single most common source of false positives in unit pricing extraction.
- **ALWAYS prioritize metric units** (g, ml, kg, l) over imperial (oz, lb, fl oz). If both are present in the title (e.g., "2.5 oz (70 g)"), use the metric value. This is critical for EU/NL compliance.
- Never overwrite existing values
- "m" standalone is ambiguous — could be meters or minutes. Only match when context suggests measurement
- Don't match model numbers as quantities (e.g., "D2S" is not 2 seconds)
- Capacity measures for electronics (mAh, Wh) are NOT unit pricing measures
- The comma/period decimal separator varies: "1,5l" and "1.5l" are both valid
- Products without a physical measure (software, services, digital goods) should be left empty
- When measure is in ml, base should be 100ml. When measure is in liters, base should be 1l.
- Google requires the number and unit to be together without spaces in the submitted value

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- This skill outputs 3 columns: `id` + `unit_pricing_measure` + `unit_pricing_base_measure`
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
