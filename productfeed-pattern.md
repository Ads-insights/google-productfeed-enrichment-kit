---
name: productfeed-pattern
description: Extract and fill missing pattern attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [pattern] values. Also triggers when user mentions "pattern attribute", "patroon invullen", "feed pattern", "missing patterns in feed", or asks to enrich/complete/fix pattern data. Use this skill even if the user just says "fill in the patterns" or "fix my feed patterns" with an uploaded file.
---

# Feed Pattern Extractor

Extract pattern values from product titles, descriptions, and other available fields to fill empty `pattern` attributes in a Google Shopping product feed.

## Why this matters

The `[pattern]` attribute is medium-impact in Google Shopping. Missing pattern data leads to:
- Missed filtered searches on Shopping tab (e.g., "gestreept overhemd")
- Lower relevance for pattern-specific queries
- Incomplete product data for variant grouping

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

### Step 2: Pattern vocabulary

```python
PATTERN_MAP = {
    # Solid/plain
    'effen': 'effen', 'solid': 'effen', 'plain': 'effen', 'uni': 'effen',

    # Stripes
    'gestreept': 'gestreept', 'strepen': 'gestreept', 'streep': 'gestreept',
    'striped': 'gestreept', 'stripes': 'gestreept', 'stripe': 'gestreept',
    'pinstripe': 'krijtstreep', 'krijtstreep': 'krijtstreep',

    # Checks & plaids
    'geruit': 'geruit', 'ruit': 'geruit', 'ruiten': 'geruit',
    'checked': 'geruit', 'checkered': 'geruit', 'plaid': 'geruit',
    'tartan': 'tartan', 'gingham': 'gingham',
    'pied-de-poule': 'pied-de-poule', 'houndstooth': 'pied-de-poule',

    # Dots
    'gestipt': 'gestipt', 'stippen': 'gestipt', 'stip': 'gestipt',
    'polka dot': 'gestipt', 'dotted': 'gestipt', 'dots': 'gestipt',

    # Floral
    'bloemen': 'bloemenprint', 'bloemenprint': 'bloemenprint', 'bloemetjes': 'bloemenprint',
    'floral': 'bloemenprint', 'flower': 'bloemenprint', 'flowers': 'bloemenprint',

    # Animal
    'luipaard': 'luipaardprint', 'luipaardprint': 'luipaardprint', 'leopard': 'luipaardprint',
    'zebra': 'zebraprint', 'zebraprint': 'zebraprint',
    'slangen': 'slangenprint', 'slangenprint': 'slangenprint', 'snake': 'slangenprint',
    'dierenprint': 'dierenprint', 'animal print': 'dierenprint',
    'koeienprint': 'koeienprint', 'cow print': 'koeienprint',
    'tijger': 'tijgerprint', 'tiger': 'tijgerprint',

    # Geometric
    'geometrisch': 'geometrisch', 'geometric': 'geometrisch',
    'abstract': 'abstract',
    'zigzag': 'zigzag', 'chevron': 'chevron',

    # Camouflage
    'camouflage': 'camouflage', 'camo': 'camouflage',

    # Paisley
    'paisley': 'paisley',

    # Marble
    'marmer': 'marmerpatroon', 'marble': 'marmerpatroon', 'gemarmerd': 'marmerpatroon',

    # Tropical
    'tropisch': 'tropisch', 'tropical': 'tropisch', 'palm': 'tropisch',

    # Other
    'tie-dye': 'tie-dye', 'batik': 'batik',
    'melange': 'melange', 'gemêleerd': 'melange',
    'gespikkeld': 'gespikkeld', 'speckled': 'gespikkeld',
    'krakele': 'krakele',
    'opaque': None,  # not a pattern
    'frosted': None,  # finish, not pattern
}

MULTI_WORD_PATTERNS = {
    'polka dot': 'gestipt',
    'animal print': 'dierenprint',
    'cow print': 'koeienprint',
    'tie dye': 'tie-dye',
}
```

### Step 3: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
import re
from collections import Counter

def _scan_for_pattern(text):
    """Scan text for pattern matches. Returns pattern or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    for pattern_str, pattern_val in MULTI_WORD_PATTERNS.items():
        if pattern_str in text_lower:
            return pattern_val
    words = re.findall(r'[a-zà-ÿ-]+', text_lower)
    for word in words:
        if word in PATTERN_MAP and PATTERN_MAP[word] is not None:
            return PATTERN_MAP[word]
    return None

def extract_pattern(title, description='', product_type='', labels='',
                    google_category='', bullet_points=''):
    """Extract pattern using 4-layer cascade. Returns (pattern, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    pat = _scan_for_pattern(title)
    if pat:
        return pat, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    pat = _scan_for_pattern(description)
    if pat:
        return pat, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        pat = _scan_for_pattern(source_text)
        if pat:
            return pat, source_name, 'medium'

    # --- LAYER 4: No pattern found ---
    return None, 'none', 'unresolved'
```

### Step 4: Apply and build supplemental feed

```python
pattern_col = next((c for c in df.columns if c in ['pattern', 'patroon', 'g:pattern']), None)
if pattern_col is None:
    pattern_col = 'pattern'
    df[pattern_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[pattern_col]).strip() if pd.notna(row[pattern_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    pat, source, confidence = extract_pattern(row.get(title_col, ''), row.get(desc_col, ''))
    if pat:
        df.at[idx, pattern_col] = pat
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'pattern': df[pattern_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_pattern.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with fill rate before/after, sample of filled values, and unresolved list.

## Important guardrails

- Never overwrite existing pattern values
- Many products have no pattern (electronics, plain items) — leave blank, don't guess "effen"
- "Marble" can be a material OR a pattern — context matters. In titles like "Marble Earth" it's likely a product name, not a pattern
- Watch for false positives in product names (e.g., "Tiger" as brand, "Stripe" as payment provider)

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `pattern`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
