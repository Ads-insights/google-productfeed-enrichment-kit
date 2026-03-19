---
name: productfeed-material
description: Extract and fill missing material attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [material] values. Also triggers when user mentions "material attribute", "materiaal invullen", "feed material", "missing materials in feed", or asks to enrich/complete/fix material data in a product feed. Use this skill even if the user just says "fill in the materials" or "fix my feed materials" with an uploaded file.
---

# Feed Material Extractor

Extract material values from product titles, descriptions, and other available fields to fill empty `material` attributes in a Google Shopping product feed.

## Why this matters

The `[material]` attribute is high-impact in Google Shopping. Missing material data leads to:
- Missed search queries containing material terms (e.g., "leren schoenen", "katoenen shirt")
- Lower ad relevance for material-specific searches
- Disapproved products in categories where material is required (apparel)
- Poor filtering experience on Shopping tab

## Workflow

### Step 1: Load and inspect the feed

Read the uploaded file (CSV, TSV, ZIP containing TSV, or Excel). The material column may be named any of: `material`, `materiaal`, `Material`, `g:material`. Also locate source columns: `title`/`titel`, `description`/`beschrijving`, `product_type`/`producttype`.

```python
import pandas as pd

# Detect file type and load (handle ZIP containing TSV)
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

Report: total products, how many have empty material, which source columns are available.

### Step 2: Material vocabulary

Support Dutch and English. Match case-insensitively.

```python
MATERIAL_MAP = {
    # Textiles
    'katoen': 'katoen', 'cotton': 'katoen', 'katoenen': 'katoen',
    'polyester': 'polyester',
    'nylon': 'nylon',
    'zijde': 'zijde', 'silk': 'zijde', 'zijden': 'zijde',
    'wol': 'wol', 'wool': 'wol', 'wollen': 'wol',
    'linnen': 'linnen', 'linen': 'linnen',
    'denim': 'denim', 'jeans': 'denim',
    'fleece': 'fleece',
    'viscose': 'viscose', 'rayon': 'viscose',
    'elastaan': 'elastaan', 'elastane': 'elastaan', 'spandex': 'elastaan', 'lycra': 'elastaan',
    'modal': 'modal',
    'cashemere': 'kasjmier', 'cashmere': 'kasjmier', 'kasjmier': 'kasjmier',
    'velours': 'velours', 'velvet': 'velours', 'fluweel': 'velours', 'fluwelen': 'velours',
    'satijn': 'satijn', 'satin': 'satijn', 'satijnen': 'satijn',
    'chiffon': 'chiffon',
    'jersey': 'jersey',
    'tweed': 'tweed',
    'canvas': 'canvas',
    'mesh': 'mesh',
    'gore-tex': 'gore-tex', 'goretex': 'gore-tex',
    'microvezel': 'microvezel', 'microfiber': 'microvezel',

    # Leather & skins
    'leer': 'leer', 'leather': 'leer', 'leren': 'leer',
    'kunstleer': 'kunstleer', 'pu-leer': 'kunstleer', 'vegan leather': 'kunstleer', 'kunstleren': 'kunstleer',
    'suède': 'suède', 'suede': 'suède',
    'nubuck': 'nubuck',

    # Metals
    'rvs': 'rvs', 'edelstaal': 'rvs', 'stainless steel': 'rvs',
    'staal': 'staal', 'steel': 'staal', 'stalen': 'staal',
    'aluminium': 'aluminium', 'aluminum': 'aluminium',
    'messing': 'messing', 'brass': 'messing',
    'koper': 'koper', 'copper': 'koper', 'koperen': 'koper',
    'titanium': 'titanium',
    'ijzer': 'ijzer', 'iron': 'ijzer',
    'goud': 'goud', 'gold': 'goud', 'gouden': 'goud',
    'zilver': 'zilver', 'silver': 'zilver', 'zilveren': 'zilver',

    # Wood
    'hout': 'hout', 'wood': 'hout', 'houten': 'hout',
    'bamboe': 'bamboe', 'bamboo': 'bamboe',
    'eiken': 'eikenhout', 'oak': 'eikenhout', 'eikenhout': 'eikenhout',
    'walnoot': 'walnoothout', 'walnut': 'walnoothout',
    'teak': 'teakhout', 'teakhout': 'teakhout',
    'grenen': 'grenenhout', 'pine': 'grenenhout',
    'beuken': 'beukenhout', 'beech': 'beukenhout',
    'mdf': 'mdf',
    'multiplex': 'multiplex', 'plywood': 'multiplex',

    # Glass & ceramics
    'glas': 'glas', 'glass': 'glas', 'glazen': 'glas',
    'kristalglas': 'kristalglas', 'crystal glass': 'kristalglas', 'kristal': 'kristalglas',
    'keramiek': 'keramiek', 'ceramic': 'keramiek', 'keramisch': 'keramiek', 'keramische': 'keramiek',
    'porselein': 'porselein', 'porcelain': 'porselein',
    'aardewerk': 'aardewerk', 'earthenware': 'aardewerk', 'pottery': 'aardewerk',

    # Plastics & synthetics
    'plastic': 'kunststof', 'kunststof': 'kunststof',
    'siliconen': 'siliconen', 'silicone': 'siliconen',
    'rubber': 'rubber',
    'acryl': 'acryl', 'acrylic': 'acryl',
    'pvc': 'pvc',
    'abs': 'abs',
    'polycarbonaat': 'polycarbonaat', 'polycarbonate': 'polycarbonaat',
    'polypropyleen': 'polypropyleen', 'polypropylene': 'polypropyleen',

    # Natural materials
    'kurk': 'kurk', 'cork': 'kurk',
    'jute': 'jute',
    'riet': 'riet', 'rattan': 'rattan', 'rotan': 'rattan',
    'marmer': 'marmer', 'marble': 'marmer', 'marmeren': 'marmer',
    'graniet': 'graniet', 'granite': 'graniet',
    'steen': 'steen', 'stone': 'steen', 'stenen': 'steen',
    'beton': 'beton', 'concrete': 'beton',
    'papier': 'papier', 'paper': 'papier',
    'karton': 'karton', 'cardboard': 'karton',
    'leer': 'leer', 'leather': 'leer',

    # Composites
    'carbon': 'carbon', 'koolstofvezel': 'carbon', 'carbon fiber': 'carbon',
    'glasvezel': 'glasvezel', 'fiberglass': 'glasvezel',
}

