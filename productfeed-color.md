---
name: productfeed-color
description: Extract and fill missing color attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [color] values. Also triggers when user mentions "color attribute", "kleur invullen", "feed color", "missing colors in feed", or asks to enrich/complete/fix color data in a product feed. Use this skill even if the user just says "fill in the colors" or "fix my feed colors" with an uploaded file.
---

# Feed Color Extractor

Extract color values from product titles, descriptions, and other available fields to fill empty `color` attributes in a Google Shopping product feed.

## Why this matters

The `[color]` attribute is marked as high-impact in Google Shopping. Missing color data leads to:
- Disapproved products in apparel/fashion categories
- Missed search queries containing color terms (e.g., "zwarte sneakers")
- Lower ad relevance and reduced impression share
- Poor variant grouping when `item_group_id` is used

## Workflow

### Step 1: Load and inspect the feed

Read the uploaded file (CSV, TSV, or Excel). Identify which columns are present. The color column may be named any of: `color`, `kleur`, `Color`, `colour`, `g:color`, `color [color]`. Also locate these source columns (used for extraction): `title`, `description`, `product_type`, `google_product_category`, `image_link`.

```python
import pandas as pd

# Detect file type and load
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

# Normalize column names to lowercase for matching
df.columns = [c.strip().lower() for c in df.columns]
```

Report to the user:
- Total products in feed
- How many have an empty/missing color value
- Which source columns are available for extraction

### Step 2: Build the color extraction logic

The extraction follows a priority cascade — stop at the first match:

**Priority 1: Direct color word match in title**
Scan the title for known color terms. This is the most reliable source because merchants typically include the primary color in the title.

**Priority 2: Color word match in description**
Same logic, applied to the description field. Less reliable due to longer text and potential false positives (e.g., "zwart gat" in a description about space).

**Priority 3: Inference from product type or category**
Some categories have implied defaults, but this is risky. Only apply when confidence is very high (e.g., "gold jewelry" → goud).

**Priority 4: Flag as unresolved**
If no color can be confidently extracted, mark the product for manual review rather than guessing.

### Step 3: Color vocabulary

Use this comprehensive color list for matching. Support both Dutch and English since feeds may use either language. Match case-insensitively.

