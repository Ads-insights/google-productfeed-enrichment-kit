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
- Netherlands is one of 12 countries where category subdivision is available for campaign targeting

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

**Language options:**
- Numeric IDs work in all languages (recommended)
- English (en-US) taxonomy paths are universally accepted
- Dutch (nl-NL) taxonomy paths are accepted for NL feeds
- IDs are identical across all language versions

**Mandatory categories**: Apparel & Accessories, Media, and Software REQUIRE google_product_category to be set for proper attribute enforcement.

## Taxonomy reference

The full taxonomy is available at:
- With IDs (EN): https://www.google.com/basepages/producttype/taxonomy-with-ids.en-US.txt
- With IDs (NL): https://www.google.com/basepages/producttype/taxonomy-with-ids.nl-NL.txt
- Without IDs (EN): https://www.google.com/basepages/producttype/taxonomy.en-US.txt

The taxonomy contains ~6000 categories organized hierarchically up to 7 levels deep. Version 2021-09-21 is the current version.

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

### Step 2: Classification strategy

This is a CLASSIFICATION skill — it maps products to a fixed taxonomy. The approach uses a 4-layer cascade:

**Layer 1: Learn from existing categories in the feed**
If some products already have a google_product_category, build a mapping from product_type/title keywords to categories. Apply this mapping to uncategorized products with similar characteristics.

**Layer 2: Map from product_type**
The product_type field often contains a merchant's own category hierarchy (e.g., "Wonen > Verlichting > Tafellampen"). Map these to the nearest Google taxonomy category.

**Layer 3: Keyword-based classification from title + description**
Use keyword matching against a curated mapping of common product keywords to Google taxonomy categories.

**Layer 4: Broad category fallback**
When only limited info is available, assign the broadest reasonable category rather than nothing.

