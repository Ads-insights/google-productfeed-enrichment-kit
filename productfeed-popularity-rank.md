---
name: productfeed-popularity-rank
description: Map, validate, or compute popularity_rank attributes for Google Shopping product feeds (the 0-100 popularity percentile used in Google's conversational AI / AI Mode shopping experience). Validates an existing rank column, or computes a percentile rank from a sales/quantity column — and reports honestly when no performance data exists, since it never invents popularity. Triggers when user uploads a product feed and wants to populate [popularity_rank] values. Also triggers when user mentions "popularity rank", "popularity_rank", "bestseller feed", "best sellers", "populariteit", "verkooprang", "sales rank attribute", or "conversational shopping attributes". Use this skill even if the user just says "fill in the popularity rank" with an uploaded file.
---

# Feed Popularity Rank Mapper & Calculator

Map, validate, or compute `popularity_rank` — a number from 0.0 to 100.0 indicating how well a product sells relative to the rest of your inventory. Higher = better performing. Used by Google's conversational AI to surface best sellers.

## Why this matters

`[popularity_rank]` lets you tell Google's conversational AI which products are your best sellers, helping it recommend them and helping shoppers make informed decisions. It's optional but a direct lever on AI-mode visibility for your strongest products.

## This is a map/validate/compute skill, not a generator

Popularity reflects real sales performance — it can't be inferred from product text. This skill operates in one of three modes, in priority order:
1. **Validate** — feed already has a `popularity_rank` column → validate and normalize it.
2. **Compute** — feed has a sales/quantity/revenue/rank-like column → compute a percentile rank (0–100) from it.
3. **Report not possible** — no performance signal anywhere → state clearly that this attribute requires merchant sales data. Never fabricate values.

## Google's specifications and rules

| Property | Value |
|---|---|
| **Type** | Number between 0.0 and 100.0 |
| **Decimals** | At most 1 decimal place |
| **Repeated field** | No |

**Minimum requirements:**
- A number between 0 and 100, max 1 decimal.
- **No `%` sign.**
- Accurate ranking, correctly reflecting relative sales vs the rest of your inventory.

**Best practices:**
- Update the value (e.g. via a supplemental data source) when popularity changes meaningfully based on recent sales.

## Workflow

### Step 1: Load the feed

```python
import pandas as pd
import numpy as np

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

### Step 2: Detect the mode

```python
import re

def col(df, *names):
    for n in names:
        if n in df.columns:
            return n
    return None

ID = col(df, 'id', 'unique merchant sku', 'merchant item id', 'g:id')

EXISTING_RANK = col(df, 'popularity_rank', 'g:popularity_rank', 'populariteit')

# Performance signals to compute from, in priority order.
# "higher is better" signals:
SALES_COLS = ['units_sold', 'units sold', 'quantity_sold', 'qty_sold', 'sales',
              'sales_count', 'orders', 'order_count', 'verkocht', 'aantal_verkocht',
              'revenue', 'omzet', 'sales_volume', 'purchases', 'conversions']
# "lower is better" signals (a literal sales rank, 1 = best):
RANK_COLS  = ['sales_rank', 'bestseller_rank', 'best_seller_rank', 'rank', 'ranking',
              'verkooprang', 'positie']

sales_col = next((col(df, c) for c in SALES_COLS if col(df, c)), None)
rank_col  = next((col(df, c) for c in RANK_COLS if col(df, c)), None)

def to_num(series):
    return pd.to_numeric(
        series.astype(str).str.replace('%', '', regex=False)
              .str.replace(r'[^0-9.,\-]', '', regex=True)
              .str.replace(',', '.', regex=False),
        errors='coerce'
    )
```

Decide mode:
- If `EXISTING_RANK` is present and mostly populated → **validate**.
- Else if `sales_col` or `rank_col` is present → **compute**.
- Else → **report not possible** and stop (no output file, or an all-empty file clearly labelled).

### Step 3a: Validate mode

```python
def validate_rank(series):
    nums = to_num(series)
    out, flags = [], {'valid': 0, 'clamped': 0, 'rounded': 0, 'invalid': 0, 'empty': 0}
    for v in nums:
        if pd.isna(v):
            out.append(''); flags['empty'] += 1; continue
        clamped = min(100.0, max(0.0, v))
        if clamped != v:
            flags['clamped'] += 1
        r = round(clamped, 1)
        if r != clamped:
            flags['rounded'] += 1
        out.append(f"{r:.1f}")
        flags['valid'] += 1
    return out, flags
```

### Step 3b: Compute mode (percentile rank)

Map the chosen signal onto 0–100 by **percentile rank** across the whole inventory, so the distribution always spans the full range and "higher value = higher popularity_rank". This matches Google's definition ("ranked as a percentage of total inventory").

```python
def compute_percentile_rank(values, higher_is_better=True):
    """values: numeric Series. Returns list of '0.0'..'100.0' strings (1 decimal)."""
    s = values.copy()
    if not higher_is_better:
        s = -s                      # invert so that rank 1 (best) → highest percentile
    # percentile rank in [0,100]; NaNs stay empty
    pct = s.rank(pct=True, method='average') * 100.0
    out = []
    for v in pct:
        out.append('' if pd.isna(v) else f"{round(min(100.0, max(0.0, v)), 1):.1f}")
    return out

# choose signal: prefer explicit sales rank if present, else sales/revenue magnitude
if rank_col:
    ranks = compute_percentile_rank(to_num(df[rank_col]), higher_is_better=False)
    source = f"computed from '{rank_col}' (sales rank, 1=best)"
elif sales_col:
    ranks = compute_percentile_rank(to_num(df[sales_col]), higher_is_better=True)
    source = f"computed from '{sales_col}' (higher sales = higher rank)"
```

### Step 4: Build supplemental feed

```python
# 'ranks' (or validated 'out') is the popularity_rank column
supplemental = pd.DataFrame({'id': df[ID], 'popularity_rank': ranks})
output_path = "/mnt/user-data/outputs/supplemental_feed_popularity_rank.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

- Mode used: validate / compute / not-possible.
- If compute: which source column, and whether higher or lower was treated as better.
- Distribution sanity: min, median, max of the output; confirm the top sellers got high ranks (show top 10 by rank with their titles).
- If validate: counts of clamped/rounded/invalid values.
- If not possible: a clear statement that `popularity_rank` requires sales/performance data not present in the feed, plus the suggestion to add a `units_sold` or `sales_rank` column (e.g. exported from the shop backend or GA4) and re-run.

## Important guardrails

- **Never invent popularity.** No deriving rank from price, recency, position in file, or any non-performance proxy. If there's no sales signal, say so.
- **Percentile, not raw value.** Computed ranks are percentile positions across the inventory so they span 0–100 as Google expects.
- **Direction matters.** A literal "sales rank" column where 1 = best must be inverted; a "units sold" column is used as-is.
- Clamp to 0–100, round to 1 decimal, no `%` sign.
- Never overwrite an existing valid `popularity_rank` unless the user explicitly asks to recompute.
- This attribute should be refreshed periodically — note in the report that it's a snapshot of current sales.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `popularity_rank` (number 0.0–100.0, 1 decimal, no `%`)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell where no performance data exists for that product)
