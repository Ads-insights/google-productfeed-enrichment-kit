---
name: productfeed-is-bundle
description: Determine and fill the is_bundle attribute in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [is_bundle] values. Also triggers when user mentions "bundle attribute", "is pakket", "bundel", or asks about bundle detection. Use this skill even if the user just says "fill in the bundle flag" with an uploaded file.
---

# Feed Bundle Detector

Determine whether products are bundles (multiple different products sold together) and fill the `is_bundle` attribute.

## Why this matters

The `[is_bundle]` attribute tells Google the product is a merchant-defined bundle. Required when selling bundles to:
- Avoid disapprovals for price mismatches
- Ensure correct product representation
- Comply with Google's bundling policies

## Valid values

- `true` — product is a bundle of multiple different items
- `false` — product is a single item (default)

Note: a bundle is NOT a multipack. A bundle = different products together (e.g., camera + lens + bag). A multipack = same product multiple times (e.g., 3-pack socks).

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

Uses the 4-layer cascade scanning for bundle signals.

```python
import re

BUNDLE_KEYWORDS = {
    # Dutch
    'bundel', 'pakket', 'startpakket', 'starterset', 'compleet pakket',
    'complete set', 'actie-pakket', 'voordeelpakket', 'combideal',
    'combi-deal', 'combinatie', 'kit',
    # English
    'bundle', 'starter kit', 'starter pack', 'combo', 'combo deal',
    'complete kit', 'complete set', 'accessory kit', 'value pack',
}

BUNDLE_MULTI_WORD = {
    'starter kit', 'starter pack', 'startpakket', 'complete set',
    'compleet pakket', 'combo deal', 'combi deal', 'combi-deal',
    'accessory kit', 'value pack', 'voordeelpakket', 'actie-pakket',
}

def _scan_for_bundle(text):
    """Scan text for bundle signals. Returns True if found."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return False
    # Multi-word first
    for kw in BUNDLE_MULTI_WORD:
        if kw in text_lower:
            return True
    # Single-word
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    if words & BUNDLE_KEYWORDS:
        return True
    return False

def extract_is_bundle(title, description='', product_type='', labels='',
                      google_category='', bullet_points=''):
    """Extract is_bundle using 4-layer cascade. Returns (value, source, confidence)."""

    # --- LAYER 1: Title ---
    if _scan_for_bundle(title):
        return 'true', 'title', 'high'

    # --- LAYER 2: Description ---
    if _scan_for_bundle(description):
        return 'true', 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        if _scan_for_bundle(source_text):
            return 'true', source_name, 'medium'

    # --- LAYER 4: Default false ---
    return 'false', 'default (no bundle signals)', 'high'
```

### Step 3: Apply and build supplemental feed

```python
bundle_col = next((c for c in df.columns if c in ['is_bundle', 'is pakket', 'g:is_bundle', 'bundle']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

results = {'filled': 0, 'already_had': 0}
bundle_values = []

for idx, row in df.iterrows():
    if bundle_col:
        current = str(row[bundle_col]).strip() if pd.notna(row[bundle_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            bundle_values.append(current)
            continue

    val, source, confidence = extract_is_bundle(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))
    bundle_values.append(val)
    results['filled'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'is_bundle': bundle_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_is_bundle.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- Default is `false` — most products are not bundles
- Don't confuse bundles with multipacks: "3-pack sokken" = multipack, "camera + tas + statief" = bundle
- "Set" is ambiguous — a "messenset" could be a bundle or a single product. Flag for review.
- Never overwrite existing values

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `is_bundle`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
