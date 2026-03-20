---
name: productfeed-size-type
description: Extract and fill missing size_type attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [size_type] values. Also triggers when user mentions "size type", "maattype invullen", "feed size type", or asks to enrich/complete/fix size type data. Use this skill even if the user just says "fill in the size type" or "fix my feed size type" with an uploaded file.
---

# Feed Size Type Extractor

Fill empty `size_type` attributes in a Google Shopping product feed.

## Why this matters

The `[size_type]` attribute tells Google what kind of sizing a product uses. Helps Google show the right size filters and match queries accurately.

## Valid values

Google accepts:
- `regular` — standard sizing (vast majority of products)
- `petite` — smaller/shorter cuts
- `plus` — plus size / large sizes
- `tall` — taller/longer cuts
- `big` — big sizes (men's)
- `maternity` — maternity sizing

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

**size_type is ONLY relevant for clothing and shoe sizing.** It tells Google whether the sizing is regular, plus, petite, tall, or maternity. Assigning `size_type=regular` to a bookshelf or a bedsheet is meaningless.

The skill first checks whether the product's size is a recognizable garment or shoe size (using the same gate as the size_system skill). If not, it leaves size_type empty.

```python
import re

# ═══════════════════════════════════════════════════════════════
# SIZE TYPE: ONLY FOR GARMENT AND SHOE SIZING
# ═══════════════════════════════════════════════════════════════
#
# ⚠️ size_type tells Google what KIND of sizing: regular, plus, petite, tall, maternity.
# This ONLY makes sense for clothing and shoes — not for furniture, textiles, or homewares.

# Recognizable garment/shoe size patterns (same logic as size_system skill)
LETTER_SIZES = {'xxs', 'xs', 's', 'm', 'l', 'xl', 'xxl', 'xxxl', '2xl', '3xl', '4xl', '5xl'}
NUMERIC_CLOTHING_SIZES = {str(n) for n in range(28, 62)}
NUMERIC_SHOE_SIZES = {str(n) for n in range(16, 51)}
NUMERIC_SHOE_HALVES = {f"{n}.5" for n in range(16, 51)}

def _is_garment_or_shoe_size(size_value):
    """Check if a size value looks like a garment or shoe size (not a dimension)."""
    if not size_value:
        return False
    size_lower = str(size_value).lower().strip()
    
    if re.search(r'\d+\s*(?:cm|mm|m\b|ml|cl|l\b|kg|g\b|mg)', size_lower):
        return False
    if re.search(r'\d+\s*x\s*\d+', size_lower):
        return False
    if size_lower in ('einheitsgröße', 'einheitsgroesse', 'one size', 'os', 'osfa', 'one size fits all', 'universal'):
        return False
    if re.search(r'\d+\s*[-]?\s*(?:sitzer|personen?|pers\.?|zonen?|latten)', size_lower):
        return False
    
    size_words = set(re.findall(r'[a-z0-9.]+', size_lower))
    if size_words & LETTER_SIZES:
        return True
    size_nums = re.findall(r'\b(\d+(?:\.\d)?)\b', size_lower)
    for num in size_nums:
        if num in NUMERIC_CLOTHING_SIZES or num in NUMERIC_SHOE_SIZES or num in NUMERIC_SHOE_HALVES:
            return True
    if re.search(r'\d+\s*/\s*\d+\s*(?:fr|eu|us|uk)\b', size_lower, re.I):
        return True
    
    return False

SIZE_TYPE_KEYWORDS = {
    'petite': {'petite', 'petit', 'kort model', 'korte maat', 'kurz'},
    'plus': {'plus size', 'plussize', 'plus-size', 'grote maat', 'grote maten',
             'grosse grösse', 'grosse größe', 'curvy'},
    'tall': {'tall', 'lang model', 'lange maat', 'extra lang'},
    'big': {'big', 'grote maat heren', 'big size'},
    'maternity': {'maternity', 'zwangerschap', 'zwangerschaps', 'positiekleding',
                  'positie', 'zwangerschapskleding', 'schwangerschaft', 'umstandsmode'},
}
# ⚠️ 'xxl', 'xxxl', '3xl', '4xl', '5xl' are REMOVED from plus-size detection.
# These are regular letter sizes, not indicators of plus-size LINES.
# A "XXL T-shirt" is a regular-sized product in size XXL, not a plus-size product.
# Plus-size detection should only trigger on explicit "plus size" / "grote maat" keywords.

def extract_size_type(title, description='', product_type='', size_value=''):
    """Extract size_type. Returns (size_type, source, confidence).
    
    ⚠️ ONLY assigns size_type when the product has a recognizable garment/shoe size."""
    
    # GATE CHECK: Is this a garment/shoe size at all?
    if not _is_garment_or_shoe_size(size_value):
        return '', 'not a garment/shoe size', 'skipped'
    
    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''
    ptype_lower = str(product_type).lower() if pd.notna(product_type) else ''
    all_text = f"{title_lower} {desc_lower} {ptype_lower}"

    # Check for special size types
    for size_type, keywords in SIZE_TYPE_KEYWORDS.items():
        for keyword in keywords:
            if keyword in all_text:
                return size_type, 'title/description/product_type', 'high'

    # Default to regular (only for confirmed garment/shoe products)
    return 'regular', 'default', 'medium'
```

### Step 3: Apply and build supplemental feed

```python
st_col = next((c for c in df.columns if c in ['size_type', 'maattype', 'g:size_type']), None)
if st_col is None:
    st_col = 'size_type'
    df[st_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[st_col]).strip() if pd.notna(row[st_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    stype, source, confidence = extract_size_type(
        row.get(title_col, ''), row.get(desc_col, ''), row.get(ptype_col, ''))
    if stype:
        df.at[idx, st_col] = stype
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'size_type': df[st_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_size_type.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- **Only assign size_type to products with garment or shoe sizes.** A wardrobe with size "Einheitsgröße" or a bedsheet with size "140 x 200 cm" should NEVER get `size_type=regular`. An empty size_type is always safe for non-apparel products.
- Never overwrite existing values
- `regular` is the safe default for 95%+ of clothing/shoe products
- **Do NOT use XXL/XXXL/3XL as plus-size indicators.** These are standard letter sizes. Only explicit "plus size" / "grote maat" / "grosse Größe" keywords should trigger `plus`.
- The plus-size detection should not trigger on standalone "xl" in non-clothing contexts
- Only fill for products that have actual clothing/shoes sizing — leave blank for electronics, furniture, home textiles, etc.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `size_type`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