# Multi-word materials
MULTI_WORD_MATERIALS = {
    'stainless steel': 'rvs',
    'carbon fiber': 'carbon',
    'vegan leather': 'kunstleer',
    'crystal glass': 'kristalglas',
    'solid wood': 'massief hout',
    'massief hout': 'massief hout',
    'gehard glas': 'gehard glas',
    'tempered glass': 'gehard glas',
    'recycled plastic': 'gerecycled kunststof',
    'pu leer': 'kunstleer',
    'pu-leer': 'kunstleer',
    'engobe keramiek': 'keramiek',
}
```

### Step 3: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
import re
from collections import Counter

def _scan_for_material(text):
    """Scan a text string for material matches. Returns material or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    # Multi-word first
    for pattern, mat in MULTI_WORD_MATERIALS.items():
        if pattern in text_lower:
            return mat
    # Single-word
    words = re.findall(r'[a-zà-ÿ-]+', text_lower)
    for word in words:
        if word in MATERIAL_MAP and MATERIAL_MAP[word] is not None:
            return MATERIAL_MAP[word]
    return None

def extract_material(title, description='', product_type='', labels='',
                     google_category='', bullet_points=''):
    """Extract material using 4-layer cascade. Returns (material, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    mat = _scan_for_material(title)
    if mat:
        return mat, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    mat = _scan_for_material(description)
    if mat:
        return mat, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        mat = _scan_for_material(source_text)
        if mat:
            return mat, source_name, 'medium'

    # --- LAYER 4: No material found ---
    return None, 'none', 'unresolved'
```

### Step 4: Apply and build supplemental feed

```python
material_col = next((c for c in df.columns if c in ['material', 'materiaal', 'g:material']), None)
if material_col is None:
    material_col = 'material'
    df[material_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[material_col]).strip() if pd.notna(row[material_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    material, source, confidence = extract_material(
        row.get(title_col, ''), row.get(desc_col, ''))
    if material:
        df.at[idx, material_col] = material
        results['filled'] += 1
    else:
        results['unresolved'] += 1

# Build supplemental feed
supplemental = pd.DataFrame({
    'id': df[id_col],
    'material': df[material_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_material.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with fill rate before/after, sample of filled values for spot-checking, and list of unresolved products.

## Important guardrails

- Never overwrite existing material values
- Always output to a new file, never modify the original
- Flag uncertainty honestly — leave blank rather than guess
- Watch for false positives: "glass" in "sunglasses" is not a material, "canvas" in "Canvas Sneaker" could be either
- Some products genuinely have no material (digital goods, services) — leave blank

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `material`
- **Multiple attributes** (combined with other feed-attribute skills): 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet for use as supplemental feed
- Include ALL products (also empty ones, so the user can see what still needs manual work)
