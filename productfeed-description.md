---
name: productfeed-description
description: Generate or improve product descriptions for Google Shopping product feeds. Only intervenes when descriptions are empty or critically short (<150 characters). Conservative approach - prefers keeping an existing description over risking a worse one. Triggers when user uploads a product feed and wants to improve [description] values. Also triggers when user mentions "description", "beschrijving", "product description", or asks about description optimization. Use this skill even if the user just says "fix the descriptions" or "improve descriptions" with an uploaded file.
---

# Feed Description Optimizer

Generate new descriptions for empty products or improve critically short descriptions (<150 characters), following Google's best practices.

## Why this matters

The `[description]` attribute is required in Google Shopping, but has lower direct impact than title:
- Google reads every word to understand the product and match to search queries
- Shoppers rarely read the full description in Shopping ads (only ~150-200 chars show in expanded views)
- Products without a description may still serve but with limited performance
- A good description helps Google match long-tail search queries
- Max 5,000 characters — the first 150-500 characters matter most

## Conservative approach

**This skill is deliberately conservative. The priority order is:**

1. Existing description ≥500 chars → **don't touch** (even if imperfect)
2. Existing description 150-500 chars → **don't touch** (acceptable for most products)
3. Existing description <150 chars → **expand** by appending attribute info (keep original text)
4. Empty description → **generate** from available attributes

**The reasoning:** a mediocre human-written description that matches the landing page is better than a polished AI description that doesn't. Google compares feed descriptions to landing pages — mismatches can cause disapprovals.

## Google's specifications

**Format**: Free text, 1-5,000 characters
**Language**: Must match feed language
**Required**: Yes, for all products (but products without description may still serve with limited performance)

**Rules — what TO do:**
- Put the most important info in the first 150-500 characters
- Include relevant attributes: brand, product type, color, size, material, key features
- Write for Google's algorithm (keyword matching), not the shopper (they'll read the landing page)
- Be accurate — description must match what's on the landing page

**Rules — what NOT to do:**
- No promotional text (pricing, shipping, "gratis", "korting", "beste prijs")
- No ALL CAPS for emphasis
- No HTML tags
- No links or references to other websites
- No comparisons with other products
- No excessive keywords/keyword stuffing
- No information about other products in your catalog

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

Report: total products, description length distribution (empty, <150, 150-500, 500+), average length.

### Step 2: Description assessment

```python
import re

INTERVENTION_THRESHOLD = 150  # Only intervene below this character count
LEAVE_ALONE_THRESHOLD = 150   # Above this, don't touch

def assess_description(description):
    """Assess a description and determine the action needed.
    Returns (action, length, issues)."""
    desc_str = str(description).strip() if pd.notna(description) else ''

    if not desc_str or desc_str.lower() in ['nan', 'none', '']:
        return 'generate', 0, ['empty']

    length = len(desc_str)

    if length >= LEAVE_ALONE_THRESHOLD:
        return 'leave', length, []

    # Short description — check quality
    issues = []
    if length < 50:
        issues.append('very_short')
    elif length < INTERVENTION_THRESHOLD:
        issues.append('short')

    # Check for red flags
    desc_lower = desc_str.lower()
    if desc_str.isupper():
        issues.append('all_caps')
    if re.search(r'gratis|free shipping|korting|sale|aanbieding|beste prijs', desc_lower):
        issues.append('promotional')
    if re.search(r'<[^>]+>', desc_str):
        issues.append('html_tags')
    if re.search(r'https?://', desc_str):
        issues.append('contains_links')

    return 'expand', length, issues
```

### Step 3: Description generation and expansion

