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

```python
import re

SIZE_SYSTEM_MAP = {
    'eu': 'EU', 'eur': 'EU', 'europa': 'EU', 'european': 'EU', 'europees': 'EU',
    'us': 'US', 'usa': 'US', 'american': 'US', 'amerikaans': 'US',
    'uk': 'UK', 'british': 'UK', 'brits': 'UK',
    'fr': 'FR', 'french': 'FR', 'frans': 'FR',
    'it': 'IT', 'italian': 'IT', 'italiaans': 'IT',
    'jp': 'JP', 'japan': 'JP', 'japanese': 'JP', 'japans': 'JP',
    'au': 'AU', 'australian': 'AU', 'australisch': 'AU',
    'cn': 'CN', 'chinese': 'CN', 'chinees': 'CN',
}

def extract_size_system(title, description='', feed_label=''):
    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''
    all_text = f"{title_lower} {desc_lower}"

    # Pass 1: Explicit size system mention (e.g., "EU maat 42", "US size 10")
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

    # Pass 3: Default to EU for Dutch webshops
    return 'EU', 'default (Dutch webshop)', 'medium'
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

- Never overwrite existing values
- Default `EU` is safe for NL/BE webshops but flag this to the user
- Only fill size_system for products that actually have a size value — leave blank if size is also empty

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `size_system`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
