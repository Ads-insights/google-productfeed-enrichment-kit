---
name: productfeed-energy-efficiency
description: Extract and fill missing energy_efficiency_class, min_energy_efficiency_class, and max_energy_efficiency_class attributes in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate energy efficiency values. Also triggers when user mentions "energy efficiency", "energielabel", "energie-efficiëntie", "energy class", or asks about energy labels. Use this skill even if the user just says "fill in the energy labels" with an uploaded file.
---

# Feed Energy Efficiency Extractor

Extract energy efficiency class values from product titles, descriptions, and other fields to fill `energy_efficiency_class`, `min_energy_efficiency_class`, and `max_energy_efficiency_class` attributes.

## Why this matters

Energy efficiency labels are legally required in the EU for many product categories. Missing energy data leads to:
- Legal non-compliance with EU energy labelling regulations
- Product disapprovals in Merchant Center for affected categories
- Missing energy label annotations on Shopping ads

## Valid values

Google accepts these values (current EU scale since March 2021):
- `A`, `B`, `C`, `D`, `E`, `F`, `G`

Legacy scale (still accepted for some categories):
- `A+++`, `A++`, `A+`, `A`, `B`, `C`, `D`, `E`, `F`, `G`

## Applicable product categories

Energy labels are required/relevant for:
- Washing machines, dryers, washer-dryers
- Refrigerators, freezers, wine coolers
- Dishwashers
- TVs, monitors, electronic displays
- Light bulbs, LED lamps
- Air conditioners, heaters
- Ovens, range hoods
- Vacuum cleaners (removed from EU labelling but still in some feeds)

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

# Valid energy classes in order from best to worst
VALID_CLASSES = ['A+++', 'A++', 'A+', 'A', 'B', 'C', 'D', 'E', 'F', 'G']

# Patterns to find energy class mentions
ENERGY_PATTERNS = [
    # "Energielabel A++", "Energy class A", "Energie-efficiëntieklasse B"
    re.compile(r'(?:energielabel|energy\s*class|energie[-\s]*efficiëntie[-\s]*klasse|energieklasse|eec)\s*[:=]?\s*(A\+{0,3}|[B-G])\b', re.I),
    # "klasse A++", "class A+"
    re.compile(r'(?:klasse|class|label)\s*[:=]?\s*(A\+{0,3}|[B-G])\b', re.I),
    # Standalone pattern with context: "(A++)" or "[A+]"
    re.compile(r'[\(\[]\s*(A\+{0,3}|[B-G])\s*[\)\]]', re.I),
    # "EEK: A+", "EEI: B"
    re.compile(r'EE[KI]\s*[:=]\s*(A\+{0,3}|[B-G])\b', re.I),
]

# Product categories likely to have energy labels
ENERGY_CATEGORIES_NL = {
    'wasmachine', 'wasdroger', 'droger', 'vaatwasser', 'koelkast',
    'vriezer', 'diepvries', 'koelvries', 'wijnkoeler', 'wijnklimaatkast',
    'televisie', 'tv', 'monitor', 'display', 'beeldscherm',
    'lamp', 'ledlamp', 'gloeilamp', 'verlichting', 'tl-buis',
    'airco', 'airconditioner', 'warmtepomp', 'verwarming', 'boiler',
    'oven', 'fornuis', 'afzuigkap', 'dampkap',
    'stofzuiger',
}

ENERGY_CATEGORIES_EN = {
    'washing machine', 'tumble dryer', 'dryer', 'dishwasher',
    'refrigerator', 'fridge', 'freezer', 'wine cooler',
    'television', 'tv', 'monitor', 'display',
    'lamp', 'led', 'light bulb', 'lighting',
    'air conditioner', 'heat pump', 'heater', 'boiler',
    'oven', 'range hood', 'cooker hood',
    'vacuum cleaner',
}

def _scan_for_energy_class(text):
    """Scan text for energy efficiency class. Returns class string or None."""
    text_str = str(text) if pd.notna(text) else ''
    if not text_str or text_str.lower() == 'nan':
        return None
    for pattern in ENERGY_PATTERNS:
        match = pattern.search(text_str)
        if match:
            cls = match.group(1)
            # Normalize: uppercase A, correct plus signs
            cls = cls.upper().replace(' ', '')
            if cls in [c.upper() for c in VALID_CLASSES]:
                return cls
    return None

def _is_energy_relevant_category(product_type, google_category, title):
    """Check if product is in a category that typically has energy labels."""
    all_text = f"{str(product_type)} {str(google_category)} {str(title)}".lower()
    for cat in ENERGY_CATEGORIES_NL | ENERGY_CATEGORIES_EN:
        if cat in all_text:
            return True
    return False

def extract_energy_efficiency(title, description='', product_type='', labels='',
                               google_category='', bullet_points=''):
    """Extract energy efficiency using 4-layer cascade.
    Returns (class, min_class, max_class, source, confidence)."""

    # --- LAYER 1: Title ---
    cls = _scan_for_energy_class(title)
    if cls:
        return cls, 'G', cls, 'title', 'high'

    # --- LAYER 2: Description ---
    cls = _scan_for_energy_class(description)
    if cls:
        return cls, 'G', cls, 'description', 'medium'

    # --- LAYER 3: Other columns ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        cls = _scan_for_energy_class(source_text)
        if cls:
            return cls, 'G', cls, source_name, 'medium'

    # --- LAYER 4: No energy class found ---
    return None, None, None, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

ee_values = []
min_values = []
max_values = []
results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    cls, min_cls, max_cls, source, confidence = extract_energy_efficiency(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))

    if cls:
        ee_values.append(cls)
        min_values.append(min_cls)
        max_values.append(max_cls)
        results['filled'] += 1
    else:
        ee_values.append('')
        min_values.append('')
        max_values.append('')
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'energy_efficiency_class': ee_values,
    'min_energy_efficiency_class': min_values,
    'max_energy_efficiency_class': max_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_energy_efficiency.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary. Also flag if products appear to be in energy-relevant categories but have no energy class detected — these may need manual review.

## Important guardrails

- Never overwrite existing values
- Only flag energy class for relevant product categories — a t-shirt doesn't need an energy label
- "A" alone is ambiguous in text — only match when preceded by energy-related context words
- The EU rescaled energy labels in March 2021: A+++ through D became A through G. Both scales are valid.
- min_energy_efficiency_class is typically `G` (worst possible on the scale)
- max_energy_efficiency_class matches the product's own class
- When multiple classes are mentioned (e.g., "A to G scale"), pick the one that refers to this specific product

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- This skill outputs 4 columns: `id` + `energy_efficiency_class` + `min_energy_efficiency_class` + `max_energy_efficiency_class`
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