```python
def gather_product_info(row, col_map):
    """Collect all available product data."""
    info = {}
    for key, possible_cols in col_map.items():
        for col in possible_cols:
            if col in row.index:
                val = str(row[col]).strip() if pd.notna(row[col]) else ''
                if val and val.lower() not in ['', 'nan', 'none']:
                    info[key] = val
                    break
    return info

COL_MAP = {
    'title': ['title', 'titel', 'product name'],
    'brand': ['brand', 'merk', 'manufacturer'],
    'color': ['color', 'kleur', 'colour'],
    'material': ['material', 'materiaal'],
    'size': ['size', 'grootte', 'maat'],
    'pattern': ['pattern', 'patroon'],
    'product_type': ['product_type', 'producttype'],
    'google_category': ['google_product_category', 'google productcategorie'],
    'condition': ['condition', 'staat'],
    'weight': ['product_weight', 'verzendgewicht', 'shipping weight'],
    'bullet1': ['product bullet point 1'],
    'bullet2': ['product bullet point 2'],
    'bullet3': ['product bullet point 3'],
    'labels': ['labels', 'aangepast label 1'],
    'gender': ['gender', 'geslacht'],
    'age_group': ['age_group', 'leeftijdsgroep'],
}


def detect_feed_language(df, title_col):
    """Detect feed language by checking titles for Dutch vs English words."""
    nl_words = {'de', 'het', 'een', 'van', 'voor', 'met', 'en', 'handgemaakt', 'maat', 'kleur'}
    en_words = {'the', 'a', 'an', 'for', 'with', 'and', 'handmade', 'size', 'color', 'colour'}
    nl_score = 0
    en_score = 0
    sample = df[title_col].dropna().head(50)
    for title in sample:
        words = set(str(title).lower().split())
        nl_score += len(words & nl_words)
        en_score += len(words & en_words)
    return 'en' if en_score > nl_score else 'nl'

DESC_TEMPLATES = {
    'nl': {
        'brand': 'Van {value}.',
        'features_label': 'Kenmerken',
        'material': 'materiaal: {value}',
        'color': 'kleur: {value}',
        'size': 'maat/formaat: {value}',
        'pattern': 'patroon: {value}',
        'weight': 'gewicht: {value}',
        'condition': 'conditie: {value}',
        'category': 'Categorie: {value}.',
        'expand_brand': 'Merk: {value}.',
        'expand_material': 'Materiaal: {value}.',
        'expand_color': 'Kleur: {value}.',
        'expand_size': 'Maat: {value}.',
        'expand_weight': 'Gewicht: {value}.',
        'shop_suffixes': r'(?:Kopen|Webshop|Online|Shop|Bestellen)',
    },
    'en': {
        'brand': 'By {value}.',
        'features_label': 'Features',
        'material': 'material: {value}',
        'color': 'colour: {value}',
        'size': 'size: {value}',
        'pattern': 'pattern: {value}',
        'weight': 'weight: {value}',
        'condition': 'condition: {value}',
        'category': 'Category: {value}.',
        'expand_brand': 'Brand: {value}.',
        'expand_material': 'Material: {value}.',
        'expand_color': 'Colour: {value}.',
        'expand_size': 'Size: {value}.',
        'expand_weight': 'Weight: {value}.',
        'shop_suffixes': r'(?:Buy|Shop|Order|Store|Buy now|Shop now)',
    },
}


def generate_description(info, lang='nl'):
    """Generate a new description from available product attributes.
    Returns description text or empty string."""

    t = DESC_TEMPLATES[lang]
    title = info.get('title', '')
    brand = info.get('brand', '')
    color = info.get('color', '')
    material = info.get('material', '')
    size = info.get('size', '')
    product_type = info.get('product_type', '')
    condition = info.get('condition', '')
    weight = info.get('weight', '')
    pattern = info.get('pattern', '')

    parts = []

    # Part 1: What it is (from title, cleaned up)
    if title:
        clean_title = re.sub(r'\s*\|.*$', '', title)
        clean_title = re.sub(r'\s*[-–—]\s*' + t['shop_suffixes'] + r'.*$', '', clean_title, flags=re.I)
        parts.append(clean_title.strip())

    # Part 2: Brand mention
    if brand and brand.lower() not in title.lower():
        parts.append(t['brand'].format(value=brand))

    # Part 3: Key features
    features = []
    if material:
        features.append(t['material'].format(value=material))
    if color:
        features.append(t['color'].format(value=color))
    if size:
        features.append(t['size'].format(value=size))
    if pattern:
        features.append(t['pattern'].format(value=pattern))
    if weight:
        features.append(t['weight'].format(value=weight))
    if condition and condition.lower() != 'new':
        features.append(t['condition'].format(value=condition))

    if features:
        parts.append(t['features_label'] + ": " + ", ".join(features) + ".")

    # Part 4: Product type context
    if product_type:
        parts.append(t['category'].format(value=product_type))

    # Part 5: Bullet points (if available)
    for bp_key in ['bullet1', 'bullet2', 'bullet3']:
        bp = info.get(bp_key, '')
        if bp:
            parts.append(bp)

    description = ' '.join(parts)

    # Clean up
    description = re.sub(r'\s+', ' ', description).strip()

    # Only return if we have enough meaningful content
    if len(description) < 30:
        return ''

    return description[:5000]  # Max 5000 chars


def expand_description(existing_desc, info, lang='nl'):
    """Expand a short description by appending relevant attribute info.
    Preserves the original text and adds to it.
    Returns expanded description or original if nothing to add."""

    t = DESC_TEMPLATES[lang]
    existing = str(existing_desc).strip()

    # Fix obvious issues first
    expanded = existing

    # Fix ALL CAPS
    if expanded.isupper():
        expanded = expanded.capitalize()

    # Remove HTML tags
    expanded = re.sub(r'<[^>]+>', '', expanded)

    # Remove links
    expanded = re.sub(r'https?://\S+', '', expanded)

    # Now try to add useful info that's not already in the description
    additions = []
    desc_lower = expanded.lower()

    brand = info.get('brand', '')
    if brand and brand.lower() not in desc_lower:
        additions.append(t['expand_brand'].format(value=brand))

    material = info.get('material', '')
    if material and material.lower() not in desc_lower:
        additions.append(t['expand_material'].format(value=material))

    color = info.get('color', '')
    if color and color.lower() not in desc_lower:
        additions.append(t['expand_color'].format(value=color))

    size = info.get('size', '')
    if size and size.lower() not in desc_lower:
        additions.append(t['expand_size'].format(value=size))

    weight = info.get('weight', '')
    if weight and weight.lower() not in desc_lower:
        additions.append(t['expand_weight'].format(value=weight))

    if additions:
        expanded = expanded.rstrip('.') + '. ' + ' '.join(additions)

    expanded = re.sub(r'\s+', ' ', expanded).strip()

    # Only return expanded version if we actually added meaningful content
    if len(expanded) > len(existing) + 20:
        return expanded[:5000]
    return existing  # Return original if we couldn't add enough
```

