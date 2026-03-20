---
name: productfeed-is-bundle
description: Determine and fill the is_bundle attribute in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [is_bundle] values. Also triggers when user mentions "bundle attribute", "is pakket", "bundel", or asks about bundle detection. Use this skill even if the user just says "fill in the bundle flag" with an uploaded file.
---

# Feed Bundle Detector

Determine whether products are bundles (multiple different products sold together) and fill the `is_bundle` attribute.

## Why this matters

The `[is_bundle]` attribute tells Google the product is a merchant-defined bundle. Required when selling bundles to:
- Avoid disapprovals for price mismatches
- Ensure correct product representation
- Comply with Google's bundling policies

## Valid values

- `true` — product is a bundle of multiple different items
- `false` — product is a single item (default)

Note: a bundle is NOT a multipack. A bundle = different products together (e.g., camera + lens + bag). A multipack = same product multiple times (e.g., 3-pack socks).

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

Uses a **conservative** approach — only flags products as bundles when there is strong, unambiguous evidence in the **title**. The description is deliberately NOT scanned for bundle signals because common words like "set" appear in too many non-bundle contexts (e.g., German "gesetzt", "festsetzt", "gesetzliche Garantie", or furniture descriptions mentioning "set" as part of room staging).

**Key principle: `is_bundle=false` is always safe. `is_bundle=true` when wrong causes price mismatch warnings and potential disapprovals. When in doubt, leave as false.**

```python
import re

# ═══════════════════════════════════════════════════════════════
# BUNDLE DETECTION: TITLE-ONLY, MULTI-WORD PATTERNS PREFERRED
# ═══════════════════════════════════════════════════════════════
#
# ⚠️ DELIBERATELY CONSERVATIVE: only flag as bundle when evidence is strong.
# A false negative (missing a bundle) is harmless — the product just shows normally.
# A false positive (flagging a non-bundle) causes price mismatch warnings in
# Google Merchant Center, because Google expects bundle prices to be higher
# than individual product prices.
#
# Words like "set", "kit", and "combinatie" are EXCLUDED as single-word triggers
# because they are too ambiguous:
#   - "Messer-Set" = could be a bundle OR a single product (knife set)
#   - "Bettwäsche-Set" = usually a single product with matching pieces
#   - "Kit" = could be a repair kit (single product)
#   - "Combinatie" = could describe a style combination, not a bundle
#
# Description scanning is DISABLED because:
#   - German "gesetzt", "festsetzt", "gesetzliche" contain "set" as substring
#   - Furniture descriptions mention staging with other products
#   - Product care instructions mention "sets" of tools
#   - Far too many false positives vs. the few real bundles found this way

BUNDLE_MULTI_WORD = {
    # Dutch
    'startpakket', 'starterset', 'compleet pakket', 'complete set',
    'actie-pakket', 'voordeelpakket', 'combideal', 'combi-deal',
    # English
    'starter kit', 'starter pack', 'combo deal', 'complete kit',
    'accessory kit', 'value pack',
    # German
    'komplettpaket', 'komplett-set', 'kombi-paket', 'starterset',
    'vorteilspaket', 'aktionspaket',
}

# Single-word triggers — ONLY very unambiguous words
# "set", "kit", "combinatie" are deliberately EXCLUDED (too many false positives)
BUNDLE_SINGLE_WORDS = {
    'bundel', 'bundle',
    'pakket', 'paket',       # Dutch/German for "package/bundle"
    'combideal',
    'voordeelpakket',
    'vorteilspaket',
}

# Numbered bundle patterns in title: "3er-Set", "5-teiliges Set", "Set von 4"
BUNDLE_NUMBERED_PATTERNS = [
    re.compile(r'\b\d+\s*[-]?\s*er[-\s]+set\b', re.I),           # "3er-Set", "5er Set"
    re.compile(r'\bset\s+(?:van|von|of)\s+\d+\b', re.I),         # "Set von 3", "Set van 4"
    re.compile(r'\b\d+\s*[-]?\s*teilig\w*\s+set\b', re.I),       # "5-teiliges Set"
]

def _scan_for_bundle(text):
    """Scan text for bundle signals. Returns True if found.
    ONLY call this on TITLE text — never on descriptions."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return False

    # Check numbered bundle patterns first (highest confidence)
    for pattern in BUNDLE_NUMBERED_PATTERNS:
        if pattern.search(text_lower):
            return True

    # Multi-word patterns (high confidence)
    for kw in BUNDLE_MULTI_WORD:
        if kw in text_lower:
            return True

    # Single-word matches (only unambiguous words, whole-word match)
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    if words & BUNDLE_SINGLE_WORDS:
        return True

    return False

def extract_is_bundle(title, description='', product_type='', labels='',
                      google_category='', bullet_points=''):
    """Extract is_bundle. Returns (value, source, confidence).

    ⚠️ CONSERVATIVE: only scans the TITLE for bundle signals.
    Description is deliberately NOT scanned due to extreme false positive rates.
    Product type and labels are also skipped — bundle info belongs in the title."""

    # --- ONLY scan the title ---
    if _scan_for_bundle(title):
        return 'true', 'title', 'high'

    # --- Default false ---
    return 'false', 'default (no bundle signals in title)', 'high'
```

### Step 3: Apply and build supplemental feed

```python
bundle_col = next((c for c in df.columns if c in ['is_bundle', 'is pakket', 'g:is_bundle', 'bundle']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

results = {'filled': 0, 'already_had': 0}
bundle_values = []

for idx, row in df.iterrows():
    if bundle_col:
        current = str(row[bundle_col]).strip() if pd.notna(row[bundle_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            bundle_values.append(current)
            continue

    val, source, confidence = extract_is_bundle(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))
    bundle_values.append(val)
    results['filled'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'is_bundle': bundle_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_is_bundle.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- Default is `false` — most products are not bundles
- **A false `is_bundle=true` is far worse than a false `is_bundle=false`.** Wrong bundle flags cause price mismatch warnings in Google Merchant Center. When in doubt, leave as false.
- Don't confuse bundles with multipacks: "3-pack sokken" = multipack, "camera + tas + statief" = bundle
- **"Set" is deliberately excluded as a single-word trigger** — it's too ambiguous across languages. A "Messer-Set" (knife set) is typically a single product, not a merchant-defined bundle. Only match "set" when combined with a number ("3er-Set", "Set von 4") or as part of a known bundle phrase ("complete set", "starterset").
- **Description scanning is disabled** — words like "set" appear in too many non-bundle contexts in descriptions (German "gesetzt"/"gesetzliche", furniture staging descriptions, care instructions). Only the title is scanned.
- Never overwrite existing values

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `is_bundle`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
