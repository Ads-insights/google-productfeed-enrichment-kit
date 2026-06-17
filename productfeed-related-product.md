---
name: productfeed-related-product
description: Generate related_product attributes for Google Shopping product feeds (the product-relationship attribute used in Google's conversational AI / AI Mode shopping experience). Links products within the same feed as part_of_set, accessory, often_bought_with, substitute, etc., using conservative cross-product heuristics. Triggers when user uploads a product feed and wants to populate [related_product] values. Also triggers when user mentions "related product", "related_product", "accessoires koppelen", "cross-sell feed", "product relationships", "often bought with", "substitute products", or "conversational shopping attributes". Use this skill even if the user just says "link related products" with an uploaded file.
---

# Feed Related Product Generator

Generate `related_product` values — relationships between products in your inventory (accessories, spare parts, substitutes, set members) that Google uses in conversational AI to suggest products to buy together with, or instead of, the current one.

## Why this matters

The `[related_product]` attribute powers cross-sell and substitution suggestions in Google's conversational AI. It lets you steer "what goes with this" and "what's a good alternative" instead of leaving Google to guess from co-views.

## This is a cross-product skill, not a per-row extraction

Unlike most feed skills, a relationship can't be read off a single row — it's a link between two products. This skill reasons across the whole feed using conservative heuristics, and flags confidence. It never asserts a relationship it can't ground in shared structured data.

## Google's specifications and rules

**Type:** Group attribute with 3 required sub-attributes.

| Sub-attribute | Required | Values |
|---|---|---|
| `relationship_type` | Yes | one of: `part_of_set`, `required_part`, `often_bought_with`, `substitute`, `different_brand`, `accessory` |
| `identifier_type` | Yes | `gtin` or `id` |
| `identifier` | Yes | the GTIN or the product `id` of the related product |

**Relationship type meanings (from Google):**
- `part_of_set` — same series/product line (a chair belonging to a table).
- `required_part` — necessary for the product to function (a battery for a battery-operated lamp).
- `often_bought_with` — frequently purchased together (a phone case with a phone).
- `substitute` — can be substituted for this product (a comparable printer).
- `different_brand` — identical product under a different brand (a cheaper house brand).
- `accessory` — an accessory to this product (a webcam for a desktop).

**Minimum requirements:**
- Submit all 3 sub-attributes.
- One `related_product` entry per related product. If a relationship type has multiple related products, repeat the attribute — **do not** comma-separate identifiers inside the `identifier` sub-attribute.
- `identifier` may only contain alphanumeric, underscores, and dashes. **No commas, colons, double quotes, or backslashes** — so no escaping is needed (and any id containing those chars is invalid and must be skipped).

**Repeated field:** yes, up to 30 entries per product.

## Output format

A product's relationships go in one supplemental cell. In Google Sheets group-attribute format each entry is `relationship_type:identifier_type:identifier`, entries comma-separated:

`required_part:id:AZ7A,required_part:id:AZ7B,accessory:gtin:811571013579`

Because identifiers can't contain reserved characters, **no escaping is required** — but validate that the identifier is clean before emitting.

```python
import re

def build_related_cell(entries):
    """entries = list of (relationship_type, identifier_type, identifier)."""
    parts = []
    for rel, idtype, ident in entries[:30]:
        ident = str(ident).strip()
        if not re.match(r'^[A-Za-z0-9_-]+$', ident):
            continue  # invalid identifier per Google spec — skip
        parts.append(f"{rel}:{idtype}:{ident}")
    return ','.join(parts)
```

## Workflow

### Step 1: Load the feed

```python
import pandas as pd

file_path = "/mnt/user-data/uploads/<filename>"
if file_path.endswith('.zip'):
    import zipfile
    with zipfile.ZipFile(file_path) as z:
        inner = [f for f in z.namelist() if f.endswith(('.tsv', '.csv', '.xlsx'))][0]
        if inner.endswith('.tsv'):
            df = pd.read_csv(z.open(inner), sep='\t', dtype=str)
        elif inner.endswith('.csv'):
            df = pd.read_csv(z.open(inner), dtype=str)
        else:
            df = pd.read_excel(z.open(inner), dtype=str)
elif file_path.endswith('.csv'):
    df = pd.read_csv(file_path, dtype=str)
elif file_path.endswith('.tsv'):
    df = pd.read_csv(file_path, sep='\t', dtype=str)
else:
    df = pd.read_excel(file_path, dtype=str)

df.columns = [c.strip().lower() for c in df.columns]
```

### Step 2: Map the columns needed for relationship reasoning

```python
def col(df, *names):
    for n in names:
        if n in df.columns:
            return n
    return None

ID   = col(df, 'id', 'unique merchant sku', 'merchant item id', 'g:id')
IGID = col(df, 'item_group_id', 'g:item_group_id', 'groeps id', 'parent sku', 'parent_sku')
BRND = col(df, 'brand', 'merk', 'manufacturer')
PT   = col(df, 'product_type', 'producttype')
GPC  = col(df, 'google_product_category', 'google productcategorie')
GTIN = col(df, 'gtin', 'ean', 'upc')
PRICE= col(df, 'price', 'prijs')
TTL  = col(df, 'title', 'titel', 'product name')
```

### Step 3: Conservative relationship heuristics

Only emit relationships that are grounded in shared structured data. Each heuristic carries a confidence level; only `high` and `medium` are written, `low` is flagged but not emitted by default.

**Heuristic A — `part_of_set` (high confidence)**
Products sharing the same `item_group_id` are variants of the same product, NOT a set. Do **not** treat variants as part_of_set. Reserve `part_of_set` for an explicit "series"/"collection"/"set" signal: a shared `collection`/`series` column, or identical normalized base title across different item_group_ids within the same brand. Without such a column, skip — variants are not sets.

**Heuristic B — `substitute` (medium confidence)**
Same `google_product_category` (or same leaf of `product_type`) + same `brand` + similar price band (within ±25%) + different `item_group_id`. These are genuine alternatives. Cap to the top N closest by price to avoid linking an entire category.

**Heuristic C — `different_brand` (medium confidence)**
Same leaf `product_type`/`GPC` + very similar normalized title + **different** brand. Identical product, different label.

**Heuristic D — `accessory` / `required_part` / `often_bought_with` (low–medium)**
These need real semantics (a case for a phone). Pure feed heuristics are weak here. Only emit when the feed has an explicit signal — e.g. a `related_skus`, `accessories`, `cross_sell`, or `bought_together` column listing target ids/GTINs. Map those directly to the right relationship type. Without such a column, do not invent accessory links.

```python
import re

def norm_title(t):
    t = str(t).lower()
    t = re.sub(r'\b(\d+)\s*(gb|tb|cm|mm|ml|l|kg|g)\b', '', t)  # strip variant tokens
    t = re.sub(r'[^a-z0-9 ]', ' ', t)
    return ' '.join(t.split())

def price_num(v):
    try:
        return float(re.sub(r'[^0-9.,]', '', str(v)).replace(',', '.'))
    except Exception:
        return None
```

### Step 4: Detect explicit relationship columns first

If the feed already lists related ids (most reliable), use them and skip heuristics for those types.

```python
REL_COL_MAP = {
    'accessory':        ['accessories', 'accessory_skus', 'accessoires'],
    'required_part':    ['required_parts', 'spare_parts', 'onderdelen'],
    'often_bought_with':['bought_together', 'cross_sell', 'frequently_bought', 'vaak_samen_gekocht'],
    'substitute':       ['substitutes', 'alternatives', 'alternatieven'],
    'part_of_set':      ['set_members', 'collection_skus', 'serie'],
}
# For each present column, split on common delimiters (, ; |) into target identifiers.
```

### Step 5: Build supplemental feed

```python
# Build lookup structures once
by_igid = df.groupby(IGID)[ID].apply(list).to_dict() if IGID else {}
id_set = set(df[ID].dropna().astype(str))

related_values = []
flags = []
for idx, row in df.iterrows():
    entries = []          # list of (rel, idtype, identifier)
    # 1) explicit columns (high confidence)
    # 2) heuristic B/C (medium) — append with dedupe
    # ... apply heuristics, respect ±25% price band, cap counts per type ...
    # Only keep entries whose identifier exists in id_set (for id type) or is a clean GTIN.
    related_values.append(build_related_cell(entries))

id_col = ID
supplemental = pd.DataFrame({'id': df[id_col], 'related_product': related_values})
output_path = "/mnt/user-data/outputs/supplemental_feed_related_product.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 6: Report

- Which method produced links: explicit columns vs heuristics (and which heuristics).
- Count of products with ≥1 relationship, and breakdown per relationship_type.
- 10–15 spot-check examples showing the source product and its linked products' titles (resolve ids back to titles for readability).
- Confidence note: heuristic substitute/different_brand links are suggestions; explicit-column links are reliable.

## Important guardrails

- **Never link a product to an id/GTIN that isn't in the feed** (for `id` type) — Google can't resolve it.
- **Variants are not sets.** Same `item_group_id` ⇒ variants, not `part_of_set`.
- **Don't comma-separate identifiers inside one entry** — one entry per related product, repeat the attribute.
- **Identifier must be alphanumeric/underscore/dash only.** Skip any id with reserved characters.
- Prefer explicit feed columns over heuristics; flag heuristic links as medium confidence.
- Cap entries per type and total at 30 to avoid linking entire categories.
- Never overwrite an existing `related_product` value.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `related_product` (entries comma-separated, format `relationship_type:identifier_type:identifier`)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell where no relationship was found)
