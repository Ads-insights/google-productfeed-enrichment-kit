---
name: productfeed-item-group-id
description: Extract and fill missing item_group_id attributes in Google Shopping product feeds. Groups product variants (same product, different color/size/material) under one ID. Triggers when user uploads a product feed and wants to populate [item_group_id] values. Also triggers when user mentions "item group", "groeps id", "variant grouping", "varianten koppelen", or asks about grouping product variants. Use this skill even if the user just says "fill in the group IDs" or "group the variants" with an uploaded file.
---

# Feed Item Group ID Generator

Generate `item_group_id` values to group product variants (same product, different color/size/material) together.

## Why this matters

The `[item_group_id]` attribute is high-impact in Google Shopping. Without it:
- Color/size variants show as separate unrelated products
- Shoppers can't switch between variants on the Shopping tab
- Google can't consolidate reviews and ratings across variants
- Products miss variant-specific search queries
- Shopping ads show duplicate products instead of variant swatches

## How item_group_id works

All variants of the same product share the same `item_group_id`. Each variant has its own unique `id` but the same `item_group_id`.

Example:
| id | title | color | size | item_group_id |
|---|---|---|---|---|
| SKU-001-BLK-M | T-shirt Zwart M | Zwart | M | TSHIRT-001 |
| SKU-001-BLK-L | T-shirt Zwart L | Zwart | L | TSHIRT-001 |
| SKU-001-RED-M | T-shirt Rood M | Rood | M | TSHIRT-001 |
| SKU-001-RED-L | T-shirt Rood L | Rood | L | TSHIRT-001 |

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

### Step 2: Grouping strategies

Use multiple strategies in order of reliability.

**Strategy 1: Parent SKU column**
If the feed has a `parent_sku`, `parent_id`, `parent name`, or `groeps id` column, use that directly — it's the most reliable.

**Strategy 2: SKU prefix pattern**
Many webshops encode variants in their SKU structure:
- `PROD-001-BLK-M` and `PROD-001-RED-L` → group = `PROD-001`
- `12345-zwart` and `12345-rood` → group = `12345`
Detect the common prefix pattern across products.

**Strategy 3: Title normalization**
Strip known variant attributes (color, size, material) from the title. Products with the same stripped title are variants.

