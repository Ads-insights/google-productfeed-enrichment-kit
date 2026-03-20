---
name: productfeed-size-system
description: Extract and fill missing size_system attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [size_system] values. Also triggers when user mentions "size system", "matensysteem invullen", "feed size system", or asks to enrich/complete/fix size system data. Use this skill even if the user just says "fill in the size system" or "fix my feed size system" with an uploaded file.
---

# Feed Size System Extractor

Fill empty `size_system` attributes in a Google Shopping product feed.

## Why this matters

The `[size_system]` attribute tells Google which sizing standard is used. Without it, Google can't properly match size filters. Required when `size` is set.

## Valid values

Google accepts: `AU`, `BR`, `CN`, `DE`, `EU`, `FR`, `IT`, `JP`, `MEX`, `UK`, `US`.

For Dutch webshops, the default is almost always `EU`.

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

**size_system is ONLY relevant for clothing and shoe sizing.** Products with dimension-based sizes (furniture: "140 x 200 cm", textiles: "50 x 70 cm"), generic sizes ("Einheitsgröße"), or capacity-based sizes ("500 ml") should NOT get a size_system value. Assigning `size_system=EU` to a wardrobe or a bedsheet is meaningless and adds noise.

The skill first checks whether the product's size is a recognizable garment or shoe size. If not, it leaves size_system empty regardless of other signals.

