---
name: productfeed-multipack
description: Extract and fill missing multipack attributes in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [multipack] values. Also triggers when user mentions "multipack attribute", "multipack invullen", "verpakkingseenheid", or asks about multipack detection. Use this skill even if the user just says "fill in the multipack" with an uploaded file.
---

# Feed Multipack Detector

Extract multipack quantities from product titles and descriptions to fill the `multipack` attribute.

## Why this matters

The `[multipack]` attribute tells Google how many identical items are in the package. Required when selling multipacks to:
- Ensure correct price-per-unit comparisons
- Avoid disapprovals for price mismatches vs single items
- Improve ad accuracy

## Valid values

A whole number indicating how many identical items: `2`, `3`, `4`, `6`, `12`, etc.
Leave empty if the product is a single item (which is the vast majority).

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

Uses the 4-layer cascade scanning for multipack patterns.

```python
import re

# Patterns that indicate a multipack with a number
MULTIPACK_PATTERNS = [
    # "3-pack", "6pack", "3 pack"
    re.compile(r'(\d+)\s*[-]?\s*pack', re.I),
    # "set van 3", "set of 6"
    re.compile(r'set\s+(?:van|of)\s+(\d+)', re.I),
    # "3 stuks", "6 stuks"
    re.compile(r'(\d+)\s+stuks?', re.I),
    # "3-delig", "6-delig"
    re.compile(r'(\d+)\s*[-]?\s*delig', re.I),
    # "3 pieces", "6 pieces"
    re.compile(r'(\d+)\s+pieces?', re.I),
    # "3x", "6x" (standalone, not part of resolution like "1920x1080" or dosage like "3x daily")
    re.compile(r'\b(\d+)x\b(?!\d)(?!\s*(?:daily|daags|dagelijks|per dag|times))', re.I),
    # "per 3", "per 6"
    re.compile(r'per\s+(\d+)', re.I),
    # "3-er set", "6-er pack"
    re.compile(r'(\d+)\s*[-]?\s*er\s+(?:set|pack)', re.I),
    # "verpakking van 3"
    re.compile(r'verpakking\s+van\s+(\d+)', re.I),
    # "duo" = 2, "trio" = 3
]

WORD_MULTIPACKS = {
    'duo': '2', 'duopack': '2', 'dubbelpak': '2', 'tweepak': '2',
    'trio': '3', 'triopack': '3', 'driepack': '3',
}

def _scan_for_multipack(text):
    """Scan text for multipack signals. Returns quantity string or None."""
    text_str = str(text) if pd.notna(text) else ''
    text_lower = text_str.lower()
    if not text_lower or text_lower == 'nan':
        return None

    # Check word-based multipacks first
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    for word, qty in WORD_MULTIPACKS.items():
        if word in words:
            return qty

    # Check regex patterns
    for pattern in MULTIPACK_PATTERNS:
        match = pattern.search(text_lower)
        if match:
            qty = match.group(1)
            # Sanity check: multipack is typically 2-100
            qty_int = int(qty)
            if 2 <= qty_int <= 100:
                return qty
    return None

def extract_multipack(title, description='', product_type='', labels='',
                      google_category='', bullet_points=''):
    """Extract multipack using 4-layer cascade. Returns (quantity, source, confidence)."""

    # --- LAYER 1: Title ---
    qty = _scan_for_multipack(title)
    if qty:
        return qty, 'title', 'high'

    # --- LAYER 2: Description ---
    qty = _scan_for_multipack(description)
    if qty:
        return qty, 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        qty = _scan_for_multipack(source_text)
        if qty:
            return qty, source_name, 'medium'

    # --- LAYER 4: Not a multipack (leave empty) ---
    return None, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
mp_col = next((c for c in df.columns if c in ['multipack', 'g:multipack']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}
mp_values = []

for idx, row in df.iterrows():
    if mp_col:
        current = str(row[mp_col]).strip() if pd.notna(row[mp_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            mp_values.append(current)
            continue

    qty, source, confidence = extract_multipack(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))
    mp_values.append(qty if qty else '')
    if qty:
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'multipack': mp_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_multipack.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- Most products are NOT multipacks — leave empty (not "1")
- Don't confuse multipacks with bundles: "3-pack sokken" (same item x3) = multipack. "Camera + lens + tas" = bundle.
- "Set" is ambiguous: "messenset" is usually a bundle, "sokken 3-pack" is a multipack
- Watch for false positives: "3x zoom" is not a multipack, "1920x1080" is not a multipack
- Sanity check: quantities should be 2-100. Anything higher is likely a false positive.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `multipack`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