```python
import re
from collections import Counter

# Words to strip from titles to find the base product name
# These are variant-specific words that differ between variants
VARIANT_WORDS_TO_STRIP = set()

# Build dynamically from the feed's color, size, material values
def build_variant_words(df, color_col, size_col, material_col):
    """Build a set of words that indicate variants (colors, sizes, materials in the feed)."""
    words = set()
    for col in [color_col, size_col, material_col]:
        if col and col in df.columns:
            for val in df[col].dropna().unique():
                for word in str(val).lower().split('/'):
                    word = word.strip()
                    if word and len(word) > 1:
                        words.add(word)
    # Add common size words
    words.update({'xxs', 'xs', 's', 'm', 'l', 'xl', 'xxl', 'xxxl',
                  '2xl', '3xl', '4xl', '5xl', 'one size', 'os'})
    return words

def normalize_title_for_grouping(title, variant_words):
    """Remove variant-specific words from title to create a base product identifier."""
    title_lower = str(title).lower().strip()
    # Remove content in parentheses (often contains variant info)
    title_clean = re.sub(r'\([^)]*\)', '', title_lower)
    # Remove variant words
    words = title_clean.split()
    base_words = [w for w in words if w not in variant_words and len(w) > 0]
    # Remove trailing separators
    base = ' '.join(base_words).strip(' -|–—')
    # Remove trailing dimensions/volumes that differ per variant
    base = re.sub(r'\s*\d+(?:[.,]\d+)?\s*(?:ml|l|cm|mm|kg|g)\s*$', '', base)
    return base.strip()

def extract_item_group_id(df, id_col, title_col, color_col, size_col,
                           material_col, parent_col):
    """Generate item_group_id for all products."""

    results = []

    # --- Strategy 1: Use parent column if available ---
    if parent_col and parent_col in df.columns:
        has_parent = df[parent_col].notna() & (df[parent_col].str.strip() != '') & (df[parent_col].str.lower() != 'nan')
        if has_parent.sum() > 0:
            for idx, row in df.iterrows():
                parent = str(row[parent_col]).strip() if pd.notna(row[parent_col]) else ''
                if parent and parent.lower() not in ['', 'nan', 'none']:
                    results.append((parent, 'parent column', 'high'))
                else:
                    results.append(('', 'none', 'unresolved'))
            return results

    # --- Strategy 2: SKU prefix detection ---
    # Analyze SKU patterns to find common prefixes
    skus = df[id_col].dropna().tolist()
    # Check if SKUs have a pattern like PREFIX-VARIANT
    sku_parts = []
    for sku in skus:
        parts = re.split(r'[-_]', str(sku))
        if len(parts) >= 2:
            sku_parts.append(parts)

    # If >50% of SKUs have multi-part structure, try prefix grouping
    if len(sku_parts) > len(skus) * 0.5:
        # Use all parts except the last 1-2 as group ID
        for idx, row in df.iterrows():
            sku = str(row[id_col])
            parts = re.split(r'[-_]', sku)
            if len(parts) >= 3:
                # Take all but last 2 parts (likely color + size)
                group = '-'.join(parts[:-2])
                results.append((group, 'sku prefix', 'medium'))
            elif len(parts) == 2:
                group = parts[0]
                results.append((group, 'sku prefix', 'medium'))
            else:
                results.append(('', 'none', 'unresolved'))

        # Validate: groups should have 2+ members
        group_counts = Counter(r[0] for r in results if r[0])
        validated = []
        for r in results:
            if r[0] and group_counts[r[0]] >= 2:
                validated.append(r)
            elif r[0] and group_counts[r[0]] < 2:
                validated.append(('', 'none', 'unresolved'))  # Solo "group" = not a real group
            else:
                validated.append(r)

        # If reasonable number of groups found, use this strategy
        real_groups = sum(1 for g, c in group_counts.items() if c >= 2)
        if real_groups >= 2:
            return validated

    # --- Strategy 3: Title normalization ---
    variant_words = build_variant_words(df, color_col, size_col, material_col)

    base_titles = []
    for idx, row in df.iterrows():
        base = normalize_title_for_grouping(row.get(title_col, ''), variant_words)
        base_titles.append(base)

    # Count how many products share each base title
    base_counts = Counter(base_titles)

    results = []
    for i, base in enumerate(base_titles):
        if base and base_counts[base] >= 2:
            # Create a clean group ID from the base title
            group_id = re.sub(r'[^a-z0-9]+', '-', base).strip('-')[:50]
            results.append((group_id, 'title normalization', 'medium'))
        else:
            results.append(('', 'none', 'unresolved'))

    return results
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
color_col = next((c for c in df.columns if c in ['color', 'kleur', 'colour']), None)
size_col = next((c for c in df.columns if c in ['size', 'grootte', 'maat']), None)
material_col = next((c for c in df.columns if c in ['material', 'materiaal']), None)
parent_col = next((c for c in df.columns if c in ['parent sku', 'parent_sku', 'parent name',
                                                     'groeps id', 'item_group_id', 'parent id']), None)
existing_group_col = next((c for c in df.columns if c in ['item_group_id', 'groeps id']), None)

# Check if already populated
if existing_group_col:
    has_group = df[existing_group_col].notna() & (df[existing_group_col].str.strip() != '') & (df[existing_group_col].str.lower() != 'nan')
    if has_group.sum() > len(df) * 0.5:
        # More than half already filled — respect existing
        group_values = df[existing_group_col].fillna('').replace('nan', '').tolist()
        # Only fill empty ones
        empty_mask = ~has_group
        if empty_mask.sum() > 0:
            extracted = extract_item_group_id(df[empty_mask], id_col, title_col,
                                              color_col, size_col, material_col, parent_col)
            j = 0
            for i in range(len(group_values)):
                if not group_values[i] or group_values[i].lower() in ['nan', 'none']:
                    if j < len(extracted):
                        group_values[i] = extracted[j][0]
                    j += 1
    else:
        extracted = extract_item_group_id(df, id_col, title_col,
                                          color_col, size_col, material_col, parent_col)
        group_values = [e[0] for e in extracted]
else:
    extracted = extract_item_group_id(df, id_col, title_col,
                                      color_col, size_col, material_col, parent_col)
    group_values = [e[0] for e in extracted]

supplemental = pd.DataFrame({
    'id': df[id_col],
    'item_group_id': group_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_item_group_id.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary showing:
- How many groups were created
- Average group size (variants per group)
- Sample of groups with their variants for spot-checking
- Products not assigned to any group (singletons)

## Important guardrails

- Never overwrite existing item_group_id values
- A group must have at least 2 products — a "group" of 1 is not a group
- Products that differ only in quantity (e.g., 100ml vs 500ml of the same cream) MAY be variants but could also be separate products — flag for review
- The item_group_id value itself doesn't appear in ads — it's purely for internal grouping
- Max 100 variants per group (Google limit)
- When using title normalization, show the user examples so they can verify the grouping makes sense
- Parent SKU is always the most reliable source — prefer it over any inference

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `item_group_id`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones — products without a group get an empty cell)