```python
COLOR_MAP = {
    # Primary colors - Dutch → normalized Dutch output
    'zwart': 'Zwart', 'black': 'Zwart',
    'wit': 'Wit', 'white': 'Wit', 'witte': 'Wit',
    'rood': 'Rood', 'red': 'Rood', 'rode': 'Rood',
    'blauw': 'Blauw', 'blue': 'Blauw', 'blauwe': 'Blauw',
    'groen': 'Groen', 'green': 'Groen', 'groene': 'Groen',
    'geel': 'Geel', 'yellow': 'Geel', 'gele': 'Geel',
    'oranje': 'Oranje', 'orange': 'Oranje',
    'paars': 'Paars', 'purple': 'Paars', 'violet': 'Paars',
    'roze': 'Roze', 'pink': 'Roze',
    'bruin': 'Bruin', 'brown': 'Bruin', 'bruine': 'Bruin',
    'grijs': 'Grijs', 'grey': 'Grijs', 'gray': 'Grijs', 'grijze': 'Grijs',
    'beige': 'Beige', 'crème': 'Beige', 'creme': 'Beige', 'cream': 'Beige',
    'goud': 'Goud', 'gold': 'Goud', 'golden': 'Goud', 'gouden': 'Goud',
    'zilver': 'Zilver', 'silver': 'Zilver', 'zilveren': 'Zilver',
    'brons': 'Brons', 'bronze': 'Brons',
    'turquoise': 'Turquoise', 'turkoois': 'Turquoise',
    'bordeaux': 'Bordeaux', 'burgundy': 'Bordeaux', 'bordeauxrood': 'Bordeaux',
    'koraal': 'Koraal', 'coral': 'Koraal',
    'magenta': 'Magenta', 'fuchsia': 'Magenta',
    'navy': 'Navy', 'marineblauw': 'Navy', 'donkerblauw': 'Navy',
    'mint': 'Mint', 'mintgroen': 'Mint',
    'ivoor': 'Ivoor', 'ivory': 'Ivoor',
    'khaki': 'Khaki', 'kaki': 'Khaki',
    'taupe': 'Taupe',
    'lila': 'Lila', 'lilac': 'Lila',
    'petrol': 'Petrol', 'teal': 'Petrol',
    'olijf': 'Olijfgroen', 'olive': 'Olijfgroen', 'olijfgroen': 'Olijfgroen',
    'cognac': 'Cognac', 'camel': 'Camel',
    'antraciet': 'Antraciet', 'anthracite': 'Antraciet',
    'ecru': 'Ecru', 'offwhite': 'Ecru', 'off-white': 'Ecru',
    'indigo': 'Indigo',
    'terra': 'Terracotta', 'terracotta': 'Terracotta',
    'mosterd': 'Mosterdgeel', 'mustard': 'Mosterdgeel',
    'pastelroze': 'Pastelroze', 'pastel pink': 'Pastelroze',
    'pastelblauw': 'Pastelblauw', 'pastel blue': 'Pastelblauw',

    # Composite modifiers (match these as prefixes)
    'donker': None,  # handled separately as prefix
    'licht': None,   # handled separately as prefix
    'dark': None,
    'light': None,
}

# Composite color patterns: "donkerblauw", "lichtgroen", "dark blue", etc.
COMPOSITE_PREFIXES_NL = ['donker', 'licht', 'helder', 'warm', 'koel']
COMPOSITE_PREFIXES_EN = ['dark', 'light', 'bright', 'warm', 'cool']

# Multi-word patterns like "army green", "sky blue", "hot pink"
MULTI_WORD_COLORS = {
    'army green': 'Legergroen',
    'sky blue': 'Hemelsblauw',
    'hot pink': 'Felroze',
    'baby blue': 'Babyblauw',
    'baby pink': 'Babyroze',
    'royal blue': 'Koningsblauw',
    'forest green': 'Bosgroen',
    'ice blue': 'IJsblauw',
    'rose gold': 'Roségoud',
    'space grey': 'Space Grey',
    'space gray': 'Space Grey',
    'midnight blue': 'Nachtblauw',
    'jet black': 'Gitzwart',
    'snow white': 'Sneeuwwit',
    'steel blue': 'Staalblauw',
}
```

### Step 4: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
import re
from collections import Counter

