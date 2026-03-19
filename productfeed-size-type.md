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

```python
import re

SIZE_TYPE_KEYWORDS = {
    'petite': {'petite', 'petit', 'kort model', 'korte maat'},
    'plus': {'plus size', 'plussize', 'plus-size', 'grote maat', 'grote maten',
             'curvy', 'xxl', 'xxxl', '3xl', '4xl', '5xl'},
    'tall': {'tall', 'lang model', 'lange maat', 'extra lang'},
    'big': {'big', 'grote maat heren', 'big size'},
    'maternity': {'maternity', 'zwangerschap', 'zwangerschaps', 'positiekleding',
                  'positie', 'zwangerschapskleding'},
}

def extract_size_type(title, description='', product_type=''):
    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''
    ptype_lower = str(product_type).lower() if pd.notna(product_type) else ''
    all_text = f"{title_lower} {desc_lower} {ptype_lower}"

    # Check for special size types
    for size_type, keywords in SIZE_TYPE_KEYWORDS.items():
        for keyword in keywords:
            if keyword in all_text:
                return size_type, 'title/description/product_type', 'high'

    # Default to regular
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

- Never overwrite existing values
- `regular` is the safe default for 95%+ of products
- Only fill for products that have clothing/shoes — leave blank for electronics, furniture, etc.
- The plus-size detection should not trigger on standalone "xl" in non-clothing contexts

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `size_type`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
