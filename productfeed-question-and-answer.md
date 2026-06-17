---
name: productfeed-question-and-answer
description: Generate question_and_answer attributes for Google Shopping product feeds (the FAQ attribute used in Google's conversational AI / AI Mode shopping experience). Creates factual Q&A pairs per product from hard feed data only. Triggers when user uploads a product feed and wants to populate [question_and_answer] values. Also triggers when user mentions "question and answer", "Q&A attribute", "FAQ feed", "veelgestelde vragen", "vraag en antwoord", "AI mode attributes", or "conversational shopping attributes". Use this skill even if the user just says "generate Q&A" or "fill in the FAQs" with an uploaded file.
---

# Feed Question and Answer Generator

Generate `question_and_answer` values — factual FAQ pairs that Google uses in conversational AI experiences (AI Mode in Google Search) to answer detailed product questions and guide buying decisions.

## Why this matters

The `[question_and_answer]` attribute is one of Google's conversational-experience attributes. It is optional but high-impact for AI-mode visibility:
- Feeds the answers Google's conversational AI gives when a shopper asks detailed questions about a product
- Lets you control the narrative on specs, ingredients, package contents, and use-cases instead of leaving Google to infer
- Gives a competitive edge over listings without structured Q&A, similar to how product_highlight wins free-listing real estate

## Google's specifications and rules

**Type:** Group attribute with 2 sub-attributes.

| Sub-attribute | Required | Limit |
|---|---|---|
| `question` | Yes | String, up to 1000 characters |
| `answer` | Yes | String, up to 1000 characters |

- **Repeated field:** yes, up to 30 Q&A pairs per product.
- **Total size limit:** all `question_and_answer` content for one product combined is capped at 10,000 characters.
- **Language:** must match the feed's source language.

**Minimum requirements:**
- Submit both `question` AND `answer` for every pair.
- Only add information about the product. No time-related info (prices, dates, delivery) — that belongs in offer attributes.
- Don't list keywords or search terms.
- Give correct, informative answers. Consumers rely on accuracy.

**Best practices:**
- Check grammar, spelling, capitalization.
- Provide variety: specs, ingredients, package contents, intended occasions/sports/activities/themes.
- **Don't duplicate** data already in `title`, `description`, `product_detail`, or `product_highlight`.
- **Don't submit** if you also submit `document_link` and the same info is in those PDFs — Google extracts FAQ from the documents instead.

## Conservative generation principle

This skill generates Q&A **only from hard, factual feed data** — never invented or inferred from category alone. A Q&A pair is only created when the feed actually contains the answer (a populated attribute, a structured product_detail section, or an explicit factual fragment in the description). If the data isn't there, no pair is generated. Better to output 2 accurate pairs than 6 padded guesses.

## Output format and escaping (read carefully)

The output is a single supplemental column `question_and_answer` containing all pairs for a product in **one cell**, using **Google Sheets escaping** (the user uploads the .xlsx to Google Drive, where it converts to a Sheet).

Google Sheets format for this group attribute:
- A pair is `question:answer`.
- Multiple pairs are comma-separated: `question1:answer1,question2:answer2`.
- Inside any sub-attribute value, the reserved characters **colon `:`, comma `,`, and backslash `\`** must be escaped with a backslash.

```python
def escape_qa_value(text):
    """Escape a single question or answer value for Google Sheets group-attribute format.
    Order matters: backslash first, then colon and comma."""
    text = str(text).strip()
    text = text.replace('\\', '\\\\')   # backslash first
    text = text.replace(':', '\\:')
    text = text.replace(',', '\\,')
    return text

def build_qa_cell(pairs):
    """pairs = list of (question, answer). Returns one Sheets-ready cell value."""
    parts = []
    for q, a in pairs:
        parts.append(f"{escape_qa_value(q)}:{escape_qa_value(a)}")
    return ','.join(parts)
```

## Workflow

### Step 1: Load and inspect the feed

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

Report: total products, how many already have `question_and_answer`, which source columns are available.

### Step 2: Detect feed language

Use the shared detection cascade (identical to the orchestrator's `detect_output_language`). Generated questions and answers must be written in the feed's language.

```python
import re

def detect_feed_language(df, title_col):
    LANGUAGE_SIGNALS = {
        'nl': r'\b(?:de|het|een|van|voor|met|uit|bij|ook|niet|maar|deze|naar)\b',
        'en': r'\b(?:the|a|an|for|with|and|from|this|that|not|but|also|into)\b',
        'de': r'\b(?:der|die|das|ein|eine|und|für|mit|von|aus|auf|ist|den|dem|nicht|auch|oder)\b',
        'fr': r'\b(?:le|la|les|un|une|des|et|pour|avec|dans|sur|qui|est|pas|mais|sont|cette)\b',
        'it': r'\b(?:il|la|le|un|una|gli|dei|per|con|che|non|sono|nel|dal|alla|questo|questa)\b',
        'es': r'\b(?:el|la|los|las|un|una|y|para|con|del|por|que|no|se|este|esta|más)\b',
        'pt': r'\b(?:o|a|os|as|um|uma|e|para|com|que|não|se|do|da|no|na|por|mais)\b',
    }
    for col_name in ['feed label', 'language', 'content language', 'target country']:
        matches = [c for c in df.columns if c.lower().strip() == col_name]
        if matches:
            val = df[matches[0]].dropna().head(1).tolist()
            if val:
                label = str(val[0]).strip().lower()
                country_to_lang = {
                    'nl': 'nl', 'be': 'nl', 'de': 'de', 'at': 'de', 'ch': 'de',
                    'fr': 'fr', 'it': 'it', 'es': 'es', 'pt': 'pt',
                    'gb': 'en', 'uk': 'en', 'us': 'en', 'au': 'en', 'ie': 'en',
                    'en': 'en', 'english': 'en', 'dutch': 'nl', 'german': 'de',
                    'deutsch': 'de', 'french': 'fr', 'français': 'fr',
                    'italian': 'it', 'italiano': 'it', 'spanish': 'es',
                    'español': 'es', 'portuguese': 'pt', 'português': 'pt',
                    'nederlands': 'nl',
                }
                if label in country_to_lang:
                    return country_to_lang[label]
    sample = ' '.join(df[title_col].dropna().head(100).tolist()).lower()
    scores = {lang: len(re.findall(p, sample)) for lang, p in LANGUAGE_SIGNALS.items()}
    scores = {k: v for k, v in scores.items() if v > 0}
    return max(scores, key=scores.get) if scores else 'en'
```

### Step 3: Gather hard facts per product

Only collect attributes that are actually populated. These are the *only* sources allowed to become Q&A pairs.

```python
COL_MAP = {
    'title': ['title', 'titel', 'product name'],
    'description': ['description', 'beschrijving', 'product description'],
    'brand': ['brand', 'merk', 'manufacturer'],
    'color': ['color', 'kleur', 'colour'],
    'material': ['material', 'materiaal'],
    'size': ['size', 'grootte', 'maat'],
    'pattern': ['pattern', 'patroon'],
    'product_type': ['product_type', 'producttype'],
    'condition': ['condition', 'staat'],
    'product_weight': ['product_weight', 'verzendgewicht', 'product weight', 'shipping weight', 'weight'],
    'product_length': ['product_length'],
    'product_width': ['product_width'],
    'product_height': ['product_height'],
    'energy_class': ['energy_efficiency_class', 'energielabel'],
    'gender': ['gender', 'geslacht'],
    'age_group': ['age_group', 'leeftijdsgroep'],
    'multipack': ['multipack'],
    'gtin': ['gtin', 'ean', 'upc'],
    'product_detail': ['product_detail', 'product detail'],
}

def gather_facts(row, col_map):
    facts = {}
    for key, candidates in col_map.items():
        for col in candidates:
            if col in row.index:
                val = str(row[col]).strip() if pd.notna(row[col]) else ''
                if val and val.lower() not in ['', 'nan', 'none']:
                    facts[key] = val
                    break
    return facts
```

### Step 4: Q&A templates per language

Each template maps a populated fact to one factual Q&A pair. Templates exist for all 7 supported languages; the NL set is shown in full, the others follow the same keys.

```python
QA_TEMPLATES = {
    'nl': {
        'material':   ("Waar is dit product van gemaakt?", "Dit product is gemaakt van {material}."),
        'color':      ("Welke kleur heeft dit product?", "De kleur is {color}."),
        'size':       ("Welke maat of welk formaat heeft dit product?", "Het formaat is {size}."),
        'dimensions': ("Wat zijn de afmetingen?", "De afmetingen zijn {dimensions}."),
        'product_weight': ("Hoeveel weegt dit product?", "Het gewicht is {product_weight}."),
        'brand':      ("Van welk merk is dit product?", "Dit product is van het merk {brand}."),
        'energy_class': ("Welk energielabel heeft dit product?", "Dit product heeft energielabel {energy_class}."),
        'condition':  ("In welke staat wordt dit product geleverd?", "Dit product wordt geleverd als {condition_nl}."),
        'multipack':  ("Hoeveel stuks zitten er in de verpakking?", "De verpakking bevat {multipack} stuks."),
        'gender':     ("Voor wie is dit product bedoeld?", "Dit product is bedoeld voor {gender_nl}."),
    },
    'en': {
        'material':   ("What is this product made of?", "This product is made of {material}."),
        'color':      ("What color is this product?", "The color is {color}."),
        'size':       ("What size is this product?", "The size is {size}."),
        'dimensions': ("What are the dimensions?", "The dimensions are {dimensions}."),
        'product_weight': ("How much does this product weigh?", "The weight is {product_weight}."),
        'brand':      ("What brand is this product?", "This product is made by {brand}."),
        'energy_class': ("What is the energy efficiency class?", "This product has energy class {energy_class}."),
        'condition':  ("What condition is this product in?", "This product is {condition_en}."),
        'multipack':  ("How many items are included?", "The pack contains {multipack} items."),
        'gender':     ("Who is this product intended for?", "This product is intended for {gender_en}."),
    },
    # de / fr / it / es / pt follow the same keys — translate the strings, keep the {placeholders}.
}

CONDITION_LABELS = {
    'nl': {'new': 'nieuw', 'used': 'tweedehands', 'refurbished': 'gereviseerd'},
    'en': {'new': 'new', 'used': 'used', 'refurbished': 'refurbished'},
}
GENDER_LABELS = {
    'nl': {'male': 'mannen', 'female': 'vrouwen', 'unisex': 'iedereen'},
    'en': {'male': 'men', 'female': 'women', 'unisex': 'everyone'},
}
```

### Step 5: Generate pairs (conservative)

```python
def generate_qa(facts, lang='nl'):
    """Build factual Q&A pairs from populated facts only.
    Returns (pairs, confidence). Each pair is (question, answer)."""
    t = QA_TEMPLATES.get(lang, QA_TEMPLATES['en'])
    pairs = []

    # Dimensions: only if at least L+W+H present
    dims = [facts.get('product_length'), facts.get('product_width'), facts.get('product_height')]
    if all(dims):
        facts['dimensions'] = " x ".join(dims)

    # Build a pair for each populated fact that has a template
    for key in ['material', 'color', 'size', 'dimensions', 'product_weight', 'brand',
                'energy_class', 'condition', 'multipack', 'gender']:
        if key not in facts and key != 'dimensions':
            continue
        if key == 'dimensions' and 'dimensions' not in facts:
            continue
        q, a_template = t[key]

        # special label substitutions
        if key == 'condition':
            cond = facts['condition'].lower().strip()
            label = CONDITION_LABELS.get(lang, CONDITION_LABELS['en']).get(cond)
            if not label:
                continue
            a = a_template.format(**{f'condition_{lang}': label}) if f'condition_{lang}' in a_template else a_template.replace('{condition_nl}', label).replace('{condition_en}', label)
        elif key == 'gender':
            g = facts['gender'].lower().strip()
            label = GENDER_LABELS.get(lang, GENDER_LABELS['en']).get(g)
            if not label:
                continue
            a = a_template.replace('{gender_nl}', label).replace('{gender_en}', label)
        else:
            a = a_template.format(**{key: facts[key]})

        if len(q) <= 1000 and len(a) <= 1000:
            pairs.append((q, a))

    # Cap at 30 pairs (Google limit) and 10,000 total chars
    capped = []
    total = 0
    for q, a in pairs[:30]:
        pair_len = len(q) + len(a)
        if total + pair_len > 10000:
            break
        capped.append((q, a))
        total += pair_len

    confidence = 'high' if len(capped) >= 2 else ('low' if capped else 'unresolved')
    return capped, confidence
```

### Step 6: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id', 'g:id']), None)
qa_col = next((c for c in df.columns if c in ['question_and_answer', 'g:question_and_answer']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), df.columns[0])

lang = detect_feed_language(df, title_col)
results = {'generated': 0, 'unresolved': 0, 'already_had': 0}
qa_values = []

for idx, row in df.iterrows():
    if qa_col:
        current = str(row[qa_col]).strip() if pd.notna(row[qa_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            qa_values.append(current)
            continue

    facts = gather_facts(row, COL_MAP)
    pairs, conf = generate_qa(facts, lang)

    if len(pairs) >= 2:          # Google needs at least meaningful content; we require >=2 to be useful
        qa_values.append(build_qa_cell(pairs))
        results['generated'] += 1
    else:
        qa_values.append('')
        results['unresolved'] += 1

supplemental = pd.DataFrame({'id': df[id_col], 'question_and_answer': qa_values})
output_path = "/mnt/user-data/outputs/supplemental_feed_question_and_answer.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 7: Report

- Fill rate before/after
- Average pairs per product
- 10–15 spot-check examples showing the generated pairs (decode the escaping for readability)
- Products left empty (insufficient hard data) — this is expected and correct, not a failure

## Important guardrails

- **Never overwrite existing `question_and_answer` values.**
- **Only generate from hard feed data.** No category-based inference, no marketing claims, no invented specs.
- **Never duplicate** what's already in title/description/product_detail/product_highlight verbatim — phrase as a genuine question and answer instead.
- **No offer data** — never put price, sale, shipping, stock, or dates in a Q&A.
- **No keyword stuffing** in questions or answers.
- Each sub-value max 1000 chars; max 30 pairs; 10,000 chars total per product.
- Require at least 2 solid pairs, otherwise leave empty (better empty than thin).
- Language must match the feed.
- Escape `:`, `,`, and `\` in every sub-value (Google Sheets format).

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original.

- **Columns:** `id` + `question_and_answer` (single cell per product, all pairs combined, Sheets-escaped)
- Column name is the **English Google Merchant Center attribute name**
- Format: **Excel (.xlsx)** — user uploads to Google Drive, which converts to a Sheet
- Include ALL products (empty cell where no Q&A could be generated)
