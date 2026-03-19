---
name: productfeed-identifier-exists
description: Determine and fill the identifier_exists attribute in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [identifier_exists] values. Also triggers when user mentions "identifier exists", "id bestaat", or asks about GTIN/MPN presence. Use this skill even if the user just says "fill in identifier exists" with an uploaded file.
---

# Feed Identifier Exists

Determine whether products have valid unique product identifiers (GTIN, MPN, brand) and fill the `identifier_exists` attribute accordingly.

## Why this matters

The `[identifier_exists]` attribute tells Google whether you have unique product identifiers. Setting it incorrectly leads to:
- Disapprovals if set to `true` but GTIN/MPN is missing or invalid
- Missed Shopping eligibility if set to `false` when identifiers are available
- Lower ad quality and matching when identifiers exist but aren't declared

## Valid values

- `true` — product has a GTIN, MPN, or brand (at least one)
- `false` — product has NO standard identifiers (custom/handmade/vintage items)

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

No cascade needed — this is a pure data check across multiple columns.

```python
def has_value(val):
    """Check if a cell has a meaningful value."""
    if pd.isna(val):
        return False
    s = str(val).strip().lower()
    return s not in ['', 'nan', 'none', '0', 'false']

def extract_identifier_exists(row, gtin_col, mpn_col, brand_col):
    """Check if product has at least one valid identifier."""
    has_gtin = has_value(row.get(gtin_col)) if gtin_col else False
    has_mpn = has_value(row.get(mpn_col)) if mpn_col else False
    has_brand = has_value(row.get(brand_col)) if brand_col else False

    if has_gtin or has_mpn or has_brand:
        return 'true', 'data check', 'high'
    return 'false', 'no identifiers found', 'high'
```

### Step 3: Apply and build supplemental feed

```python
# Find relevant columns
gtin_col = next((c for c in df.columns if c in ['gtin', 'ean', 'upc', 'isbn', 'barcode']), None)
mpn_col = next((c for c in df.columns if c in ['mpn', 'manufacturer_part_number']), None)
brand_col = next((c for c in df.columns if c in ['brand', 'merk', 'manufacturer', 'g:brand']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
ie_col = next((c for c in df.columns if c in ['identifier_exists', 'id bestaat', 'g:identifier_exists']), None)

results = {'filled': 0, 'already_had': 0}

id_exists_values = []
for idx, row in df.iterrows():
    if ie_col:
        current = str(row[ie_col]).strip() if pd.notna(row[ie_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            id_exists_values.append(current)
            continue

    val, source, confidence = extract_identifier_exists(row, gtin_col, mpn_col, brand_col)
    id_exists_values.append(val)
    results['filled'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'identifier_exists': id_exists_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_identifier_exists.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- This is a data check, not text extraction — confidence is always high
- If GTIN exists but looks invalid (wrong length, all zeros), still set to `true` but warn the user
- Custom/handmade products without EAN/MPN should be `false`
- When in doubt, `true` is safer — Google will flag invalid GTINs separately

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `identifier_exists`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