def _scan_for_color(text):
    """Scan a text string for color matches. Returns (color, match_type) or (None, None)."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None, None
    # Multi-word first
    for pattern, color in MULTI_WORD_COLORS.items():
        if pattern in text_lower:
            return color, 'multi-word'
    # Single-word
    words = re.findall(r'[a-zà-ÿ-]+', text_lower)
    for word in words:
        if word in COLOR_MAP and COLOR_MAP[word] is not None:
            return COLOR_MAP[word], 'single'
    # Composite (donkerblauw, lichtgroen)
    for word in words:
        for prefix in COMPOSITE_PREFIXES_NL + COMPOSITE_PREFIXES_EN:
            if word.startswith(prefix) and len(word) > len(prefix):
                base = word[len(prefix):]
                if base in COLOR_MAP and COLOR_MAP[base] is not None:
                    return word.capitalize(), 'composite'
    return None, None

def extract_color(title, description='', product_type='', labels='',
                  google_category='', image_link='', bullet_points=''):
    """Extract color using 4-layer cascade. Returns (color, source, confidence)."""

    # --- LAYER 1: Title (high confidence) ---
    color, _ = _scan_for_color(title)
    if color:
        return color, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    color, _ = _scan_for_color(description)
    if color:
        return color, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    # Check product_type, google_product_category, labels, bullet points
    for source_name, source_text in [
        ('product_type', product_type),
        ('google_category', google_category),
        ('labels', labels),
        ('bullet_points', bullet_points),
    ]:
        color, _ = _scan_for_color(source_text)
        if color:
            return color, source_name, 'medium'

    # Check image_link filename (low confidence fallback)
    if pd.notna(image_link) and str(image_link).strip():
        img_filename = str(image_link).split('/')[-1].split('?')[0].lower()
        img_words = re.findall(r'[a-zà-ÿ]+', img_filename)
        for word in img_words:
            if word in COLOR_MAP and COLOR_MAP[word] is not None:
                return COLOR_MAP[word], 'image_link filename', 'low'

    # --- LAYER 4: No color found ---
    return None, 'none', 'unresolved'
```

### Step 5: Apply to the feed and generate output

```python
# Identify rows with missing color
color_col = next((c for c in df.columns if c in ['color', 'kleur', 'colour', 'g:color']), None)

if color_col is None:
    # Create the column if it doesn't exist at all
    color_col = 'color'
    df[color_col] = ''

# Track changes for reporting
results = {'filled': 0, 'unresolved': 0, 'already_had': 0, 'details': []}

for idx, row in df.iterrows():
    current_color = str(row[color_col]).strip() if pd.notna(row[color_col]) else ''

    if current_color and current_color.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue

    # Try extraction
    title = row.get('title', '')
    desc = row.get('description', '')
    ptype = row.get('product_type', '')

    color, source, confidence = extract_color(title, desc, ptype)

    if color:
        df.at[idx, color_col] = color
        results['filled'] += 1
        results['details'].append({
            'id': row.get('id', idx),
            'title': str(title)[:60],
            'color_found': color,
            'source': source,
            'confidence': confidence
        })
    else:
        results['unresolved'] += 1
        results['details'].append({
            'id': row.get('id', idx),
            'title': str(title)[:60],
            'color_found': None,
            'source': 'none',
            'confidence': 'unresolved'
        })
```

### Step 6: Save as supplemental feed and report

The output is always a **supplemental feed** — not a copy of the full feed. This means:

- **Column 1**: `id` — the product ID (always the Google-recognized attribute name `id`)
- **Column 2**: `color` — the Google-recognized English attribute name, with the extracted or existing color value

Include **ALL products** (not just the ones that were enriched), so the user can see which products still have empty values and need manual attention.

**Single-attribute output format (when only this skill is used):**

| id | color |
|---|---|
| product_123 | zwart |
| product_456 | rood |
| product_789 | |

**Multi-attribute output format (when multiple feed-attribute skills are combined in one request):**

When the user asks to run multiple attribute skills at once (e.g., "fill in color, material, and gender"), combine the results into a single supplemental feed with one `id` column and one column per attribute, using the Google-recognized English attribute names:

| id | color | material | gender |
|---|---|---|---|
| product_123 | zwart | katoen | male |
| product_456 | rood | leer | female |
| product_789 | | | male |

**Important rules for the supplemental feed output:**
- Column names are ALWAYS the English Google Merchant Center attribute names: `id`, `color`, `material`, `gender`, `size`, `age_group`, `pattern`, `product_type`, `google_product_category`, `condition`, `brand`, etc.
- Never use Dutch column names in the output (no `kleur`, `materiaal`, `geslacht`, etc.)
- Include all products, even if the attribute value is empty — leave the cell blank
- The output format is always **Excel (.xlsx)**. Tip the user that they can upload this to Google Drive where it automatically becomes a Google Sheet, ready to use as supplemental feed.

```python
# Build supplemental feed with only id + color
supplemental = pd.DataFrame({
    'id': df['id'],  # Always use the product ID column from the original feed
    'color': df[color_col]  # The enriched color values
})

# Save as temporary Excel, then upload to Google Sheets via Google Drive
output_path = "/home/claude/supplemental_feed_color.xlsx"
supplemental.to_excel(output_path, index=False)

# Then use Google Drive tools to create a Google Sheet from this data
```

**To create the Google Sheet:** After building the supplemental DataFrame, use the Google Drive / Google Sheets tools to create a new spreadsheet directly in the user's Drive. Name it `Supplemental Feed - Color` (or matching the attribute name). If Google Drive tools are unavailable, fall back to Excel output in `/mnt/user-data/outputs/`.

Present the user with a summary:

```
=== Color Extraction Results ===
Total products:        {total}
Already had color:     {already_had}
Successfully filled:   {filled}  (high confidence: X, medium: Y)
Still empty:           {unresolved}
Fill rate before:      X%
Fill rate after:       Y%
```

Also show a sample of 10-15 filled values so the user can spot-check, and list the products that remain empty.

### Step 7: Language detection

Before outputting, check if the existing color values in the feed are in Dutch or English. Match the output language to whatever the feed already uses. If no existing colors are present, default to Dutch (since this is for Dutch webshops).

To detect: look at the non-empty color values already in the feed. If >50% match English color words, output in English. Otherwise output in Dutch.

### Multi-color detection

Before running the single-color extraction, check for slash-separated or "en"/"and"-separated color combinations in the title. These are common in variant products.

```python
def extract_multi_color(title):
    """Detect color combinations like 'Zwart/Wit', 'Rood en Blauw', 'Black/Gold'."""
    title_lower = str(title).lower()

    # Pattern: "Color1/Color2" or "Color1 / Color2"
    slash_pattern = re.findall(r'([a-zà-ÿ-]+)\s*/\s*([a-zà-ÿ-]+)', title_lower)
    for c1, c2 in slash_pattern:
        if c1 in COLOR_MAP and c2 in COLOR_MAP:
            mapped1 = COLOR_MAP[c1]
            mapped2 = COLOR_MAP[c2]
            if mapped1 and mapped2:
                return f"{mapped1}/{mapped2}", 'title (multi-color)', 'high'

    # Pattern: "Color1 en Color2" or "Color1 and Color2"
    and_pattern = re.findall(r'([a-zà-ÿ-]+)\s+(?:en|and)\s+([a-zà-ÿ-]+)', title_lower)
    for c1, c2 in and_pattern:
        if c1 in COLOR_MAP and c2 in COLOR_MAP:
            mapped1 = COLOR_MAP[c1]
            mapped2 = COLOR_MAP[c2]
            if mapped1 and mapped2:
                return f"{mapped1}/{mapped2}", 'title (multi-color)', 'high'

    return None, None, None
```

Insert this as **Priority 0** in the extraction function — run it before the single-color passes. If it returns a result, use it and skip the rest.

## Edge cases to handle

- **Multiple colors in title**: e.g., "Sneaker Zwart/Wit" → use "Zwart/Wit" (preserve the combination)
- **Color codes instead of names**: e.g., "#000000" or "RAL 9010" → skip, flag for manual review
- **Brand names that look like colors**: e.g., "Blue Lagoon Parfum" where Blue Lagoon is the product name → use word position heuristics (colors near the end of a title are more likely actual product colors)
- **Abbreviations**: "blk" → "Zwart", "wht" → "Wit", "gry" → "Grijs"
- **Mixed feeds**: Some products have colors, others don't. Only process empty rows.
- **Color in image_link filename**: Sometimes the color is encoded in the image URL (e.g., `/product-black-xl.jpg`). This is a low-confidence fallback — only use when title and description yield nothing, and flag confidence as "low".

## Important guardrails

- Never overwrite existing color values, even if they look "wrong"
- Always preserve the original file as-is; output to a new file
- Flag uncertainty honestly — a "medium" or "unresolved" tag is more valuable than a wrong color
- If < 30% of empty colors could be filled, warn the user that the feed may need manual enrichment or that titles/descriptions lack color information
- Products in categories where color is irrelevant (e.g., software, digital goods) should be skipped entirely

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 kolommen → `id` + `color`
- **Multiple attributes** (when combined with other feed-attribute skills): 1 `id` kolom + 1 kolom per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet for use as supplemental feed
- Include ALL products (also empty ones, so the user can see what still needs manual work)
