---
name: productfeed-multipack
description: Extract and fill missing multipack attributes in Google Shopping product feeds. Triggers when user uploads a product feed and wants to populate [multipack] values. Also triggers when user mentions "multipack attribute", "multipack invullen", "verpakkingseenheid", or asks about multipack detection. Use this skill even if the user just says "fill in the multipack" with an uploaded file.
---

# Feed Multipack Detector

Extract multipack quantities from product titles and descriptions to fill the `multipack` attribute.

## Why this matters

The `[multipack]` attribute tells Google how many identical items are in the package. Required when selling multipacks to:
- Ensure correct price-per-unit comparisons
- Avoid disapprovals for price mismatches vs single items
- Improve ad accuracy

## Valid values

A whole number indicating how many identical items: `2`, `3`, `4`, `6`, `12`, etc.
Leave empty if the product is a single item (which is the vast majority).

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

Uses a **conservative** approach — only flags products as multipacks when there is strong, unambiguous evidence. Many numeric patterns in product titles are NOT multipacks:
- "3-Sitzer" (3-seater sofa) = NOT a multipack
- "4-teiliges Sofa" (4-piece modular sofa) = NOT a multipack  
- "5-Zonen-Matratze" (5-zone mattress) = NOT a multipack
- "18 Latten" (18 slats in a bed frame) = NOT a multipack
- "60 capsules" (dosage count) = NOT a multipack

**Key principle: an empty multipack field is always safe. A wrong multipack value causes price-per-unit mismatches and potential disapprovals.**

```python
import re

# ═══════════════════════════════════════════════════════════════
# MULTIPACK DETECTION: CONSERVATIVE, TITLE-ONLY
# ═══════════════════════════════════════════════════════════════
#
# ⚠️ "-teilig" / "-delig" are EXCLUDED — they mean "consisting of X parts"
# (e.g., a 4-piece modular sofa, a 24-piece cutlery set). These are single
# products with multiple components, NOT multipacks of identical items.
#
# A multipack is specifically: the same identical product × N units.
# "3-pack sokken" = 3 identical pairs → multipack=3
# "3-delig bankstel" = 1 sofa in 3 modules → NOT a multipack
# "5-teiliges Frottee-Set" = 1 set of 5 different towels → NOT a multipack (it's a bundle)

# Patterns that reliably indicate a multipack
MULTIPACK_PATTERNS = [
    # "3-pack", "6pack", "3 pack" — reliable multipack signal
    re.compile(r'\b(\d+)\s*[-]?\s*pack\b', re.I),
    # "3 stuks", "6 stuks", "3 Stück" — explicit count of identical items
    re.compile(r'\b(\d+)\s+(?:stuks?|stücke?)\b', re.I),
    # "3 pieces", "6 pieces" — explicit count
    re.compile(r'\b(\d+)\s+pieces?\b', re.I),
    # "3er-Pack", "6er Pack" — German multipack pattern
    re.compile(r'\b(\d+)\s*[-]?\s*er[-\s]+pack\b', re.I),
    # "verpakking van 3" — Dutch packaging pattern
    re.compile(r'verpakking\s+van\s+(\d+)\b', re.I),
    # "Packung mit 3" — German packaging pattern
    re.compile(r'packung\s+(?:mit|von)\s+(\d+)\b', re.I),
]

# ⚠️ REMOVED PATTERNS (too many false positives):
# - r'(\d+)\s*[-]?\s*teilig'  → "4-teiliges Sofa" is modular furniture, not multipack
# - r'(\d+)\s*[-]?\s*delig'   → "6-delig bestek" is a single cutlery set, not multipack
# - r'\b(\d+)x\b'             → "3x zoom", resolution patterns, dosage "3x täglich"
# - r'per\s+(\d+)'            → "per 3 stuks" is ambiguous, could be pricing info
# - r'set\s+(?:van|of)\s+(\d+)' → sets are bundles, not multipacks

WORD_MULTIPACKS = {
    'duopack': '2', 'doppelpack': '2', 'dubbelpak': '2', 'tweepak': '2',
    'triopack': '3', 'driepack': '3', 'dreierpack': '3',
}
# ⚠️ 'duo' and 'trio' are EXCLUDED as standalone words — they appear in
# product names ("Duo Matratze", "Trio Kinderwagen") where they describe
# a product feature, not a multipack quantity.

# Patterns that should NEVER be treated as multipack, even if they contain numbers
# These are checked BEFORE multipack patterns to prevent false positives
MULTIPACK_EXCLUSIONS = re.compile(
    r'\b\d+\s*[-]?\s*(?:'
    r'sitzer|seitig|zonen|zone|latten|'           # furniture: 3-Sitzer, 5-Zonen
    r'teilig|delig|'                                # modular: 4-teilig, 6-delig
    r'personen?|pers\.?|'                           # capacity: 6 Personen
    r'türig|schublade|fächer|fach|'                 # furniture features: 2-türig
    r'flammig|armig|'                               # lighting: 3-flammig
    r'cm|mm|m\b|kg|g\b|ml|l\b'                     # dimensions/measurements
    r')\b',
    re.I
)

def _scan_for_multipack(text):
    """Scan text for multipack signals. Returns quantity string or None.
    CONSERVATIVE: checks exclusions first, only matches unambiguous patterns."""
    text_str = str(text) if pd.notna(text) else ''
    text_lower = text_str.lower()
    if not text_lower or text_lower == 'nan':
        return None

    # Check word-based multipacks first (highest confidence)
    words = set(re.findall(r'[a-zà-ÿ-]+', text_lower))
    for word, qty in WORD_MULTIPACKS.items():
        if word in words:
            return qty

    # Check regex patterns
    for pattern in MULTIPACK_PATTERNS:
        match = pattern.search(text_lower)
        if match:
            qty = match.group(1)
            qty_int = int(qty)
            # Sanity check: multipack is typically 2-48
            # (>48 is almost certainly a dosage count or product feature)
            if 2 <= qty_int <= 48:
                # Double-check: is the matched number part of an exclusion pattern?
                # e.g., "3-pack" is fine, but "3-sitzer" near "3-pack" could confuse
                # Check the immediate context around the match
                start = max(0, match.start() - 5)
                end = min(len(text_lower), match.end() + 15)
                context = text_lower[start:end]
                if not MULTIPACK_EXCLUSIONS.search(context):
                    return qty
    return None

def extract_multipack(title, description='', product_type='', labels='',
                      google_category='', bullet_points=''):
    """Extract multipack quantity. Returns (quantity, source, confidence).

    ⚠️ CONSERVATIVE: only scans the TITLE for multipack signals.
    Description scanning is disabled due to high false positive rates in
    product descriptions that mention quantities in other contexts
    (care instructions, dosage, product specifications)."""

    # --- ONLY scan the title ---
    qty = _scan_for_multipack(title)
    if qty:
        return qty, 'title', 'high'

    # --- Not a multipack (leave empty) ---
    return None, 'none', 'unresolved'
```

