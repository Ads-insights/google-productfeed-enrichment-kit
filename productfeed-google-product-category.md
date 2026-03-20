---
name: productfeed-google-product-category
description: Classify and fill missing google_product_category attributes in Google Shopping product feeds. Maps products to Google's official taxonomy of 6000+ categories. Triggers when user uploads a product feed and wants to populate [google_product_category] values. Also triggers when user mentions "google product category", "google categorie", "GPC", "taxonomy", "product categorization", or asks about product classification. Use this skill even if the user just says "categorize the products" or "fill in the google categories" with an uploaded file.
---

# Feed Google Product Category Classifier

Classify products into Google's official product taxonomy and fill the `google_product_category` attribute.

## Why this matters

The `[google_product_category]` attribute is optional but high-impact in Google Shopping:
- Google auto-assigns categories, but these are often too broad or incorrect
- Correct categorization improves ad relevance, search matching, and Shopping tab filtering
- Some categories (Apparel, Media, Software) require specific additional attributes — wrong category = wrong requirements
- Category directly affects bidding segmentation in Google Ads campaigns
- A wrong GPC is worse than no GPC — it causes Merchant Center disapprovals

## Google's specifications

**Format**: Either the numeric ID or the full category path (not both)
- Numeric ID: `3580` (preferred — stable, language-neutral)
- Full path: `Arts & Entertainment > Hobbies & Creative Arts > Arts & Crafts`

**Key rules:**
- Use only predefined categories from Google's taxonomy (~6000 categories)
- Use the MOST SPECIFIC category possible (deepest level in the tree)
- Categorize based on the product's MAIN FUNCTION, not how it's marketed
- A "gaming chair" is Furniture > Chairs, not Electronics
- One category per product (no multi-category)
- If no category fits, don't assign one — use product_type instead

## Step 1: Load the Google taxonomy

**Before classifying any products, fetch and parse the official Google taxonomy.** This is non-negotiable — without it, classification is guesswork.

```python
import re
import pandas as pd

# Fetch the official taxonomy
taxonomy_url = "https://www.google.com/basepages/producttype/taxonomy-with-ids.en-US.txt"

# Use web_fetch tool to download the taxonomy file, then parse it:
taxonomy = {}        # {id_str: full_path}  e.g. {'212': 'Apparel & Accessories > Clothing > Shirts & Tops'}
taxonomy_reverse = {}  # {full_path_lower: id_str} for reverse lookup
top_level_map = {}   # {id_str: top_level_category} e.g. {'212': 'Apparel & Accessories'}

for line in taxonomy_text.strip().split('\n'):
    if line.startswith('#') or not line.strip():
        continue
    parts = line.split(' - ', 1)
    if len(parts) == 2:
        cat_id = parts[0].strip()
        cat_path = parts[1].strip()
        taxonomy[cat_id] = cat_path
        taxonomy_reverse[cat_path.lower()] = cat_id
        top_level_map[cat_id] = cat_path.split(' > ')[0]

print(f"Loaded {len(taxonomy)} Google taxonomy categories")
```

This gives you three powerful tools:
1. **`taxonomy[id]`** — validate any GPC code exists and see its full path
2. **`taxonomy_reverse[path]`** — look up the ID for a category path
3. **`top_level_map[id]`** — get the top-level category for cross-checking

## Step 2: Load and inspect the feed

```python
file_path = "/mnt/user-data/uploads/<filename>"
# [standard file loading code — see orchestrator for format detection]

df.columns = [c.strip().lower() for c in df.columns]
```

## Step 3: Classification strategy

Uses a 4-layer cascade. **Every assigned GPC is validated against the taxonomy before output.**

### Layer 1: Learn from existing categories in the feed

If some products already have a google_product_category, build a mapping from product_type/title keywords to categories. This is the most powerful strategy — it uses the merchant's own categorization as ground truth.

```python
def build_learned_map(df, gpc_col, ptype_col, title_col):
    """Learn category assignments from products that already have a GPC."""
    learned = {}
    if not gpc_col or gpc_col not in df.columns:
        return learned

    has_gpc = df[gpc_col].notna() & (df[gpc_col].str.strip() != '') & (df[gpc_col].str.lower() != 'nan')

    # Map product_type → most common GPC
    if ptype_col and ptype_col in df.columns:
        for ptype in df.loc[has_gpc, ptype_col].dropna().unique():
            ptype_clean = str(ptype).strip().lower()
            if ptype_clean and ptype_clean not in ['', 'nan']:
                mask = has_gpc & (df[ptype_col].str.lower().str.strip() == ptype_clean)
                if mask.sum() > 0:
                    gpc_val = df.loc[mask, gpc_col].mode().iloc[0]
                    # VALIDATE against taxonomy
                    if str(gpc_val).strip() in taxonomy:
                        learned[ptype_clean] = str(gpc_val).strip()

    return learned
```

