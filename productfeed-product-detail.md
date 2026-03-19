---
name: productfeed-product-detail
description: Generate product_detail attributes for Google Shopping product feeds. Creates structured technical specifications in section_name:attribute_name:attribute_value format. Triggers when user uploads a product feed and wants to populate [product_detail] values. Also triggers when user mentions "product detail", "product details", "technische specificaties", "specifications", or asks about product specs. Use this skill even if the user just says "fill in the product details" or "add specifications" with an uploaded file.
---

# Feed Product Detail Generator

Generate `product_detail` values — structured technical specifications that help Google match products to specific search queries.

## Why this matters

The `[product_detail]` attribute is recommended and high-impact in Google Shopping:
- Provides readable, structured data for technical specifications
- Enhances Google's ability to match individual products to specific queries
- Appears as structured specs in product listings
- Covers technical details not captured by other attributes (memory, capacity, dimensions, compatibility, etc.)
- Particularly valuable for electronics, appliances, and technical products

## Google's specifications

**Format**: Each product_detail has 3 sub-attributes:
- `section_name` (optional but recommended) — groups related specs together
- `attribute_name` (required) — the spec name
- `attribute_value` (required) — the spec value

**In text/TSV feeds**: `section_name:attribute_name:attribute_value`
- Multiple details separated by commas
- If no section_name, start with colon: `:attribute_name:attribute_value`
- If a value contains commas or colons, enclose in double quotes
- Example: `General:Product Type:Digital player,Display:Resolution:432 x 240`

**Limits**:
- Section name: up to 140 characters
- Attribute name: up to 140 characters
- Attribute value: up to 1,000 characters
- Up to 1,000 product details per product (repeated field)

**Key rules:**
- Use sentence case ("Focal length" not "focal length" or "FOCAL LENGTH")
- Only provide confirmed values — don't guess specs
- No promotional text, pricing, shipping, or company info
- No SEO keywords
- No duplicate information within the attribute
- Don't repeat information already in other attributes (color, size, material, etc.)

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

Report: total products, how many already have product_detail, which source columns are available.

### Step 2: Product detail generation strategy

Product details are structured technical specs. The approach:

**Layer 1: Extract from existing structured data in the feed**
Use known attributes (material, size, weight, brand, condition, etc.) to build product_detail entries for specs not already in other attributes.

**Layer 2: Extract from description**
Parse the description for technical specifications using pattern matching (e.g., "Capaciteit: 500ml", "Gewicht: 2.5 kg", "Afmetingen: 30x20x15cm").

**Layer 3: Extract from bullet points / labels**
Many feeds have bullet point columns or label fields with structured product info.

**Layer 4: Infer from title and product type**
Extract technical identifiers from the title (model numbers, capacities, versions).

