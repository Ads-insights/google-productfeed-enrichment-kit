---
name: productfeed-condition
description: Extract and fill missing condition attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [condition] values. Also triggers when user mentions "condition attribute", "staat invullen", "feed condition", "missing condition", or asks to enrich/complete/fix condition data. Use this skill even if the user just says "fill in the condition" or "fix my feed condition" with an uploaded file.
---

# Feed Condition Extractor

Extract condition values from product titles, descriptions, and other available fields to fill empty `condition` attributes in a Google Shopping product feed.

## Why this matters

The `[condition]` attribute is required for all products in Google Shopping. Missing condition leads to:
- Product disapprovals
- Incorrect expectations for buyers (new vs used)
- Issues with price benchmarking (used items priced differently)

## Valid values

Google only accepts these exact values for `condition`:
- `new`
- `refurbished`
- `used`

Always output one of these three English values.

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

### Step 2: Condition vocabulary

```python
CONDITION_NEW = {
    'nieuw', 'nieuwe', 'new', 'gloednieuw', 'brand new', 'splinternieuw',
}

CONDITION_REFURBISHED = {
    'refurbished', 'gereviseerd', 'gereviseerde', 'renewed', 'reconditioned',
    'gerenoveerd', 'hersteld', 'opgeknapt', 'certified pre-owned',
    'factory refurbished',
}

CONDITION_USED = {
    'gebruikt', 'gebruikte', 'used', 'tweedehands', 'tweede hands',
    'second hand', 'secondhand', 'occasion', 'pre-owned', 'vintage',
    'antiek', 'antique',
}

MULTI_WORD_CONDITIONS = {
    'brand new': 'new',
    'certified pre-owned': 'refurbished',
    'factory refurbished': 'refurbished',
    'second hand': 'used',
    'tweede hands': 'used',
}
```

### Step 3: Extraction algorithm

```python
import re

def extract_condition(title, description='', condition_existing=''):
    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''

    all_text = f"{title_lower} {desc_lower}"

    # Pass 1: Multi-word patterns
    for pattern, cond in MULTI_WORD_CONDITIONS.items():
        if pattern in all_text:
            return cond, 'title/description (multi-word)', 'high'

    # Pass 2: Direct word match
    words = set(re.findall(r'[a-zà-ÿ-]+', all_text))

    if words & CONDITION_REFURBISHED:
        return 'refurbished', 'title/description', 'high'
    if words & CONDITION_USED:
        return 'used', 'title/description', 'high'
    if words & CONDITION_NEW:
        return 'new', 'title/description', 'high'

    # Pass 3: Default to 'new' for standard webshops
    # Most webshops sell new products. If no signal is found,
    # default to 'new' with medium confidence.
    # This is a reasonable default for most Dutch webshops.
    return 'new', 'default (no used/refurbished signals)', 'medium'
```

### Step 4: Apply and build supplemental feed

```python
cond_col = next((c for c in df.columns if c in ['condition', 'staat', 'g:condition']), None)
if cond_col is None:
    cond_col = 'condition'
    df[cond_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[cond_col]).strip() if pd.notna(row[cond_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    cond, source, confidence = extract_condition(
        row.get(title_col, ''), row.get(desc_col, ''))
    if cond:
        df.at[idx, cond_col] = cond
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'condition': df[cond_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_condition.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with fill rate before/after. Specifically highlight any products detected as `used` or `refurbished` since those are the exception.

## Important guardrails

- Condition values are ALWAYS in English: `new`, `refurbished`, `used`
- Never overwrite existing values
- Defaulting to `new` is safe for most webshops — but flag this to the user so they can verify
- `vintage` maps to `used`, not a separate category
- If the webshop sells primarily used/refurbished items, ask the user what the default should be before applying

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `condition`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