### Layer 2: Map from product_type using taxonomy path matching

Match the merchant's product_type hierarchy against Google's taxonomy paths. Find the most specific matching category.

```python
def match_product_type_to_taxonomy(product_type):
    """Find the best taxonomy match for a merchant's product_type.
    Returns (gpc_id, confidence) or (None, None)."""
    if not product_type or str(product_type).lower() in ['nan', '']:
        return None, None

    pt_lower = str(product_type).lower().strip()
    pt_parts = [p.strip() for p in pt_lower.split('>')]

    # Try matching the last (most specific) part of product_type against taxonomy paths
    best_match_id = None
    best_match_depth = 0

    for cat_id, cat_path in taxonomy.items():
        cat_lower = cat_path.lower()
        cat_parts = [p.strip() for p in cat_lower.split('>')]

        # Check how many levels match from the end (most specific first)
        matches = 0
        for pt_part in reversed(pt_parts):
            for cat_part in reversed(cat_parts):
                if pt_part in cat_part or cat_part in pt_part:
                    matches += 1
                    break

        # Prefer deeper matches (more specific categories)
        cat_depth = len(cat_parts)
        if matches > best_match_depth or (matches == best_match_depth and cat_depth > best_match_depth):
            if matches >= 1:
                best_match_id = cat_id
                best_match_depth = matches

    if best_match_id:
        return best_match_id, 'medium' if best_match_depth >= 2 else 'low'
    return None, None
```

### Layer 3: Keyword-based classification from title + description

Use Claude's understanding of the product to classify it. **Do not use a hardcoded keyword map.** Instead, Claude should:
1. Read the product title and description
2. Determine what the product IS (its main function)
3. Search the loaded taxonomy for the most specific matching category
4. Return the numeric ID

This is where Claude's language understanding matters — it can classify a "Kimono-bademantel Scenario 350 G/m² Für Erwachsene" as a bathrobe (GPC 2302) regardless of language, because it understands German.

### Layer 4: No match found

If no category fits well, leave empty. A wrong GPC is worse than no GPC.

## Step 4: Post-classification validation

**Every GPC assignment must pass validation before output.** This is the critical step that prevents the "BH on a children's T-shirt" problem.

```python
def validate_gpc(gpc_id, title, product_type, age_group='', gender=''):
    """Validate a NEWLY ASSIGNED GPC before accepting it.
    Only used for GPCs that this skill assigns — never for existing merchant GPCs.
    Returns (validated_gpc, reason). Empty string if invalid."""
    if not gpc_id:
        return '', 'no_gpc'

    gpc_str = str(gpc_id).strip()

    # CHECK 1: Does this GPC exist in the taxonomy?
    if gpc_str not in taxonomy:
        return '', f'gpc_not_in_taxonomy: {gpc_str} is not a valid Google category ID'

    # CHECK 2: Get the full path and top-level category
    gpc_path = taxonomy[gpc_str]
    gpc_top = top_level_map[gpc_str]
    gpc_path_lower = gpc_path.lower()
    title_lower = str(title).lower()
    ptype_lower = str(product_type).lower()
    context = f"{title_lower} {ptype_lower}"
    age = str(age_group).lower().strip()

    # CHECK 3: Top-level category coherence
    # If the product is clearly in one vertical but got a GPC from another, that's wrong.
    # Example: a bedsheet (Home & Garden) should not get an Apparel GPC.
    VERTICAL_SIGNALS = {
        'Apparel & Accessories': ['shirt', 'blouse', 'dress', 'pants', 'jeans', 'jacket', 'coat',
            'shoe', 'sneaker', 'boot', 'sock', 'underwear', 'bra', 'lingerie', 'hat', 'scarf',
            'kleid', 'hose', 'jacke', 'schuh', 'hemd', 'jurk', 'broek', 'jas', 'schoen'],
        'Home & Garden': ['curtain', 'bedding', 'pillow', 'mattress', 'towel', 'rug', 'carpet',
            'candle', 'vase', 'lamp', 'sofa', 'chair', 'table', 'shelf', 'cabinet',
            'vorhang', 'kissen', 'matratze', 'teppich', 'gardine', 'bettbezug', 'spannbett',
            'gordijn', 'kussen', 'matras', 'tapijt', 'dekbed', 'laken', 'handdoek'],
        'Furniture': ['sofa', 'couch', 'bed ', 'desk', 'bookcase', 'wardrobe', 'dresser',
            'bett', 'schreibtisch', 'schrank', 'regal', 'bank', 'kast', 'bureau'],
    }

    # Determine what vertical the PRODUCT belongs to (from title/product_type)
    product_vertical = None
    for vertical, signals in VERTICAL_SIGNALS.items():
        if any(sig in context for sig in signals):
            product_vertical = vertical
            break

    # If we detected a vertical AND the GPC top-level doesn't match, flag it
    if product_vertical and gpc_top != product_vertical:
        # Special cases: Furniture is a subset of Home & Garden in some contexts
        if not (product_vertical == 'Furniture' and gpc_top == 'Home & Garden'):
            if not (product_vertical == 'Home & Garden' and gpc_top == 'Furniture'):
                return '', f'gpc_vertical_mismatch: product seems like {product_vertical} but GPC is in {gpc_top}'

    # CHECK 4: Age-restricted categories
    # Lingerie, adult underwear, nightwear → not for children
    ADULT_ONLY_PATHS = ['lingerie', 'underwear > bra', 'sleepwear & loungewear > nightgown']
    if age in ('kids', 'infant', 'toddler', 'newborn'):
        if any(adult_path in gpc_path_lower for adult_path in ADULT_ONLY_PATHS):
            return '', f'gpc_age_mismatch: age_group={age} but GPC path contains adult-only category'

    # CHECK 5: GPC numeric format
    if not gpc_str.isdigit():
        return '', f'gpc_not_numeric: {gpc_str}'

    return gpc_str, 'valid'
```