```python
import re

# Patterns to extract structured specs from text
SPEC_PATTERNS = [
    # "Capaciteit: 500ml" or "Capacity: 500ml"
    (re.compile(r'(?:capaciteit|capacity|inhoud|volume)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Capacity'),
    # "Gewicht: 2.5 kg" or "Weight: 2.5kg"
    (re.compile(r'(?:gewicht|weight|wt)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Weight'),
    # "Afmetingen: 30x20x15 cm" or "Dimensions: 30x20x15cm"
    (re.compile(r'(?:afmetingen|dimensions?|maten|lxbxh|l\s*x\s*b\s*x\s*h)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Dimensions'),
    # "Vermogen: 1500W" or "Power: 1500W" or "Wattage: 1500W"
    (re.compile(r'(?:vermogen|power|wattage|watt)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Power'),
    # "Voltage: 220V" or "Spanning: 220V"
    (re.compile(r'(?:spanning|voltage)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Electrical', 'Voltage'),
    # "Batterij: 5000mAh" or "Battery: 5000mAh"
    (re.compile(r'(?:batterij|battery|accu)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Battery', 'Capacity'),
    # "Schermformaat: 6.7 inch" or "Screen size: 6.7"
    (re.compile(r'(?:scherm(?:formaat|grootte)?|screen\s*size|display)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Display', 'Size'),
    # "Resolutie: 1920x1080" or "Resolution: 4K"
    (re.compile(r'(?:resolutie|resolution)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Display', 'Resolution'),
    # "Geheugen: 256GB" or "Memory: 8GB RAM" or "Opslag: 512GB"
    (re.compile(r'(?:geheugen|memory|ram|opslag|storage)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Memory', 'Storage'),
    # "Bereik: 50km" or "Range: 50km"
    (re.compile(r'(?:bereik|range|actieradius)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Performance', 'Range'),
    # "Snelheid: 25 km/h" or "Speed: 25 km/h"
    (re.compile(r'(?:(?:max(?:imale)?\.?\s*)?snelheid|speed)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'Performance', 'Max speed'),
    # "Compatibel met: iOS, Android"
    (re.compile(r'(?:compatibel\s*met|compatible\s*with|geschikt\s*voor|suitable\s*for)\s*[:=]\s*(.+?)(?:[.\n]|$)', re.I),
     'Compatibility', 'Compatible with'),
    # "Garantie: 2 jaar" or "Warranty: 2 years"
    (re.compile(r'(?:garantie|warranty)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Warranty'),
    # "Certificering: CE" or "Certification: CE"
    (re.compile(r'(?:certificering|certification|keurmerk)\s*[:=]\s*(.+?)(?:[,.\n]|$)', re.I),
     'General', 'Certification'),
]

# Inline spec patterns (no colon separator, found in running text)
INLINE_SPEC_PATTERNS = [
    # "5000mAh batterij" or "5000mAh battery"
    (re.compile(r'(\d+)\s*mAh', re.I), 'Battery', 'Capacity', '{0} mAh'),
    # "256GB opslag" or "512GB storage"
    (re.compile(r'(\d+)\s*(?:GB|TB)\s*(?:opslag|storage|ssd|hdd)?', re.I), 'Memory', 'Storage', '{0}'),
    # "IP67" or "IP68" waterproof rating
    (re.compile(r'\b(IP\d{2})\b', re.I), 'General', 'Water resistance', '{0}'),
    # "Bluetooth 5.0" or "WiFi 6"
    (re.compile(r'\b(Bluetooth\s*\d[\d.]*)\b', re.I), 'Connectivity', 'Bluetooth', '{0}'),
    (re.compile(r'\b(Wi-?Fi\s*\d[\w]*)\b', re.I), 'Connectivity', 'WiFi', '{0}'),
    # "USB-C" or "USB Type-C"
    (re.compile(r'\b(USB[-\s]?(?:C|Type[-\s]?C))\b', re.I), 'Connectivity', 'Port', '{0}'),
]


def extract_specs_from_text(text):
    """Extract structured specs from a text string.
    Returns list of (section_name, attribute_name, attribute_value) tuples."""
    specs = []
    text_str = str(text) if pd.notna(text) else ''
    if not text_str or text_str.lower() == 'nan':
        return specs

    # Try structured patterns first (with colon separator)
    for pattern, section, attr_name in SPEC_PATTERNS:
        match = pattern.search(text_str)
        if match:
            value = match.group(1).strip().rstrip(',.')
            if value and len(value) < 200:
                specs.append((section, attr_name, value))

    # Try inline patterns
    for pattern, section, attr_name, template in INLINE_SPEC_PATTERNS:
        match = pattern.search(text_str)
        if match:
            value = template.format(match.group(0))
            # Check we don't already have this spec
            existing_attrs = [(s, a) for s, a, v in specs]
            if (section, attr_name) not in existing_attrs:
                specs.append((section, attr_name, value))

    return specs


def build_specs_from_attributes(context):
    """Build product_detail entries from known feed attributes.
    Only include specs NOT already covered by standard attributes."""
    specs = []

    # Volume/capacity from title (for products like urns, bottles)
    title = context.get('title', '')
    vol_match = re.search(r'(\d+(?:[.,]\d+)?)\s*(?:ml|l|liter|L)\b', str(title), re.I)
    if vol_match:
        specs.append(('General', 'Capacity', vol_match.group(0).strip()))

    # Model number from title
    model_match = re.search(r'\b([A-Z][A-Z0-9]+-?[A-Z0-9]+)\b', str(title))
    if model_match:
        model = model_match.group(1)
        # Filter out common false positives
        if len(model) >= 3 and model not in ['USB', 'LED', 'LCD', 'RGB', 'NFC']:
            specs.append(('General', 'Model', model))

    return specs


def generate_product_details(title, description='', product_type='', labels='',
                             google_category='', bullet_points='', brand='',
                             existing_attrs=None):
    """Generate product_detail using the 4-layer cascade.
    Returns (details_string, source, confidence).
    details_string is in the format: section:attr_name:attr_value,section:attr_name:attr_value"""

    all_specs = []
    sources = []

    # Build context
    context = {'title': title, 'brand': brand}

    # --- LAYER 1: Extract from known attributes ---
    attr_specs = build_specs_from_attributes(context)
    if attr_specs:
        all_specs.extend(attr_specs)
        sources.append('attributes')

    # --- LAYER 2: Extract from description ---
    desc_specs = extract_specs_from_text(description)
    if desc_specs:
        all_specs.extend(desc_specs)
        sources.append('description')

    # --- LAYER 3: Extract from bullet points / labels ---
    for extra_text in [bullet_points, labels]:
        extra_specs = extract_specs_from_text(extra_text)
        if extra_specs:
            all_specs.extend(extra_specs)
            sources.append('bullet_points/labels')

    # --- LAYER 4: Extract from title ---
    title_specs = extract_specs_from_text(title)
    if title_specs:
        # Only add specs not already found
        existing_keys = [(s, a) for s, a, v in all_specs]
        for spec in title_specs:
            if (spec[0], spec[1]) not in existing_keys:
                all_specs.append(spec)
                sources.append('title')

    if not all_specs:
        return '', 'none', 'unresolved'

    # Deduplicate (keep first occurrence)
    seen = set()
    unique_specs = []
    for section, attr_name, attr_value in all_specs:
        key = (section, attr_name)
        if key not in seen:
            seen.add(key)
            unique_specs.append((section, attr_name, attr_value))

    # Format as Google's TSV format: section_name:attribute_name:attribute_value
    # Use sentence case for attribute names
    formatted = []
    for section, attr_name, attr_value in unique_specs[:10]:  # Max 10 details per product
        # Escape values with commas or colons
        attr_value_clean = attr_value.strip()
        if ',' in attr_value_clean or ':' in attr_value_clean:
            attr_value_clean = f'"{attr_value_clean}"'
        formatted.append(f"{section}:{attr_name}:{attr_value_clean}")

    details_string = ','.join(formatted)
    source = ' + '.join(set(sources))
    confidence = 'high' if 'description' in source else 'medium'

    return details_string, source, confidence
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
brand_col = next((c for c in df.columns if c in ['brand', 'merk', 'manufacturer']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gpc_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp1_col = next((c for c in df.columns if c in ['product bullet point 1']), None)
bp2_col = next((c for c in df.columns if c in ['product bullet point 2']), None)
pd_col = next((c for c in df.columns if c in [
    'product_detail', 'product detail',
    'product detail(attribute name:attribute value:section name)'
]), None)

results = {'generated': 0, 'unresolved': 0, 'already_had': 0}
detail_values = []

for idx, row in df.iterrows():
    if pd_col:
        current = str(row[pd_col]).strip() if pd.notna(row[pd_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            detail_values.append(current)
            continue

    # Combine bullet points
    bp_text = ''
    for bp_key in [bp1_col, bp2_col]:
        if bp_key:
            bp = str(row.get(bp_key, '')).strip()
            if bp and bp.lower() != 'nan':
                bp_text += f" {bp}"

    details, source, confidence = generate_product_details(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gpc_col, ''), bp_text.strip(),
        row.get(brand_col, ''))

    detail_values.append(details)
    if details:
        results['generated'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'product_detail': detail_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_product_detail.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with:
- Fill rate before/after
- Average number of specs per product
- Most common spec types found (e.g., Capacity, Weight, Dimensions)
- 10-15 examples showing generated details for spot-checking
- Products that couldn't get any details (insufficient data)

## Category-specific spec templates

Different product categories benefit from different specs. When processing a feed, identify the dominant product category and prioritize these specs:

**Electronics**: Memory, Storage, Battery capacity, Screen size, Resolution, Connectivity (Bluetooth, WiFi, USB), Processor, Weight
**Apparel**: Sustainability info (recycled, organic), Care instructions, Fit type
**Appliances**: Power (Wattage), Energy rating, Capacity, Noise level, Dimensions
**Vehicles/E-bikes**: Range, Max speed, Battery capacity, Weight, Motor power, Wheel size
**Food & Beverages**: Ingredients, Allergens, Nutritional info, Origin, Volume
**Furniture**: Dimensions, Max load, Assembly required, Material details
**Cosmetics**: Skin type, Application area, Volume, Ingredients, Scent

## Important guardrails

- Never overwrite existing product_detail values
- Format MUST be `section_name:attribute_name:attribute_value` with colons as separator
- Use sentence case for attribute names ("Focal length" not "focal length")
- Only provide CONFIRMED specs — never guess technical values
- No promotional text, pricing, shipping, company info, or SEO keywords
- Don't duplicate information already in other feed attributes (color, size, material, weight)
- Values containing commas or colons must be enclosed in double quotes
- Max 1,000 characters per attribute value
- If insufficient data is available to generate meaningful specs, leave empty
- Specs extracted from descriptions are only reliable if they follow a clear "label: value" pattern

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `product_detail`
- Multiple details are comma-separated within the single `product_detail` column
- Format follows Google's TSV spec: `section:name:value,section:name:value`
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