### Step 3: Apply and build supplemental feed

```python
mp_col = next((c for c in df.columns if c in ['multipack', 'g:multipack']), None)
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
label_col = next((c for c in df.columns if c in ['labels', 'aangepast label 1', 'custom_label_0']), None)
gcat_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)
bp_col = next((c for c in df.columns if c in ['product bullet point 1', 'product_highlight']), None)

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}
mp_values = []

for idx, row in df.iterrows():
    if mp_col:
        current = str(row[mp_col]).strip() if pd.notna(row[mp_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            mp_values.append(current)
            continue

    qty, source, confidence = extract_multipack(
        row.get(title_col, ''), row.get(desc_col, ''),
        row.get(ptype_col, ''), row.get(label_col, ''),
        row.get(gcat_col, ''), row.get(bp_col, ''))
    mp_values.append(qty if qty else '')
    if qty:
        results['filled'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'multipack': mp_values
})

output_path = "/mnt/user-data/outputs/supplemental_feed_multipack.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Important guardrails

- **An empty multipack field is always safe. A wrong multipack value causes price-per-unit mismatches.** When in doubt, leave empty.
- Most products are NOT multipacks — leave empty (not "1")
- Don't confuse multipacks with bundles: "3-pack sokken" (same item ×3) = multipack. "Camera + lens + tas" = bundle.
- **"-teilig" / "-delig" are NOT multipack indicators.** "4-teiliges Sofa" = 1 modular sofa with 4 sections. "6-delig bestek" = 1 cutlery set with 6 pieces. These are single products, not multipacks.
- **"-Sitzer" is NOT a multipack indicator.** "3-Sitzer" = a 3-seat sofa, not 3 identical sofas.
- **Furniture terms like "Zonen" (zones), "Latten" (slats), "Personen" (person capacity) are NOT multipack quantities.**
- Watch for false positives: "3x zoom" is not a multipack, "1920x1080" is not a multipack, "60 capsules" is a dosage count not a multipack
- Sanity check: quantities should be 2-48. Anything higher is almost certainly a dosage count, product feature, or dimensional value.
- **Only scan titles, not descriptions.** Product descriptions contain too many numeric-plus-word patterns in non-multipack contexts (care instructions, specifications, marketing copy).

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `multipack`
- **Multiple attributes**: 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
