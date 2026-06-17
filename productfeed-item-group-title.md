---
name: productfeed-item-group-title
description: Generate item_group_title attributes for Google Shopping product feeds (the common variant-group title used in Google's conversational AI / AI Mode and Search experiences). Derives one shared, generic title per item_group_id by stripping variant tokens (color, size, etc.) from variant titles. Triggers when user uploads a product feed and wants to populate [item_group_title] values. Also triggers when user mentions "item group title", "item_group_title", "variant group title", "gemeenschappelijke titel", "groepstitel", or "conversational shopping attributes". Use this skill even if the user just says "fill in the group titles" with an uploaded file.
---

# Feed Item Group Title Generator

Generate `item_group_title` values — a single common title shared by all variants of a product, while each variant keeps its own specific `title`. Used in conversational AI and Search to present a product with its variants under one heading.

## Why this matters

When variants are grouped with `item_group_id`, the `[item_group_title]` gives Google one clean parent title (e.g. "Organic Cotton Men's T-Shirt") instead of forcing it to pick among variant titles like "...- Black - M". This improves how product groups surface in conversational AI and variant swatches in Search.

## Google's specifications and rules

| Property | Value |
|---|---|
| **Type** | String (Unicode; ASCII recommended) |
| **Limit** | 1–150 characters (Google truncates longer and warns) |
| **Repeated field** | No |
| **Schema.org** | `ProductGroup.name` |

**Minimum requirements:**
- **Same value for all variants** sharing an `item_group_id`. Every variant in a group must carry the identical `item_group_title`.
- Must be **different** from the individual variant `title` values.
- Submit alongside `item_group_id` and `variant_option`.
- Professional, grammatically correct. No all-caps for emphasis, no symbols, no HTML.
- No foreign-language gimmicks; majority must match data-source language.
- No promotional text (price, sale, dates, shipping, company name).
- No extra whitespace.

**Best practices:**
- Shorter and more generic than variant titles — exclude variant-identifying details (size, color).
- Most important details first (users see ~first 70 chars).
- Useful keywords: product name, brand, defining qualifiers ("maternity", "waterproof").
- Only details that apply to the whole group, not individual variants.
- Add brand if it's a differentiating factor.

## This skill depends on item_group_id

`item_group_title` only applies to grouped variants. The skill requires an `item_group_id` column (or runs after the `productfeed-item-group-id` skill has produced one). Products without a group get no `item_group_title`.

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
TTL  = col(df, 'title', 'titel', 'product name')
BRND = col(df, 'brand', 'merk', 'manufacturer')
COLOR= col(df, 'color', 'kleur', 'colour')
SIZE = col(df, 'size', 'grootte', 'maat')
MAT  = col(df, 'material', 'materiaal')
PAT  = col(df, 'pattern', 'patroon')
IGT  = col(df, 'item_group_title', 'g:item_group_title')
```

If `IGID` is missing: report that `item_group_title` needs grouped variants. Suggest running the `item_group_id` skill first, then re-run this one.

### Step 3: Derive the common title per group

The robust approach: for each group, take the variant titles and remove the per-variant tokens (the values of color/size/material/pattern that differ within the group). What remains, shared across all variants, is the group title.

```python
import re
from collections import Counter

def variant_tokens(group_df, cols):
    """Collect the distinct variant-identifying values present in this group."""
    tokens = set()
    for c in cols:
        if c and c in group_df.columns:
            for v in group_df[c].dropna().astype(str):
                v = v.strip()
                if v and v.lower() not in ('nan', 'none', ''):
                    tokens.add(v.lower())
    return tokens

def strip_tokens(title, tokens):
    t = str(title)
    for tok in sorted(tokens, key=len, reverse=True):
        t = re.sub(r'(?i)\b' + re.escape(tok) + r'\b', ' ', t)
    # tidy separators left behind (-, |, /, commas, doubled spaces)
    t = re.sub(r'\s*[-|/,]\s*', ' ', t)
    t = re.sub(r'\s{2,}', ' ', t).strip(' -|/,')
    return t.strip()

def common_prefix_title(titles):
    """Fallback: longest common word-prefix across variant titles."""
    split = [t.split() for t in titles if t]
    if not split:
        return ''
    out = []
    for words in zip(*split):
        if len(set(w.lower() for w in words)) == 1:
            out.append(words[0])
        else:
            break
    return ' '.join(out).strip(' -|/,')

def derive_group_title(group_df):
    titles = [str(t).strip() for t in group_df[TTL].dropna() if str(t).strip()]
    if not titles:
        return '', 'none'
    tokens = variant_tokens(group_df, [COLOR, SIZE, MAT, PAT])
    candidates = [strip_tokens(t, tokens) for t in titles]
    candidates = [c for c in candidates if c]
    # Most common stripped form wins; fall back to common prefix
    if candidates:
        winner = Counter(candidates).most_common(1)[0][0]
    else:
        winner = ''
    if not winner or len(winner) < 3:
        winner = common_prefix_title(titles)
    # Ensure brand is present if it differentiates and isn't already there
    if BRND:
        brands = [str(b).strip() for b in group_df[BRND].dropna() if str(b).strip()]
        if brands:
            brand = brands[0]
            if brand.lower() not in winner.lower():
                winner = f"{brand} {winner}".strip()
    winner = re.sub(r'\s{2,}', ' ', winner).strip()
    if len(winner) > 150:
        winner = winner[:150].rstrip()
    return winner, ('high' if candidates else 'medium')
```

### Step 4: Validate against Google rules

```python
def valid_group_title(group_title, variant_titles):
    if not group_title or not (1 <= len(group_title) <= 150):
        return False
    # Must differ from every variant title
    if any(group_title.strip().lower() == vt.strip().lower() for vt in variant_titles):
        return False
    # No all-caps emphasis (allow short abbreviations)
    words = group_title.split()
    if words and sum(1 for w in words if w.isupper() and len(w) > 3) > len(words) / 2:
        return False
    # No HTML
    if re.search(r'<[^>]+>', group_title):
        return False
    return True
```

### Step 5: Apply per group and build supplemental feed

```python
results = {'generated': 0, 'already_had': 0, 'singletons': 0, 'unresolved': 0}
group_title_map = {}   # item_group_id -> title

for igid, gdf in df.groupby(IGID):
    if not str(igid).strip() or str(igid).lower() in ('nan', 'none'):
        continue
    if len(gdf) < 2:
        results['singletons'] += 1     # a lone product isn't a variant group
        continue
    # respect existing values: if all rows already share a valid item_group_title, keep it
    if IGT:
        existing = [str(v).strip() for v in gdf[IGT].dropna() if str(v).strip()]
        if existing and len(set(existing)) == 1:
            group_title_map[igid] = existing[0]
            results['already_had'] += 1
            continue
    gt, conf = derive_group_title(gdf)
    vts = [str(t) for t in gdf[TTL].dropna()]
    if valid_group_title(gt, vts):
        group_title_map[igid] = gt
        results['generated'] += 1
    else:
        results['unresolved'] += 1

# Assign the group title to every row in the group (same value for all variants)
out_values = []
for idx, row in df.iterrows():
    igid = str(row[IGID]).strip() if pd.notna(row[IGID]) else ''
    out_values.append(group_title_map.get(igid, ''))

supplemental = pd.DataFrame({'id': df[ID], 'item_group_title': out_values})
output_path = "/mnt/user-data/outputs/supplemental_feed_item_group_title.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 6: Report

- Number of variant groups found and how many got a title.
- 10–15 examples: group id → derived group title + its variant titles, so the user can confirm the group title is correctly more generic.
- Singletons skipped (not a group) and any unresolved groups.

## Important guardrails

- **Same value for every variant** in a group — assign per group, write to all rows.
- **Must differ from variant titles** — a group title equal to a variant title is invalid; skip it.
- **More generic than variants** — strip color/size/material/pattern tokens; never add them.
- Max 150 chars (truncate, don't error).
- No promotional text, no all-caps emphasis, no HTML, no extra whitespace.
- Only for groups of 2+ variants; singletons get nothing.
- Never overwrite an existing consistent `item_group_title`.
- Language matches the feed (it's derived from existing titles, so this holds naturally).

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `item_group_title` (same value repeated across a group's variants)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell for ungrouped products)