```python
import re

# ═══════════════════════════════════════════════════════════════
# SIZE SYSTEM: ONLY FOR GARMENT AND SHOE SIZING
# ═══════════════════════════════════════════════════════════════
#
# ⚠️ size_system tells Google which sizing STANDARD is used (EU, US, UK, etc.)
# This ONLY makes sense for:
#   - Letter sizes: S, M, L, XL, XXL → size_system tells Google it's EU S vs US S
#   - Numeric clothing sizes: 36, 38, 40, 42 → EU 38 ≠ US 38
#   - Shoe sizes: 39, 40, 41, 42 → EU 42 ≠ US 42
#
# It does NOT make sense for:
#   - Dimensions: "140 x 200 cm" (this is a measurement, not a sizing standard)
#   - Volumes: "500 ml" (not a garment size)
#   - Weights: "350 g/m²" (fabric weight, not a sizing standard)
#   - Generic: "Einheitsgröße" / "One size" (no system needed)
#   - Furniture: "2-Sitzer" (seat count, not a sizing standard)

# Recognizable garment/shoe size patterns
LETTER_SIZES = {'xxs', 'xs', 's', 'm', 'l', 'xl', 'xxl', 'xxxl', '2xl', '3xl', '4xl', '5xl'}
NUMERIC_CLOTHING_SIZES = {str(n) for n in range(28, 62)}  # 28-61 (clothing)
NUMERIC_SHOE_SIZES = {str(n) for n in range(16, 51)}      # 16-50 (shoes)
NUMERIC_SHOE_HALVES = {f"{n}.5" for n in range(16, 51)}   # half sizes

# GPC categories where size_system is relevant (apparel, shoes, accessories)
APPAREL_GPC_PREFIXES = {
    '166', '167', '168', '169', '170', '171', '172', '173', '174', '175',  # Apparel
    '176', '177', '178', '179', '180', '181', '182', '183', '184', '185',
    '186', '187', '188', '189', '190', '191', '192', '193', '194', '195',
    '196', '197', '198', '199', '200', '201', '202', '203', '204', '205',
    '206', '207', '208', '209', '210', '211', '212', '213', '214',
    '1580', '1581', '1582', '1583', '1584', '1585',  # Shoes
    '2302', '2303', '2304', '2305', '2306',  # Bathrobe/sleepwear
    '5322', '5323', '5324', '5325', '5326', '5327', '5328', '5329',  # Shoe accessories
}

def _is_garment_or_shoe_size(size_value):
    """Check if a size value looks like a garment or shoe size (not a dimension)."""
    if not size_value:
        return False
    size_lower = str(size_value).lower().strip()
    
    # Exclude dimension-like sizes (contain cm, mm, m, x, ml, g, kg, etc.)
    if re.search(r'\d+\s*(?:cm|mm|m\b|ml|cl|l\b|kg|g\b|mg)', size_lower):
        return False
    # Exclude "NxN" dimension patterns (e.g., "140 x 200")
    if re.search(r'\d+\s*x\s*\d+', size_lower):
        return False
    # Exclude generic sizes
    if size_lower in ('einheitsgröße', 'einheitsgroesse', 'one size', 'os', 'osfa', 'one size fits all', 'universal'):
        return False
    # Exclude furniture/capacity patterns
    if re.search(r'\d+\s*[-]?\s*(?:sitzer|personen?|pers\.?|zonen?|latten)', size_lower):
        return False
    
    # Check for letter sizes
    size_words = set(re.findall(r'[a-z0-9.]+', size_lower))
    if size_words & LETTER_SIZES:
        return True
    # Check for numeric clothing/shoe sizes (standalone number or with FR/EU suffix)
    size_nums = re.findall(r'\b(\d+(?:\.\d)?)\b', size_lower)
    for num in size_nums:
        if num in NUMERIC_CLOTHING_SIZES or num in NUMERIC_SHOE_SIZES or num in NUMERIC_SHOE_HALVES:
            return True
    # Check for FR/EU size patterns like "50/52 FR"
    if re.search(r'\d+\s*/\s*\d+\s*(?:fr|eu|us|uk)\b', size_lower, re.I):
        return True
    
    return False

SIZE_SYSTEM_MAP = {
    'eu': 'EU', 'eur': 'EU', 'europa': 'EU', 'european': 'EU', 'europees': 'EU', 'europäisch': 'EU',
    'us': 'US', 'usa': 'US', 'american': 'US', 'amerikaans': 'US', 'amerikanisch': 'US',
    'uk': 'UK', 'british': 'UK', 'brits': 'UK', 'britisch': 'UK',
    'fr': 'FR', 'french': 'FR', 'frans': 'FR', 'französisch': 'FR',
    'it': 'IT', 'italian': 'IT', 'italiaans': 'IT', 'italienisch': 'IT',
    'jp': 'JP', 'japan': 'JP', 'japanese': 'JP', 'japans': 'JP', 'japanisch': 'JP',
    'au': 'AU', 'australian': 'AU', 'australisch': 'AU',
    'cn': 'CN', 'chinese': 'CN', 'chinees': 'CN', 'chinesisch': 'CN',
}

def extract_size_system(title, description='', feed_label='', size_value='', google_product_category=''):
    """Extract size_system. Returns (system, source, confidence).
    
    ⚠️ ONLY assigns size_system when the product has a recognizable garment/shoe size.
    Products with dimension-based sizes (cm, mm), volumes, or generic sizes get NO size_system."""
    
    # GATE CHECK: Is this a garment/shoe size at all?
    if not _is_garment_or_shoe_size(size_value):
        return '', 'not a garment/shoe size', 'skipped'
    
    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''
    all_text = f"{title_lower} {desc_lower}"

    # Pass 1: Explicit size system mention (e.g., "EU maat 42", "US size 10", "FR 38")
    for keyword, system in SIZE_SYSTEM_MAP.items():
        pattern = rf'\b{re.escape(keyword)}\b'
        if re.search(pattern, all_text):
            if keyword == 'de' and not re.search(r'\bde\s+maat\b', all_text):
                continue
            return system, 'title/description', 'high'

    # Pass 2: Infer from feed label / language
    feed_label_lower = str(feed_label).lower() if pd.notna(feed_label) else ''
    if feed_label_lower in ['nl', 'be', 'de', 'at', 'fr', 'it', 'es', 'pt']:
        return 'EU', 'feed label (European country)', 'high'

    # Pass 3: Default to EU for European webshops
    return 'EU', 'default (European webshop)', 'medium'
```

### Step 3: Apply and build supplemental feed

```python
ss_col = next((c for c in df.columns if c in ['size_system', 'matensysteem', 'g:size_system']), None)
if ss_col is None:
    ss_col = 'size_system'
    df[ss_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)
fl_col = next((c for c in df.columns if c in ['feedlabel', 'feed_label', 'feed label']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[ss_col]).strip() if pd.notna(row[ss_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    system, source, confidence = extract_size_system(
        row.get(title_col, ''), row.get(desc_col, ''), row.get(fl_col, ''))
    if system:
        df.at[idx, ss_col] = system
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'size_system': df[ss_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_size_system.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- **Only assign size_system to products with garment or shoe sizes.** Dimension-based sizes (cm, mm, "140 x 200"), volumes (ml, l), generic sizes ("Einheitsgröße", "One size"), and furniture sizes ("2-Sitzer", "6 Personen") should NEVER get a size_system. An empty size_system is always safe.
- Never overwrite existing values
- Default `EU` is safe for NL/BE/DE webshops but ONLY when the size is actually a garment/shoe size
- Only fill size_system for products that actually have a recognizable garment/shoe size value — leave blank if size is a dimension, volume, or generic value

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `size_system`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
