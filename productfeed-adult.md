---
name: productfeed-adult
description: Determine and fill the adult attribute in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [adult] values. Also triggers when user mentions "adult attribute", "inhoud voor volwassenen", "18+", or asks about adult content flagging. Use this skill even if the user just says "fill in the adult flag" with an uploaded file.
---

# Feed Adult Flag

Determine whether products contain adult/mature content and fill the `adult` attribute accordingly.

## Why this matters

The `[adult]` attribute is required when products contain sexually suggestive or adult content. Incorrect flagging leads to:
- Account suspension if adult products are not flagged
- Unnecessary restriction of non-adult products if over-flagged
- Policy violations in Google Merchant Center

## Valid values

- `true` — product contains adult/mature content
- `false` — product is suitable for general audiences (default for most products)

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

Uses the 4-layer cascade scanning for adult content signals.

```python
import re

ADULT_KEYWORDS = {
    # Dutch
    'erotisch', 'erotische', 'erotiek', 'sekspeeltje', 'seksspeeltje',
    'vibrator', 'dildo', 'lingerie', 'fetish', 'bondage',
    'sekspop', 'lustopwekkend', 'volwassen speelgoed',
    'condoom', 'condooms', 'glijmiddel',
    # English
    'erotic', 'adult toy', 'sex toy', 'vibrator', 'dildo',
    'lingerie', 'fetish', 'bondage', 'adult only', '18+',
    'sensual', 'intimate massager', 'pleasure',
}

ADULT_CATEGORIES = {
    'erotiek', 'adult', 'seksualiteit', 'intiem', 'intimate',
    'erotic', 'sex toys', 'adult toys',
}

def _scan_for_adult(text):
    """Scan text for adult content signals. Returns True if found."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return False
    words = set(re.findall(r'[a-zà-ÿ0-9+-]+', text_lower))
    if words & ADULT_KEYWORDS:
        return True
    for kw in ADULT_KEYWORDS:
        if ' ' in kw and kw in text_lower:
            return True
    return False

def extract_adult(title, description='', product_type='', labels='',
                  google_category='', bullet_points=''):
    """Extract adult flag using 4-layer cascade. Returns (value, source, confidence)."""

    # --- LAYER 1: Title ---
    if _scan_for_adult(title):
        return 'true', 'title', 'high'

    # --- LAYER 2: Description ---
    if _scan_for_adult(description):
        return 'true', 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        if _scan_for_adult(source_text):
            return 'true', source_name, 'medium'

    # --- LAYER 4: Default false ---
    return 'false', 'default (no adult signals)', 'high'
```

### Step 3: Apply and build supplemental feed

```python
adult_col = next((c for c in df.columns if c in ['adult', 'inhoud voor volwassenen', 'g:adult']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

results = {'filled': 0, 'already_had': 0}
adult_values = []

for idx, row in df.iterrows():
    if adult_col:
        current = str(row[adult_col]).strip() if pd.notna(row[adult_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            adult_values.append(current)
            continue

    val, source, confidence = extract_adult(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))
    adult_values.append(val)
    results['filled'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'adult': adult_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_adult.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- Default is `false` — most webshops don't sell adult products
- When adult signals are found, flag the products to the user for verification before submitting
- "Lingerie" is borderline — some lingerie is non-adult. Flag as `true` to be safe but mention to user
- Never overwrite existing values

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `adult`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
