---
name: productfeed-document-link
description: Map and validate document_link attributes for Google Shopping product feeds (PDF documentation links used in Google's conversational AI / AI Mode shopping experience). Detects an existing PDF-URL source column, validates each URL against Google's spec, and reports honestly when no source data exists — it never invents URLs. Triggers when user uploads a product feed and wants to populate [document_link] values. Also triggers when user mentions "document link", "document_link", "PDF manual feed", "handleiding link", "manual URL", "package insert", "assembly instructions feed", or "conversational shopping attributes". Use this skill even if the user just says "fill in the document links" with an uploaded file.
---

# Feed Document Link Mapper & Validator

Map and validate `document_link` values — links to authoritative product PDFs (manuals, user guides, assembly instructions, package inserts) that Google uses in conversational AI experiences to answer detailed product questions.

## Why this matters

The `[document_link]` attribute feeds Google's conversational AI with authoritative source documents. When present, Google extracts FAQ-style information from these PDFs (and will *skip* `question_and_answer` content covering the same ground). It is optional but valuable for products where the real answers live in a manual or spec sheet.

## This is a map-and-validate skill, not a generator

A PDF URL cannot be derived from product text. It must already exist somewhere in the merchant's data. This skill therefore:
1. **Detects** whether the feed contains a column holding document/PDF URLs.
2. **Validates** each URL against Google's spec.
3. **Reports honestly** when no source column exists — it never fabricates a URL.

If there is no source column, the correct output is a clear message that this attribute requires merchant-supplied PDF URLs, plus guidance on how to add them. Do not guess.

## Google's specifications and rules

| Property | Value |
|---|---|
| **Type** | Document URL, must start with `http://` or `https://`, ASCII only, RFC 3986 compliant |
| **Limit** | Up to 2,000 characters per URL |
| **Supported file format** | `.PDF` only |
| **Repeated field** | Yes, up to 5 |
| **Max file size** | 50 MB per document |

**Minimum requirements:**
- URL starts with `http://` or `https://`.
- Special characters URL-encoded (e.g. comma → `%2C`).
- URL points to a PDF file.
- Publicly accessible — Googlebot must reach it without login and without robots.txt blocks. Stable, crawlable.
- Static, non-expiring links.

**Text feed format:** one URL, or multiple URLs separated by a comma (`,`):
`https://example.com/manual.pdf, https://example.com/assembly_instructions.pdf`

**Best practices:**
- Document is about the product (or the product plus related ones), not the company.
- Textual and graphical product info both fine.
- High-quality, authoritative documents with coherent structure.
- Provide all consumer-relevant docs: manuals, guides, assembly instructions, package inserts, packaging designs.

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

### Step 2: Detect a source column

```python
import re

SOURCE_CANDIDATES = [
    'document_link', 'g:document_link', 'document link', 'documentlink',
    'manual_url', 'manual', 'handleiding', 'handleiding_url', 'manual link',
    'datasheet', 'datasheet_url', 'spec_sheet', 'specsheet', 'instructions_url',
    'user_guide', 'userguide', 'pdf', 'pdf_url', 'pdf link', 'document_url',
    'package_insert', 'assembly_instructions', 'bijsluiter',
]

def find_source_column(df):
    existing_target = None
    candidates = []
    for c in df.columns:
        cl = c.lower().strip()
        if cl in ('document_link', 'g:document_link', 'document link'):
            existing_target = c
        if cl in SOURCE_CANDIDATES:
            candidates.append(c)
    # A column that actually contains .pdf URLs is a strong signal even if oddly named
    for c in df.columns:
        if c in candidates:
            continue
        sample = df[c].dropna().astype(str).head(50).str.lower()
        if (sample.str.contains(r'https?://').any() and sample.str.contains(r'\.pdf').any()):
            candidates.append(c)
    return existing_target, candidates

target_col, source_cols = find_source_column(df)
```

If `source_cols` is empty: **stop and report**. Tell the user the feed contains no PDF-URL column, so `document_link` cannot be populated — it requires merchant-supplied PDF links. Offer to proceed once a column (e.g. `manual_url`) is added.

### Step 3: Validate URLs

```python
def validate_doc_url(raw):
    """Returns (valid_url_or_empty, status). Splits comma-separated lists, validates each."""
    raw = str(raw).strip()
    if not raw or raw.lower() in ('nan', 'none'):
        return '', 'empty'

    urls = [u.strip() for u in raw.split(',') if u.strip()]
    valid = []
    issues = []
    for u in urls[:5]:  # max 5
        if not re.match(r'^https?://', u, re.I):
            issues.append(f"not http(s): {u[:60]}")
            continue
        if len(u) > 2000:
            issues.append(f"too long (>2000): {u[:60]}")
            continue
        # must look like a PDF (path ends in .pdf, ignoring query string)
        path = u.split('?', 1)[0].split('#', 1)[0]
        if not path.lower().endswith('.pdf'):
            issues.append(f"not a .pdf URL: {u[:60]}")
            continue
        if not u.isascii():
            issues.append(f"non-ASCII chars (URL-encode them): {u[:60]}")
            continue
        valid.append(u)

    if not valid:
        return '', 'invalid'
    # Google's text-feed format: comma + space separated
    return ', '.join(valid), 'valid' if not issues else 'partial'
```

Note: this validates *format*, not live reachability/file size. Optionally, if the environment allows network access and the feed is small, do a lightweight HEAD request to confirm the URL resolves and the content-type is `application/pdf`. Keep this opt-in and rate-limited; never block the whole run on it.

### Step 4: Build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id', 'g:id']), None)

results = {'valid': 0, 'invalid': 0, 'empty': 0, 'partial': 0}
flags = []
out_values = []

src = source_cols[0]  # if multiple, prefer the most pdf-dense; report the choice
for idx, row in df.iterrows():
    cleaned, status = validate_doc_url(row[src])
    out_values.append(cleaned)
    results[status] = results.get(status, 0) + 1
    if status in ('invalid', 'partial'):
        flags.append((row[id_col], status))

supplemental = pd.DataFrame({'id': df[id_col], 'document_link': out_values})
output_path = "/mnt/user-data/outputs/supplemental_feed_document_link.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

- Source column used (and why, if auto-detected by content).
- Counts: valid / partial / invalid / empty.
- 10–15 examples, including any invalid ones with the reason (not a PDF, not http, non-ASCII, too long).
- Clear note if NO source column was found: attribute cannot be filled without merchant PDF URLs.

## Important guardrails

- **Never invent or guess a URL.** No constructing `domain.com/manual.pdf` from a product link.
- **Never overwrite** an existing valid `document_link`.
- Drop URLs that aren't PDFs, aren't http(s), exceed 2,000 chars, or contain non-ASCII (advise URL-encoding).
- Max 5 URLs per product.
- Format reachability/file-size limits (50 MB, public, crawlable) can't be verified offline — surface them as merchant responsibilities in the report.
- If no source data exists, the honest output is "not possible without merchant data", not an empty guess dressed up as success.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `document_link` (comma-space separated URLs when multiple)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell where no valid URL exists)