## Step 5: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype', 'product type']), None)
gpc_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie', 'google product category']), None)
age_col = next((c for c in df.columns if c in ['age_group', 'age group', 'leeftijdsgroep']), None)
gender_col = next((c for c in df.columns if c in ['gender', 'geslacht']), None)

learned_map = build_learned_map(df, gpc_col, ptype_col, title_col)

results = {'classified': 0, 'unresolved': 0, 'already_had': 0}
gpc_values = []

for idx, row in df.iterrows():
    # Check existing value — NEVER touch existing GPCs, always keep as-is
    existing = str(row.get(gpc_col, '')).strip() if gpc_col else ''
    if existing and existing.lower() not in ['', 'nan', 'none']:
        gpc_values.append(existing)
        results['already_had'] += 1
        continue

    # Classify using cascade
    cat = None

    # Layer 1: Learned map
    ptype = str(row.get(ptype_col, '')).lower().strip() if ptype_col else ''
    if ptype in learned_map:
        cat = learned_map[ptype]

    # Layer 2: Product type → taxonomy matching
    if not cat and ptype_col:
        cat, conf = match_product_type_to_taxonomy(row.get(ptype_col, ''))

    # Layer 3: Claude's classification based on title + description
    # (Claude applies its understanding of the product here)

    # Validate before accepting
    if cat:
        validated, reason = validate_gpc(
            cat,
            row.get(title_col, ''),
            row.get(ptype_col, ''),
            row.get(age_col, '') if age_col else '',
            row.get(gender_col, '') if gender_col else ''
        )
        if validated:
            gpc_values.append(validated)
            results['classified'] += 1
        else:
            gpc_values.append('')
            results['cleared'] += 1
    else:
        gpc_values.append('')
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'google_product_category': gpc_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_google_product_category.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Step 6: Report

Present summary with:
- Fill rate before/after
- How many products already had GPCs (kept as-is)
- Distribution of newly assigned categories (top 10 with taxonomy paths)
- Products that couldn't be classified (need manual review)
- Newly assigned GPCs that were cleared by validation and why

## Important guardrails

- **Never overwrite existing google_product_category values.** If the merchant already has a GPC, keep it as-is, always. Even if it looks wrong — it's the merchant's data. This skill only fills empty cells.
- **Always fetch and use the official taxonomy.** Never classify from memory or hardcoded lists. The taxonomy is the single source of truth.
- **Always validate newly assigned GPCs.** Every GPC that this skill assigns must pass validation against the taxonomy. A wrong GPC is worse than no GPC.
- OUTPUT NUMERIC IDs (not text paths) — they are stable, language-neutral, and unambiguous
- Always choose the MOST SPECIFIC category possible — "Shoes > Athletic Shoes" beats "Shoes"
- Classify by MAIN FUNCTION, not marketing angle ("gaming chair" = Furniture > Chairs)
- If no category fits well, leave empty — Google auto-assigns categories anyway
- The learned_map strategy (Layer 1) is the most reliable — it uses the merchant's own existing categorization
- **Clearing a wrong value is always better than keeping it.** An empty cell means Google falls back to auto-classification. A wrong cell actively misleads.

## Output format

- **Single attribute**: 2 columns → `id` + `google_product_category`
- Values should be numeric IDs (e.g., `3580`) not text paths
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)**
- Include ALL products (also empty ones)