### Step 4: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)

results = {'generated': 0, 'expanded': 0, 'left_alone': 0, 'insufficient': 0}
desc_values = []

for idx, row in df.iterrows():
    current_desc = row.get(desc_col, '') if desc_col else ''
    action, length, issues = assess_description(current_desc)

    if action == 'leave':
        # Description is fine — empty cell in supplemental = no override
        desc_values.append('')
        results['left_alone'] += 1

    elif action == 'generate':
        # Empty description — generate from attributes
        info = gather_product_info(row, COL_MAP)
        new_desc = generate_description(info)
        if new_desc:
            desc_values.append(new_desc)
            results['generated'] += 1
        else:
            desc_values.append('')
            results['insufficient'] += 1

    elif action == 'expand':
        # Short description — try to expand
        info = gather_product_info(row, COL_MAP)
        expanded = expand_description(current_desc, info)
        if expanded != str(current_desc).strip():
            desc_values.append(expanded)
            results['expanded'] += 1
        else:
            desc_values.append('')  # Couldn't improve — leave as is
            results['left_alone'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'description': desc_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_description.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with:
- Total products analyzed
- Left alone (≥150 chars, acceptable): X
- Expanded (was <150 chars, improved): X
- Generated (was empty): X
- Insufficient data (empty, couldn't generate): X
- Description length distribution before vs after
- Show 10-15 examples of generated/expanded descriptions for spot-checking
- Show a few "left alone" examples to confirm they were correctly identified as acceptable

## Important guardrails

- **CONSERVATIVE FIRST**: when in doubt, don't touch the description
- Only intervene on descriptions below 150 characters or empty — never "optimize" a 500+ char description
- When expanding, PRESERVE the original text and append attribute info — never rewrite
- When generating, only use confirmed data from the feed — never invent product features
- Generated descriptions will be simpler and more formulaic than hand-written ones — that's acceptable
- Language must match the feed language
- No promotional text, pricing, shipping info, links, HTML, or ALL CAPS
- No comparisons with other products or references to other catalog items
- Max 5,000 characters (practically, aim for 150-500 for generated descriptions)
- Empty cells in the supplemental feed = no override (description stays as-is)
- The first 150 characters are the most important — front-load key product info
- A factual description built from attributes ("Nike Air Max 90 sneaker. Materiaal: leer. Kleur: zwart. Maat: 43.") is better than no description at all, even if it reads dry

## Quality philosophy

The description is primarily for Google's algorithm, not for shoppers (they'll read the landing page). A structured, attribute-rich description that reads a bit dry but is accurate and keyword-rich is more valuable than a creative but vague marketing text. Think of it as feeding Google information, not writing ad copy.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `description`
- Empty cells = description was already acceptable (no override needed)
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (empty cells for unchanged descriptions)
