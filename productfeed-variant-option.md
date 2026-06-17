---
name: productfeed-variant-option
description: Generate variant_option attributes for Google Shopping product feeds (the variant-identifying name/value pairs used in Google's conversational AI / AI Mode and Search experiences). Extracts each variant's distinguishing dimensions (size, color, material, or custom dimensions like memory/display) into name:value pairs. Triggers when user uploads a product feed and wants to populate [variant_option] values. Also triggers when user mentions "variant option", "variant_option", "variant dimensions", "variantkenmerken", "variant eigenschappen", or "conversational shopping attributes". Use this skill even if the user just says "fill in the variant options" with an uploaded file.
---

# Feed Variant Option Generator

Generate `variant_option` values — the variant-identifying properties of a product (e.g. `size:8`, `color:black`, `memory:128GB`) that Google uses to show a product together with all its variants in conversational AI and Search.

## Why this matters

`[variant_option]` makes a product's variant dimensions explicit and machine-readable, beyond the standard `color`/`size` attributes (which also apply to non-variant products). It is what lets Google present a clean variant picker and reason about variants in conversational AI.

## Google's specifications and rules

**Type:** Group attribute with 2 sub-attributes.

| Sub-attribute | Required | Limit |
|---|---|---|
| `name` | Yes | Text, max 250 characters |
| `value` | Yes | Text, max 250 characters |

- **Repeated field:** yes, up to 30 (one per variant dimension).
- Submit alongside `item_group_id` and `item_group_title`.

**Minimum requirements:**
- Submit both `name` and `value` for each dimension.
- Every variant sharing an `item_group_id` must use the **same set of `name` sub-attributes**. Variants are distinguished by a unique combination of `value`s.
- The variant details shown on the landing page must match the submitted values.

**Best practices:**
- Don't submit `variant_option` for non-variant products.
- Submit with `item_group_title` + `item_group_id`.
- If the product varies by the standard attributes (color, pattern, material, age_group, gender, size), also express those exact variant dimensions here.

## Output format and escaping

A product's variant dimensions go in one supplemental cell. Google Sheets group-attribute format: each pair is `name:value`, pairs comma-separated:

`shoe width:narrow,size:8`  ·  `cpu:ultra,display:max,memory:128GB`

Reserved characters (`:`, `,`, `\`) inside a `name` or `value` must be escaped with a backslash. Variant values rarely contain them, but escape defensively.

```python
def escape_vo(text):
    text = str(text).strip()
    text = text.replace('\\', '\\\\')
    text = text.replace(':', '\\:')
    text = text.replace(',', '\\,')
    return text

def build_variant_cell(pairs):
    """pairs = list of (name, value)."""
    return ','.join(f"{escape_vo(n)}:{escape_vo(v)}" for n, v in pairs[:30])
```

## This skill depends on item_group_id

`variant_option` only applies to grouped variants. Requires an `item_group_id` column (or run after `productfeed-item-group-id`). The critical constraint — **same set of `name`s across all variants in a group** — is enforced per group.

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

### Step 2: Map columns and require a group id

```python
def col(df, *names):
    for n in names:
        if n in df.columns:
            return n
    return None

ID   = col(df, 'id', 'unique merchant sku', 'merchant item id', 'g:id')
IGID = col(df, 'item_group_id', 'g:item_group_id', 'groeps id', 'parent sku', 'parent_sku')
VO   = col(df, 'variant_option', 'g:variant_option')

# Standard variant-defining attributes, with the canonical name Google expects
STD_DIMS = {
    'color':     col(df, 'color', 'kleur', 'colour'),
    'size':      col(df, 'size', 'grootte', 'maat'),
    'material':  col(df, 'material', 'materiaal'),
    'pattern':   col(df, 'pattern', 'patroon'),
    'age_group': col(df, 'age_group', 'leeftijdsgroep'),
    'gender':    col(df, 'gender', 'geslacht'),
}
STD_DIMS = {k: v for k, v in STD_DIMS.items() if v}
```

If `IGID` is missing: report that `variant_option` needs grouped variants; suggest running `item_group_id` first.

### Step 3: Determine each group's variant dimensions

A dimension belongs to a group only if its value **varies within that group**. An attribute identical across all variants is not a variant dimension (it's a product property), and must be excluded — otherwise the "same set of names, unique combination of values" rule breaks.

```python
def group_dimensions(gdf, std_dims):
    """Return the list of dimension keys whose value actually varies within the group."""
    dims = []
    for key, c in std_dims.items():
        vals = set(
            str(v).strip().lower()
            for v in gdf[c].dropna()
            if str(v).strip() and str(v).lower() not in ('nan', 'none')
        )
        if len(vals) >= 2:        # varies → it's a variant dimension
            dims.append(key)
    return dims
```

Note on custom dimensions (memory, display, cpu): these usually aren't separate feed columns. If the feed has explicit columns for them, include them the same way. Otherwise, do **not** try to parse them out of the title — that's unreliable and risks inconsistent name sets across the group. Stick to columns that exist.

### Step 4: Build per-variant pairs with a consistent name set

```python
results = {'generated': 0, 'singletons': 0, 'no_varying_dim': 0, 'already_had': 0}
vo_values = [''] * len(df)
df_reset = df.reset_index(drop=True)

# Map original index → position for assignment
pos_by_id = {row_id: i for i, row_id in enumerate(df_reset[ID])}

for igid, gdf in df_reset.groupby(IGID):
    if not str(igid).strip() or str(igid).lower() in ('nan', 'none'):
        continue
    if len(gdf) < 2:
        results['singletons'] += 1
        continue

    dims = group_dimensions(gdf, STD_DIMS)
    if not dims:
        results['no_varying_dim'] += 1
        continue

    for _, vrow in gdf.iterrows():
        # respect existing value
        if VO:
            cur = str(vrow[VO]).strip() if pd.notna(vrow[VO]) else ''
            if cur and cur.lower() not in ('nan', 'none'):
                vo_values[pos_by_id[vrow[ID]]] = cur
                results['already_had'] += 1
                continue
        pairs = []
        for key in dims:                       # SAME order & set for every variant
            c = STD_DIMS[key]
            val = str(vrow[c]).strip() if pd.notna(vrow[c]) else ''
            # Every variant must carry every name; if a value is missing, still emit the name
            # with an empty value is invalid — instead skip this variant from VO to keep the
            # group's name-set consistent, and flag it.
            if not val or val.lower() in ('nan', 'none'):
                pairs = None
                break
            pairs.append((key, val))
        if pairs:
            vo_values[pos_by_id[vrow[ID]]] = build_variant_cell(pairs)
            results['generated'] += 1

supplemental = pd.DataFrame({'id': df_reset[ID], 'variant_option': vo_values})
output_path = "/mnt/user-data/outputs/supplemental_feed_variant_option.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

- Variant groups processed; dimensions detected per group (e.g. "group TSHIRT-001 varies by color, size").
- 10–15 examples showing each variant's `variant_option` cell next to its title.
- Groups skipped: singletons, groups with no varying dimension, groups with inconsistent missing values (flagged).

## Important guardrails

- **Same set of `name`s for all variants in a group.** Only include a dimension if it varies within the group; if any variant lacks a value for a chosen dimension, skip that variant's VO and flag rather than emit an inconsistent name set.
- **Don't emit `variant_option` for non-variant products** (singletons / no varying dimension).
- **Don't parse custom dimensions out of titles** — only use real columns.
- Each `name` and `value` max 250 chars.
- Max 30 pairs per product.
- Escape `:`, `,`, `\` in names and values.
- Never overwrite an existing `variant_option`.
- Submit together with `item_group_id` and `item_group_title` (these are separate skills/columns).

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `variant_option` (pairs comma-separated, `name:value`)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell for non-variant products)
