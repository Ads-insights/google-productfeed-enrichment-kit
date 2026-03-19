---
name: productfeed-gender
description: Extract and fill missing gender attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [gender] values. Also triggers when user mentions "gender attribute", "geslacht invullen", "feed gender", "missing gender in feed", or asks to enrich/complete/fix gender data in a product feed. Use this skill even if the user just says "fill in the gender" or "fix my feed gender" with an uploaded file.
---

# Feed Gender Extractor

Extract gender values from product titles, descriptions, and other available fields to fill empty `gender` attributes in a Google Shopping product feed.

## Why this matters

The `[gender]` attribute is high-impact in Google Shopping. Missing gender data leads to:
- Disapproved products in apparel categories (gender is required for clothing)
- Missed gendered search queries (e.g., "damesjurk", "heren overhemd")
- Products shown to wrong audience, wasting ad spend
- Poor Shopping tab filtering

## Valid values

Google only accepts these exact values for `gender`:
- `male`
- `female`
- `unisex`

Always output one of these three English values, regardless of the feed language.

## Workflow

### Step 1: Load and inspect the feed

```python
import pandas as pd

# Load file (handle ZIP, CSV, TSV, Excel)
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

### Step 2: Gender vocabulary

```python
# Direct gender indicators
GENDER_MALE = {
    'heren', 'heer', 'man', 'mannen', 'mannelijk',
    'men', 'mens', 'male', 'boy', 'boys',
    'jongens', 'jongen', 'him', 'his',
    'gentleman', 'gentlemen',
}

GENDER_FEMALE = {
    'dames', 'dame', 'vrouw', 'vrouwen', 'vrouwelijk',
    'women', 'womens', "women's", 'female', 'lady', 'ladies',
    'girl', 'girls', 'meisjes', 'meisje', 'her', 'hers',
}

GENDER_UNISEX = {
    'unisex', 'genderneutraal', 'gender-neutral',
}

# Product type indicators (strong signals)
MALE_PRODUCT_TYPES = {
    'herenoverhemden', 'herenkleding', 'herenbroek', 'herenjas',
    'herenschoenen', 'herenhorloge', 'herenpolo', 'herenboxer',
    'stropdas', 'manchetknopen',
}

FEMALE_PRODUCT_TYPES = {
    'dameskleding', 'damesjurk', 'damestop', 'damesbroek',
    'damesjas', 'damesschoenen', 'jurk', 'jurken',
    'rok', 'rokken', 'beha', 'bh', 'legging', 'panty',
    'bikini', 'tankini', 'badpak',
}

# Compound word patterns (Dutch compound words)
MALE_PREFIXES = ['heren', 'mannen', 'jongens']
FEMALE_PREFIXES = ['dames', 'vrouwen', 'meisjes']
```

### Step 3: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
import re

def _scan_for_gender(text):
    """Scan text for gender signals. Returns 'male', 'female', 'unisex', or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    # Check compound words
    has_male = bool(words & GENDER_MALE) or bool(words & MALE_PRODUCT_TYPES)
    has_female = bool(words & GENDER_FEMALE) or bool(words & FEMALE_PRODUCT_TYPES)
    for word in words:
        for prefix in MALE_PREFIXES:
            if word.startswith(prefix) and len(word) > len(prefix):
                has_male = True
        for prefix in FEMALE_PREFIXES:
            if word.startswith(prefix) and len(word) > len(prefix):
                has_female = True
    if words & GENDER_UNISEX: return 'unisex'
    if has_male and has_female: return 'unisex'
    if has_male: return 'male'
    if has_female: return 'female'
    return None

def extract_gender(title, description='', product_type='', labels='',
                   google_category='', bullet_points=''):
    """Extract gender using 4-layer cascade. Returns (gender, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    gender = _scan_for_gender(title)
    if gender:
        return gender, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    gender = _scan_for_gender(description)
    if gender:
        return gender, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        gender = _scan_for_gender(source_text)
        if gender:
            return gender, source_name, 'medium'

    # --- LAYER 4: No gender found ---
    return None, 'none', 'unresolved'
```

### Step 4: Apply and build supplemental feed

```python
gender_col = next((c for c in df.columns if c in ['gender', 'geslacht', 'g:gender']), None)
if gender_col is None:
    gender_col = 'gender'
    df[gender_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[gender_col]).strip() if pd.notna(row[gender_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    gender, source, confidence = extract_gender(
        row.get(title_col, ''), row.get(desc_col, ''), row.get(ptype_col, ''))
    if gender:
        df.at[idx, gender_col] = gender
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'gender': df[gender_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_gender.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with fill rate before/after, sample of filled values, and unresolved list.

## Important guardrails

- Gender values are ALWAYS in English: `male`, `female`, or `unisex`
- Never overwrite existing values
- Many product categories don't need gender (electronics, furniture, food) — leave blank, don't guess
- When both male and female signals are found, default to `unisex`
- Product type column is a strong signal — use it alongside title

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `gender`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