```python
import re
from collections import Counter

# === Common keyword → Google category mappings ===
# These cover the most frequent Dutch webshop product types
# Format: (keywords_to_match, google_category_id, google_category_path)

KEYWORD_CATEGORY_MAP = [
    # Electronics
    (['smartphone', 'telefoon', 'mobile phone', 'iphone', 'samsung galaxy'],
     267, 'Electronics > Communications > Telephony > Mobile Phones'),
    (['laptop', 'notebook'], 
     328, 'Electronics > Computers > Laptops'),
    (['tablet', 'ipad'],
     4745, 'Electronics > Computers > Tablet Computers'),
    (['koptelefoon', 'headphone', 'earbuds', 'oordopjes', 'oortjes'],
     505794, 'Electronics > Audio > Audio Accessories > Headphones & Headsets'),
    (['speaker', 'luidspreker', 'bluetooth speaker'],
     236, 'Electronics > Audio > Speakers'),
    (['tv', 'televisie', 'television', 'monitor', 'beeldscherm'],
     404, 'Electronics > Video > Televisions'),
    (['camera', 'fotocamera', 'fototoestel'],
     152, 'Cameras & Optics > Cameras > Digital Cameras'),
    (['smartwatch', 'horloge', 'watch'],
     201, 'Apparel & Accessories > Jewelry > Watches'),
    (['stofzuiger', 'vacuum cleaner'],
     623, 'Home & Garden > Household Appliances > Vacuums'),
    (['wasmachine', 'washing machine'],
     620, 'Home & Garden > Household Appliances > Laundry Appliances > Washing Machines'),

    # Apparel & Accessories
    (['sneaker', 'sneakers', 'schoen', 'schoenen', 'shoes'],
     187, 'Apparel & Accessories > Shoes'),
    (['jurk', 'dress', 'jurken'],
     2271, 'Apparel & Accessories > Clothing > Dresses'),
    (['t-shirt', 'tshirt', 'shirt'],
     212, 'Apparel & Accessories > Clothing > Shirts & Tops'),
    (['broek', 'pants', 'jeans', 'trousers'],
     204, 'Apparel & Accessories > Clothing > Pants'),
    (['jas', 'jacket', 'coat', 'jas'],
     5598, 'Apparel & Accessories > Clothing > Outerwear > Coats & Jackets'),
    (['zonnebril', 'sunglasses'],
     178, 'Apparel & Accessories > Clothing Accessories > Sunglasses'),
    (['tas', 'bag', 'handtas', 'rugzak', 'backpack'],
     6551, 'Apparel & Accessories > Handbags, Wallets & Cases'),

    # Home & Garden
    (['meubel', 'furniture', 'kast', 'tafel', 'stoel', 'bank', 'sofa'],
     436, 'Home & Garden > Furniture'),
    (['lamp', 'verlichting', 'lighting', 'led lamp'],
     594, 'Home & Garden > Lighting > Lamps'),
    (['bed', 'matras', 'mattress'],
     4299, 'Home & Garden > Furniture > Beds & Accessories > Beds & Bed Frames'),
    (['kussen', 'pillow', 'cushion'],
     567, 'Home & Garden > Linens & Bedding > Pillows'),
    (['gordijn', 'curtain', 'raamdecoratie'],
     1723, 'Home & Garden > Decor > Window Treatments > Curtains & Drapes'),

    # Sports & Outdoors
    (['fiets', 'bike', 'bicycle', 'e-bike', 'ebike', 'elektrische fiets'],
     1025, 'Vehicles & Parts > Vehicle Parts & Accessories > Bicycle Parts & Accessories'),
    (['fitness', 'sport', 'gym', 'dumbbell', 'halter'],
     990, 'Sporting Goods > Exercise & Fitness'),

    # Food & Beverages
    (['koffie', 'coffee', 'thee', 'tea'],
     2427, 'Food, Beverages & Tobacco > Beverages'),
    (['wijn', 'wine', 'bier', 'beer'],
     499676, 'Food, Beverages & Tobacco > Beverages > Alcoholic Beverages'),

    # Health & Beauty
    (['parfum', 'perfume', 'eau de toilette'],
     2619, 'Health & Beauty > Personal Care > Cosmetics > Perfume & Cologne'),
    (['shampoo', 'conditioner', 'haarproduct'],
     2441, 'Health & Beauty > Personal Care > Hair Care'),
    (['crème', 'cream', 'moisturizer', 'huidverzorging', 'skincare'],
     2844, 'Health & Beauty > Personal Care > Cosmetics > Skin Care'),

    # Baby & Toddler
    (['kinderwagen', 'stroller', 'buggy'],
     568, 'Baby & Toddler > Baby Transport > Strollers'),
    (['luier', 'diaper'],
     553, 'Baby & Toddler > Diapering'),

    # Arts & Crafts / Memorial
    (['candle', 'kaars', 'waxmelt', 'geurkaars', 'scented candle'],
     3580, 'Arts & Entertainment > Hobbies & Creative Arts > Arts & Crafts'),

    # Pet Supplies
    (['hondenvoer', 'dog food', 'kattenvoer', 'cat food', 'huisdier', 'pet'],
     2, 'Animals & Pet Supplies > Pet Supplies'),

    # Vehicles
    (['auto-onderdeel', 'car part', 'autoband', 'tire'],
     5613, 'Vehicles & Parts > Vehicle Parts & Accessories'),
    (['scooter', 'elektrische scooter'],
     5190, 'Vehicles & Parts > Vehicles > Motor Vehicles > Motorcycles & Scooters'),
]


def build_category_map_from_feed(df, gpc_col, ptype_col, title_col):
    """Learn category assignments from products that already have a GPC.
    Returns a dict mapping product_type values to (category_id, category_path)."""
    learned_map = {}
    if not gpc_col:
        return learned_map

    has_gpc = df[gpc_col].notna() & (df[gpc_col].str.strip() != '') & (df[gpc_col].str.lower() != 'nan')
    if has_gpc.sum() == 0:
        return learned_map

    # Group by product_type and find most common GPC
    if ptype_col and ptype_col in df.columns:
        for ptype in df.loc[has_gpc, ptype_col].dropna().unique():
            ptype_clean = str(ptype).strip()
            if ptype_clean and ptype_clean.lower() not in ['', 'nan']:
                mask = has_gpc & (df[ptype_col] == ptype)
                if mask.sum() > 0:
                    most_common_gpc = df.loc[mask, gpc_col].mode().iloc[0]
                    learned_map[ptype_clean.lower()] = most_common_gpc

    return learned_map


def classify_product(title, description='', product_type='', labels='',
                     google_category_existing='', learned_map=None):
    """Classify a product into Google's taxonomy.
    Returns (category_value, source, confidence).
    category_value is the numeric ID (preferred) or full path."""

    title_lower = str(title).lower() if pd.notna(title) else ''
    desc_lower = str(description).lower() if pd.notna(description) else ''
    ptype_lower = str(product_type).lower() if pd.notna(product_type) else ''
    labels_lower = str(labels).lower() if pd.notna(labels) else ''

    # --- LAYER 1: Match from learned category map ---
    if learned_map and ptype_lower:
        if ptype_lower in learned_map:
            return learned_map[ptype_lower], 'learned from feed (product_type)', 'high'

    # --- LAYER 2: Keyword matching in title + product_type ---
    all_text = f"{title_lower} {ptype_lower} {labels_lower}"
    for keywords, cat_id, cat_path in KEYWORD_CATEGORY_MAP:
        for kw in keywords:
            if kw in all_text:
                return str(cat_id), f'keyword match: "{kw}"', 'medium'

    # --- LAYER 3: Keyword matching in description ---
    if desc_lower and desc_lower != 'nan':
        all_text_desc = f"{desc_lower}"
        for keywords, cat_id, cat_path in KEYWORD_CATEGORY_MAP:
            for kw in keywords:
                if kw in all_text_desc:
                    return str(cat_id), f'description keyword: "{kw}"', 'medium'

    # --- LAYER 4: No match found ---
    return '', 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gpc_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)

# Build learned map from existing data
learned_map = build_category_map_from_feed(df, gpc_col, ptype_col, title_col)

results = {'classified': 0, 'unresolved': 0, 'already_had': 0}
gpc_values = []

for idx, row in df.iterrows():
    if gpc_col:
        current = str(row[gpc_col]).strip() if pd.notna(row[gpc_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            gpc_values.append(current)
            continue

    cat, source, confidence = classify_product(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        '', learned_map)

    gpc_values.append(cat)
    if cat:
        results['classified'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'google_product_category': gpc_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_google_product_category.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 4: Report

Present summary with:
- Fill rate before/after
- Distribution of assigned categories (top 10)
- Sample of classified products for spot-checking
- Products that couldn't be classified (need manual review)
- Highlight any products that got broad categories where a more specific one might exist

## Important guardrails

- Never overwrite existing google_product_category values
- OUTPUT NUMERIC IDs (not text paths) — they are stable, language-neutral, and unambiguous
- Always choose the MOST SPECIFIC category possible — "Shoes > Athletic Shoes" beats "Shoes"
- Classify by MAIN FUNCTION, not marketing angle ("gaming chair" = Furniture > Chairs)
- If no category fits well, leave empty — a wrong category is worse than no category
- Google auto-assigns categories anyway — this skill is for overriding incorrect auto-assignments
- The keyword map in this skill covers common Dutch webshop products but is not exhaustive — expand it based on the specific feed being processed
- For feeds with many unique product types, the "learn from existing data" strategy is the most powerful approach

## Extending the keyword map

The KEYWORD_CATEGORY_MAP in this skill covers the most common product types but will need expanding for niche feeds. When processing a new feed:

1. Check which product_types exist in the feed
2. Look up the most specific Google category for each product_type
3. Add new keyword → category mappings as needed
4. The learned_map strategy automatically handles this for feeds with partial categorization

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `google_product_category`
- Values should be numeric IDs (e.g., `3580`) not text paths
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
