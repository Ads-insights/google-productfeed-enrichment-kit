---
name: productfeed-age-group
description: Extract and fill missing age_group attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [age_group] values. Also triggers when user mentions "age group attribute", "leeftijdsgroep invullen", "feed age group", "missing age group", or asks to enrich/complete/fix age group data. Use this skill even if the user just says "fill in the age groups" or "fix my feed age groups" with an uploaded file.
---

# Feed Age Group Extractor

Extract age_group values from product titles, descriptions, and other available fields to fill empty `age_group` attributes in a Google Shopping product feed.

## Why this matters

The `[age_group]` attribute is medium-high impact in Google Shopping. Missing age_group data leads to:
- Disapproved products in apparel categories (required for clothing)
- Products shown to wrong age demographics
- Missed filtered searches on Shopping tab
- Poor audience targeting in Smart Shopping campaigns

## Valid values

Google only accepts these exact values for `age_group`:
- `newborn` (0-3 months)
- `infant` (3-12 months)
- `toddler` (1-5 years)
- `kids` (5-13 years)
- `adult` (13+ years)

Always output one of these five English values.

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

### Step 2: Age group vocabulary

```python
AGE_NEWBORN = {
    'newborn', 'pasgeboren', 'pasgeborene', '0-3 maanden', 'newborn',
}

AGE_INFANT = {
    'infant', 'baby', "baby's", 'babykleding', 'zuigeling',
    '3-12 maanden', '6-12 maanden',
}

AGE_TODDLER = {
    'toddler', 'peuter', 'peuters', 'peuterkleding', 'dreumes',
    '1-5 jaar', '2-4 jaar',
}

AGE_KIDS = {
    'kids', 'kind', 'kinderen', 'kinder', 'junior', 'junioren',
    'jongens', 'meisjes', 'boys', 'girls', 'children', 'child',
    'kinderkleding', 'kinderschoenen', '5-13 jaar', '6-12 jaar',
}

AGE_ADULT = {
    'adult', 'volwassen', 'volwassenen', 'heren', 'dames',
    'men', 'women', 'man', 'vrouw',
}

# Compound word prefixes
KIDS_PREFIXES = ['kinder', 'baby', 'peuter', 'junior']
ADULT_PREFIXES = ['heren', 'dames', 'mannen', 'vrouwen']
```

### Step 3: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
import re

def _scan_for_age_group(text):
    """Scan text for age group signals. Returns age group or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    # Check compound words
    for word in words:
        for prefix in KIDS_PREFIXES:
            if word.startswith(prefix) and len(word) > len(prefix):
                return 'kids'
        for prefix in ADULT_PREFIXES:
            if word.startswith(prefix) and len(word) > len(prefix):
                return 'adult'
    # Priority order: most specific first
    if words & AGE_NEWBORN: return 'newborn'
    if words & AGE_INFANT: return 'infant'
    if words & AGE_TODDLER: return 'toddler'
    if words & AGE_KIDS: return 'kids'
    if words & AGE_ADULT: return 'adult'
    return None

def extract_age_group(title, description='', product_type='', labels='',
                      google_category='', bullet_points=''):
    """Extract age_group using 4-layer cascade. Returns (age_group, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    age = _scan_for_age_group(title)
    if age:
        return age, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    age = _scan_for_age_group(description)
    if age:
        return age, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        age = _scan_for_age_group(source_text)
        if age:
            return age, source_name, 'medium'

    # --- LAYER 4: No age group found ---
    return None, 'none', 'unresolved'
```

### Step 4: Apply and build supplemental feed

```python
age_col = next((c for c in df.columns if c in ['age_group', 'leeftijdsgroep', 'g:age_group']), None)
if age_col is None:
    age_col = 'age_group'
    df[age_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[age_col]).strip() if pd.notna(row[age_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    age, source, confidence = extract_age_group(
        row.get(title_col, ''), row.get(desc_col, ''), row.get(ptype_col, ''))
    if age:
        df.at[idx, age_col] = age
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'age_group': df[age_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_age_group.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with fill rate before/after, sample of filled values, and unresolved list.

## Important guardrails

- Age group values are ALWAYS in English: `newborn`, `infant`, `toddler`, `kids`, `adult`
- Never overwrite existing values
- Most products for a typical webshop are `adult` — but don't blindly assign this without evidence
- Categories like electronics, furniture, food typically don't need age_group — leave blank
- When both kids and adult signals appear, prefer `kids` (more restrictive/specific)

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `age_group`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
