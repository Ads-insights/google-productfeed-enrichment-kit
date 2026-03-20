---
name: productfeed-orchestrator
description: Orchestrate all productfeed enrichment skills on a single feed. Analyzes the feed, detects output language, determines which attributes need enrichment, runs all relevant skills in the correct order, and outputs a single combined supplemental feed. Triggers when user uploads a product feed and asks to "enrich everything", "run all skills", "complete feed enrichment", "volledige feed verrijking", or wants a comprehensive supplemental feed. Use this skill even if the user just says "enrich this feed" or "optimize my feed" with an uploaded file.
---

# Productfeed Orchestrator

The orchestrator is a **traffic controller** — it doesn't help Claude understand the feed (Claude already understands any language), it tells Claude **what to do** with the information: which output language, which column names, which execution order, which guardrails.

## Role of this skill

- **NOT a translator.** Claude reads Dutch, English, Romanian, German — any language. No vocabulary lists or word-level translation needed for input processing.
- **A routing controller.** It determines: output language, execution order, dependency chain, column mapping, and quality gates.
- The individual 25 skills contain the extraction/generation logic. This orchestrator coordinates them.

## Step 1: Load the feed

```python
import pandas as pd
import re

file_path = "/mnt/user-data/uploads/<filename>"

# Auto-detect format
if file_path.endswith('.zip'):
    import zipfile
    with zipfile.ZipFile(file_path) as z:
        inner = z.namelist()[0]
        if inner.endswith('.tsv'):
            df = pd.read_csv(z.open(inner), sep='\t', dtype=str)
        elif inner.endswith('.csv'):
            df = pd.read_csv(z.open(inner), dtype=str)
        else:
            df = pd.read_excel(z.open(inner), dtype=str)
elif file_path.endswith('.gz'):
    import gzip
    with gzip.open(file_path, 'rt') as f:
        first_line = f.readline()
    sep = '\t' if '\t' in first_line else ','
    df = pd.read_csv(file_path, sep=sep, dtype=str, compression='gzip')
elif file_path.endswith('.csv'):
    df = pd.read_csv(file_path, dtype=str)
elif file_path.endswith('.tsv'):
    df = pd.read_csv(file_path, sep='\t', dtype=str)
elif file_path.endswith('.xlsx') or file_path.endswith('.xls'):
    df = pd.read_excel(file_path, dtype=str)
else:
    with open(file_path, 'r') as f:
        first_line = f.readline()
    if '|' in first_line:
        df = pd.read_csv(file_path, sep='|', dtype=str)
    elif '\t' in first_line:
        df = pd.read_csv(file_path, sep='\t', dtype=str)
    else:
        df = pd.read_csv(file_path, dtype=str)

df.columns = [c.strip().lower() for c in df.columns]
total = len(df)
```

## Step 2: Detect output language

This does NOT help Claude read the feed — Claude understands any language natively. The purpose is to determine what language the **generated content** should be written in.

The rule: **generated content must match the feed's source language.** A Dutch feed gets Dutch descriptions. A German feed gets German descriptions. You never want a French webshop to suddenly get English titles in their supplemental feed.

Supported output languages: `nl`, `en`, `de`, `fr`, `it`, `es`, `pt`.

```python
LANGUAGE_SIGNALS = {
    'nl': r'\b(?:de|het|een|van|voor|met|uit|bij|ook|niet|maar|deze|naar)\b',
    'en': r'\b(?:the|a|an|for|with|and|from|this|that|not|but|also|into)\b',
    'de': r'\b(?:der|die|das|ein|eine|und|für|mit|von|aus|auf|ist|den|dem|nicht|auch|oder)\b',
    'fr': r'\b(?:le|la|les|un|une|des|et|pour|avec|dans|sur|qui|est|pas|mais|sont|cette)\b',
    'it': r'\b(?:il|la|le|un|una|gli|dei|per|con|che|non|sono|nel|dal|alla|questo|questa)\b',
    'es': r'\b(?:el|la|los|las|un|una|y|para|con|del|por|que|no|se|este|esta|más)\b',
    'pt': r'\b(?:o|a|os|as|um|uma|e|para|com|que|não|se|do|da|no|na|por|mais)\b',
}

def detect_output_language(df, title_col, col_map=None):
    """Determine the language for generated content by sampling titles.
    Claude understands any input language — this only controls OUTPUT language.
    
    Detection cascade:
    1. Check 'feed label' or 'language' column in the feed (explicit merchant signal)
    2. Count language-specific function words in titles (statistical signal)
    3. Fallback to 'en' if truly ambiguous
    """
    
    # --- Signal 1: Explicit feed label or language column ---
    # Many feeds (especially Google Merchant Center exports) have a 'feed label' or 'language' column
    # that directly states the target language/country.
    if col_map:
        for col_name in ['feed label', 'language', 'content language', 'target country']:
            actual_col = col_map.get(col_name) or next(
                (c for c in df.columns if c.lower().strip() == col_name), None)
            if actual_col and actual_col in df.columns:
                val = df[actual_col].dropna().head(1).tolist()
                if val:
                    label = str(val[0]).strip().lower()
                    # Map country codes to language codes
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
    
    # --- Signal 2: Statistical language detection from titles ---
    sample = ' '.join(df[title_col].dropna().head(100).tolist()).lower()
    
    scores = {}
    for lang, pattern in LANGUAGE_SIGNALS.items():
        scores[lang] = len(re.findall(pattern, sample))
    
    # Filter out languages with 0 score
    scores = {k: v for k, v in scores.items() if v > 0}
    
    if not scores:
        return 'en'  # Fallback when no signals found at all
    
    # Winner must have at least 1.3x the score of the runner-up to be confident
    sorted_scores = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    winner_lang, winner_score = sorted_scores[0]
    
    if len(sorted_scores) > 1:
        runner_up_score = sorted_scores[1][1]
        if winner_score < runner_up_score * 1.3:
            # Too close to call — check if feed label column can break the tie
            # If not, default to the winner anyway (best guess)
            pass
    
    return winner_lang

output_lang = detect_output_language(df, title_col, col_map)
```

### Important: `detect_output_language` is the shared language detection function

All skills that need language detection should use this same logic. When a skill is run standalone (outside the orchestrator), it should implement the same detection cascade. The orchestrator passes `output_lang` to dependent skills; standalone skills detect it themselves.

## Where language matters (and where it doesn't)

There are three categories of output attributes. Language ONLY matters for the third:

### Category 1: Technical attributes — always English, always fixed values
These are codes/enums that Google Merchant Center expects in English regardless of feed language. Claude outputs these directly, no language logic needed.

| Attribute | Possible values |
|---|---|
| `condition` | `new`, `used`, `refurbished` |
| `gender` | `male`, `female`, `unisex` |
| `age_group` | `newborn`, `infant`, `toddler`, `kids`, `adult` |
| `adult` | `true`, `false` |
| `is_bundle` | `true`, `false` |
| `identifier_exists` | `true`, `false` |
| `size_system` | `EU`, `US`, `UK`, `AU` |
| `size_type` | `regular`, `plus`, `tall`, `petite`, `maternity` |
| `google_product_category` | numeric ID (e.g., `2831`) |

### Category 2: Extracted attributes — output must match the feed's language
These are extracted from the feed and **output in the feed's detected language**. If the title says "schwarz", color becomes "Schwarz". If it says "black", color becomes "Black". If it says "noir", color becomes "Noir". The skill's vocabulary maps input words from any supported language to the correct output word for the detected `output_lang`.

| Attribute | Example NL | Example EN | Example DE | Example FR |
|---|---|---|---|---|
| `color` | Zwart | Black | Schwarz | Noir |
| `material` | Katoen | Cotton | Baumwolle | Coton |
| `pattern` | Gestreept | Striped | Gestreift | Rayé |
| `brand` | (same in any language) | (same) | (same) | (same) |
| `size` | 500 ml | 500 ml | 500 ml | 500 ml |
| `product_type` | Kleding > Jassen | Clothing > Jackets | Kleidung > Jacken | Vêtements > Vestes |
| `unit_pricing_measure` | 100g | 100g | 100g | 100g |
| `item_group_id` | (derived from data) | (derived) | (derived) | (derived) |

**Pass `output_lang` to color and material skills.** The other extraction skills (pattern, brand, size, etc.) extract verbatim from the feed and don't need language routing.

### Category 3: Generated attributes — must match the detected output language
These are the ONLY attributes where `output_lang` matters. Claude generates new text and must write it in the same language as the feed.

| Attribute | What `output_lang` controls |
|---|---|
| `title` | Optimized/generated titles match source language |
| `description` | Generated descriptions use language-appropriate templates |
| `short_title` | Strip patterns use language-appropriate CTA words |
| `product_highlight` | Bullet text: "Gemaakt van katoen" vs "Made from cotton" vs "Hergestellt aus Baumwolle" |
| `product_detail` | Section labels: "Algemeen" vs "General" vs "Allgemein" (note: Google accepts English labels regardless) |

**Pass `output_lang` to these 5 generative skills AND to color and material extraction skills.** The other 18 skills ignore it.

## Step 3: Column mapping

Map the feed's actual column names to the standard names that all skills expect.

```python
COLUMN_CANDIDATES = {
    'id':               ['id', 'product id', 'unique merchant sku', 'merchant item id', 'sku', 'item_id', 'g:id'],
    'title':            ['title', 'titel', 'product name', 'product_name', 'name', 'g:title'],
    'description':      ['description', 'beschrijving', 'product description', 'product_description', 'g:description'],
    'brand':            ['brand', 'merk', 'manufacturer', 'mfr', 'vendor', 'g:brand'],
    'color':            ['color', 'kleur', 'colour', 'g:color'],
    'material':         ['material', 'materiaal', 'g:material'],
    'size':             ['size', 'grootte', 'maat', 'g:size'],
    'size_system':      ['size_system', 'matensysteem', 'g:size_system'],
    'size_type':        ['size_type', 'maattype', 'g:size_type'],
    'gender':           ['gender', 'geslacht', 'g:gender'],
    'age_group':        ['age_group', 'leeftijdsgroep', 'g:age_group'],
    'pattern':          ['pattern', 'patroon', 'g:pattern'],
    'condition':        ['condition', 'staat', 'g:condition'],
    'adult':            ['adult', 'inhoud voor volwassenen', 'g:adult'],
    'is_bundle':        ['is_bundle', 'is pakket', 'bundle', 'g:is_bundle'],
    'multipack':        ['multipack', 'g:multipack'],
    'identifier_exists':['identifier_exists', 'id bestaat', 'g:identifier_exists'],
    'google_product_category': ['google_product_category', 'google productcategorie', 'g:google_product_category'],
    'product_type':     ['product_type', 'producttype', 'product web category', 'product purchasing category'],
    'item_group_id':    ['item_group_id', 'groeps id', 'parent sku', 'parent_sku', 'g:item_group_id'],
    'product_highlight':['product_highlight', 'product highlight'],
    'product_detail':   ['product_detail', 'product detail'],
    'short_title':      ['short_title', 'korte titel'],
    'image_link':       ['image_link', 'afbeeldingslink', 'product image', 'image_url'],
    'link':             ['link', 'product url', 'url', 'product_url'],
    'price':            ['price', 'prijs', 'product value', 'current price'],
    'availability':     ['availability', 'beschikbaarheid', 'stock availability'],
    'gtin':             ['gtin', 'ean', 'upc', 'isbn', 'barcode'],
    'mpn':              ['mpn', 'manufacturer_part_number'],
    'labels':           ['labels', 'aangepast label 1', 'custom_label_0'],
    'weight':           ['product_weight', 'verzendgewicht', 'shipping weight', 'product weight', 'weight'],
    'bullet1':          ['product bullet point 1'],
    'bullet2':          ['product bullet point 2'],
    'bullet3':          ['product bullet point 3'],
    'unit_pricing_measure': ['unit_pricing_measure', 'eenheidsprijs hoeveelheid'],
    'unit_pricing_base_measure': ['unit_pricing_base_measure', 'eenheidsprijs basishoeveelheid'],
}

def map_columns(df):
    """Map feed columns to standard names. Returns dict {standard_name: actual_col_name}."""
    col_map = {}
    df_cols_lower = {c.lower(): c for c in df.columns}
    for standard_name, candidates in COLUMN_CANDIDATES.items():
        for candidate in candidates:
            if candidate.lower() in df_cols_lower:
                col_map[standard_name] = df_cols_lower[candidate.lower()]
                break
    return col_map

col_map = map_columns(df)
```

## Step 4: Feed analysis

Analyze the current state of the feed before running skills.

```python
def analyze_feed(df, col_map, total):
    """Report fill rates per attribute."""
    report = {}
    for attr, col in col_map.items():
        filled = df[col].notna() & (df[col].str.strip() != '') & (df[col].str.lower() != 'nan')
        report[attr] = {
            'column': col,
            'filled': filled.sum(),
            'empty': total - filled.sum(),
            'fill_pct': round(filled.sum() / total * 100, 1),
        }
    return report

feed_report = analyze_feed(df, col_map, total)
```

Present a summary to the user BEFORE running skills:
- Total products
- Detected output language and why
- Columns found with fill rates
- Columns missing entirely (= skills that will generate from scratch)
- Ask user to confirm before proceeding

## Step 5: Execution order

Skills run in 4 phases. Each phase depends on results from the previous phase.

```
Phase 1 — EXTRACTION (14 skills, independent)
  ┌─ brand          ← manufacturer/title first-word
  ├─ color          ← title/description/labels              [output_lang]
  ├─ material       ← title/description/labels              [output_lang]
  ├─ gender         ← title/product_type
  ├─ age_group      ← title/product_type
  ├─ pattern        ← title/description/labels
  ├─ condition      ← title/description (default: new)
  ├─ adult          ← title/category (default: false)
  ├─ is_bundle      ← title/description (default: false)
  ├─ multipack      ← title/description
  ├─ size           ← title/description (metric-first!)
  ├─ unit_pricing   ← title/description (metric-first!)
  ├─ energy_eff     ← title/description
  └─ dimensions     ← title/description

Phase 2 — DEPENDENT EXTRACTION (3 skills, need Phase 1)
  ├─ size_system    ← needs: size (only if size exists)
  ├─ size_type      ← needs: size (only if size exists)
  └─ identifier_exists ← needs: brand + gtin + mpn

Phase 3 — CLASSIFICATION (3 skills, improved by Phase 1)
  ├─ google_product_category ← title + product_type + brand
  ├─ product_type            ← title + category + url + brand
  └─ item_group_id           ← sku + title + color + size + material

Phase 4 — GENERATIVE (5 skills, need all previous + output_lang)
  ├─ title             ← title + brand/color/size           [output_lang]  ← FIRST: optimize/generate
  ├─ short_title       ← title (strips variants)           [output_lang]  ← THEN: shorten the (now better) title
  ├─ product_highlight ← all attributes + improved title    [output_lang]
  ├─ product_detail    ← all attributes                     [output_lang]
  └─ description       ← all attributes + improved title    [output_lang]  ← LAST: uses everything
```

## Step 6: Running the skills

**The orchestrator does NOT reimplement skill logic.** For each skill, read and follow the corresponding `productfeed-<attribute>/SKILL.md` instructions. The orchestrator's job:

1. **Route**: determine which skills to run based on feed analysis
2. **Order**: execute skills in the correct phase sequence
3. **Wire**: pass the right source columns from `col_map` to each skill
4. **Language**: pass `output_lang` to Phase 4 generative skills only
5. **Collect**: gather all output values for the combined supplemental feed

For each skill:

```python
# Phase 1: color and material need output_lang for correct output language
color_value = extract_color(title, description, product_type, labels, lang=output_lang)
material_value = extract_material(title, description, product_type, labels, lang=output_lang)

# Phase 1: other extraction skills — no language parameter needed
size_value = extract_size(title, description, product_type, labels)

# Phase 4: pass output_lang to generative skills
description_value = generate_description(info, lang=output_lang)
highlights_value = generate_highlights(context, lang=output_lang)
```

## Step 7: Build supplemental feed

```python
# All 35 output columns with Google Merchant Center attribute names
OUTPUT_COLUMNS = [
    # Phase 1: Extraction
    'color', 'material', 'brand', 'gender', 'age_group', 'pattern',
    'condition', 'adult', 'is_bundle', 'multipack', 'size',
    'unit_pricing_measure', 'unit_pricing_base_measure',
    'energy_efficiency_class', 'min_energy_efficiency_class', 'max_energy_efficiency_class',
    'product_weight', 'product_length', 'product_width', 'product_height',
    'shipping_weight', 'shipping_length', 'shipping_width', 'shipping_height',
    # Phase 2: Dependent
    'size_system', 'size_type', 'identifier_exists',
    # Phase 3: Classification
    'google_product_category', 'product_type', 'item_group_id',
    # Phase 4: Generative (title first, then short_title from improved title)
    'title', 'short_title', 'product_highlight', 'product_detail', 'description',
]

supplemental = pd.DataFrame({'id': df[col_map['id']]})
for col in OUTPUT_COLUMNS:
    supplemental[col] = enriched_values[col]

output_path = "/mnt/user-data/outputs/supplemental_feed_complete.xlsx"
supplemental.to_excel(output_path, index=False)
```

## Step 7.5: Post-enrichment validation

After all 25 skills have run and the supplemental feed is assembled, run these cross-attribute sanity checks. These are pure logical checks on the output — language-independent, vertical-independent. They catch contradictions between attributes that no individual skill can detect on its own.

**The orchestrator fixes what it can automatically, and flags the rest in the quality report.**

```python
validation_flags = []

for idx, row in supplemental.iterrows():
    product_id = row['id']

    # ── CHECK 1: size_system / size_type only when size is meaningful sizing ──
    # size_system and size_type are for clothing/shoe sizing only.
    # If size contains metric units (ml, g, kg, cm, l) or count units (capsule, stuks, bustine),
    # it's a product measurement, not a garment size → clear size_system and size_type.
    size_val = str(row.get('size', '')).lower().strip()
    if size_val and re.search(r'\d+\s*(ml|cl|l|g|gr|kg|mg|cm|mm|m|oz|lb|capsul|bustin|porzioni|porties|stuks|tabletten|druppels)\b', size_val):
        supplemental.at[idx, 'size_system'] = ''
        supplemental.at[idx, 'size_type'] = ''

    # ── CHECK 2: gender + age_group coherence ──
    # Products with age_group in (newborn, infant, toddler) should not have gender set.
    # Google does not require gender for baby products, and assigning it is usually a guess.
    age = str(row.get('age_group', '')).lower().strip()
    if age in ('newborn', 'infant', 'toddler'):
        if str(row.get('gender', '')).strip():
            supplemental.at[idx, 'gender'] = ''

    # ── CHECK 3: gender vs product_type / title coherence ──
    # If the product_type or title clearly indicates one gender, but gender says another, flag it.
    gender = str(row.get('gender', '')).lower().strip()
    product_type = str(row.get('product_type', '')).lower()
    title = str(row.get('title', '')).lower()
    context = f"{product_type} {title}"
    if gender == 'female' and any(w in context for w in ['heren', '/heren/', 'men ', 'uomo', 'herren']):
        validation_flags.append((product_id, 'gender_mismatch', f"gender=female but title/product_type suggests male"))
        supplemental.at[idx, 'gender'] = ''  # Clear rather than guess wrong
    if gender == 'male' and any(w in context for w in ['dames', '/dames/', 'women', 'donna', 'damen']):
        validation_flags.append((product_id, 'gender_mismatch', f"gender=male but title/product_type suggests female"))
        supplemental.at[idx, 'gender'] = ''

    # ── CHECK 3b: age_group vs title coherence ──
    # If the title clearly contains child-related words but age_group is 'adult', correct it.
    # This catches cases where the SOURCE FEED has wrong age_group values that the skill
    # preserved (because the skill respects existing values). The orchestrator overrides
    # because title evidence is stronger than a likely-incorrect feed value.
    # The same applies to baby/infant products.
    age = str(row.get('age_group', '')).lower().strip()
    title_lower = str(row.get('title', '')).lower()
    if age == 'adult' or not age:
        # Check for child product signals in title
        has_kinder = bool(re.search(r'\bkinder[-\s]|\bfür\s+kinder\b|\bkids\b|\bchildren\b', title_lower))
        has_baby = bool(re.search(r'\bbaby[-\s]|\bbébé\b|\bsäugling\b|\bneugeboren\b|\bnewborn\b', title_lower))
        if has_baby:
            supplemental.at[idx, 'age_group'] = 'infant'
            if age == 'adult':
                validation_flags.append((product_id, 'age_title_mismatch', f"age_group=adult but title contains baby signal → corrected to infant"))
        elif has_kinder:
            supplemental.at[idx, 'age_group'] = 'kids'
            if age == 'adult':
                validation_flags.append((product_id, 'age_title_mismatch', f"age_group=adult but title contains kinder signal → corrected to kids"))

    # ── CHECK 4: google_product_category exists and is numeric ──
    gpc = str(row.get('google_product_category', '')).strip()
    if gpc:
        if not gpc.isdigit():
            validation_flags.append((product_id, 'gpc_not_numeric', f"GPC value '{gpc}' is not a numeric ID"))
            supplemental.at[idx, 'google_product_category'] = ''

    # ── CHECK 5: age_group vs google_product_category coherence ──
    # If age_group is kids/infant/toddler/newborn, GPC should not be in adult-only categories.
    # Lingerie (GPC 212, 213, 1772), Adult content categories → mismatch with child age_groups.
    ADULT_ONLY_GPC = {'212', '213', '1772', '5713'}  # Bras, underwear, lingerie, nightwear
    if age in ('kids', 'infant', 'toddler', 'newborn') and gpc in ADULT_ONLY_GPC:
        validation_flags.append((product_id, 'gpc_age_mismatch', f"age_group={age} but GPC={gpc} is adult-only category"))
        supplemental.at[idx, 'google_product_category'] = ''  # Clear — better no GPC than wrong GPC

    # ── CHECK 6: is_bundle and multipack mutual exclusivity ──
    # A product should not be both a bundle AND a multipack. If both are set, prefer multipack
    # (more common, more specific) and clear is_bundle.
    is_bundle = str(row.get('is_bundle', '')).lower().strip()
    multipack = str(row.get('multipack', '')).strip()
    if is_bundle == 'true' and multipack:
        supplemental.at[idx, 'is_bundle'] = 'false'
        validation_flags.append((product_id, 'bundle_multipack_conflict', f"Both is_bundle=true and multipack={multipack} — kept multipack, cleared bundle"))

    # ── CHECK 7: identifier_exists coherence ──
    # If identifier_exists=true, at least one of (gtin, mpn, brand) must have a value in the source feed.
    # This check uses the ORIGINAL feed data, not enriched, because identifier_exists reflects
    # whether the merchant HAS identifiers, not whether we extracted them.
    ie = str(row.get('identifier_exists', '')).lower().strip()
    if ie == 'false':
        # Double-check: if brand is filled in source, identifier_exists should be true
        orig_brand = df.iloc[idx].get(col_map.get('brand', ''), '') if 'brand' in col_map else ''
        orig_gtin = df.iloc[idx].get(col_map.get('gtin', ''), '') if 'gtin' in col_map else ''
        if has_val(orig_brand) or has_val(orig_gtin):
            supplemental.at[idx, 'identifier_exists'] = 'true'

# Summary
fixes_applied = len(supplemental) - len(validation_flags)  # auto-fixed don't show as flags
print(f"\nPost-enrichment validation: {len(validation_flags)} flags raised")
for flag_type, count in Counter(f[1] for f in validation_flags).items():
    print(f"  {flag_type}: {count}")
```

### Validation rules reference

| # | Check | Action on fail | Why |
|---|---|---|---|
| 1 | size contains units (ml, g, capsule, etc.) → size_system/size_type should be empty | Auto-clear | size_system=EU on "500ml" makes no sense |
| 2 | age_group is baby/toddler → gender should be empty | Auto-clear | Google doesn't require gender for babies |
| 3 | gender contradicts title/product_type | Auto-clear + flag | Wrong gender = worse than no gender |
| 3b | age_group=adult but title contains "Kinder-"/"Baby-" | Auto-correct to kids/infant + flag | Source feed has wrong age_group; title evidence is stronger |
| 4 | GPC is non-numeric | Auto-clear + flag | Google requires numeric ID |
| 5 | Child age_group + adult-only GPC | Auto-clear + flag | Prevents Merchant Center disapproval |
| 6 | is_bundle=true AND multipack set | Keep multipack, clear bundle + flag | Mutually exclusive per Google spec |
| 7 | identifier_exists=false but brand/gtin present | Auto-fix to true | Prevents missed Shopping eligibility |

**Key principle: clearing a wrong value is always better than keeping it.** An empty cell in a supplemental feed means Google falls back to the primary feed or its own inference. A wrong value actively misleads Google and can cause disapprovals.

## Step 8: Quality report

Present AFTER running all skills:

```
══════════════════════════════════════════════════════
  FEED ENRICHMENT REPORT
══════════════════════════════════════════════════════

  Feed:      <filename>
  Products:  <total>
  Output language: <NL/EN> (detected from titles)

  ── FILL RATES ──────────────────────────────────────
                              BEFORE    AFTER    CHANGE
  brand                       50.3%   100.0%   +49.7%
  color                       12.5%    78.3%   +65.8%
  size                         0.0%    79.3%   +79.3%
  condition                    0.0%   100.0%  +100.0%
  ...

  ── SKILLS EXECUTED ─────────────────────────────────
  Phase 1 (Extraction):    14 skills
  Phase 2 (Dependent):      3 skills
  Phase 3 (Classification): 3 skills
  Phase 4 (Generative):     5 skills (output_lang = NL)

  ── SPOT CHECK (10 random samples) ─────────────────
  [show 10 random products with before/after comparison]

  ── QUALITY FLAGS ───────────────────────────────────
  ⚠️ 120 products: no color extracted (no color info in title)
  ⚠️ 30 titles optimized (score < 60 → review recommended)
  ⚠️ item_group_id via title normalization (medium confidence)

  ── VALIDATION (post-enrichment checks) ───────────
  ✅ 450 products: size_system/size_type cleared (metric size, not garment sizing)
  ✅ 12 products: gender cleared (baby/toddler age_group)
  ⚠️ 3 products: GPC cleared (child product in adult-only category)
  ⚠️ 1 product: bundle/multipack conflict resolved

  ── OUTPUT ──────────────────────────────────────────
  File: supplemental_feed_complete.xlsx
  Columns: id + 35 attributes
  Products: <total> (all included, empty cells where unresolved)
══════════════════════════════════════════════════════
```

## Key rules

1. **The orchestrator is a traffic controller, not a translator.** It routes data and controls output language — it does not help Claude understand input.
2. **Language detection supports 7 languages.** NL, EN, DE, FR, IT, ES, PT. It determines which output vocabulary/templates Phase 1 (color, material) and Phase 4 (generative) skills use. Other skills ignore it.
3. **Column mapping is flexible.** Handles NL, EN, DE, FR, g:-prefixed, and vendor-specific column names.
4. **Execution order is strict.** Phase 1 → 2 → 3 → 4. Dependencies flow downward.
5. **Never reimplement skill logic.** Read each skill's SKILL.md and follow it. The orchestrator coordinates, the skills execute.
6. **All products in output.** Empty cells where no enrichment was possible.
7. **English column names always.** Google Merchant Center attribute names regardless of feed language.
8. **Metric-first.** For size and unit_pricing, always prefer metric (g, ml, kg) over imperial (oz, lb).
9. **Conservative on generation.** Titles only optimized if score < 60. Descriptions only generated/expanded if < 150 chars or empty.
10. **Show before/after.** Always present spot-check examples so the user can verify quality.
11. **Validate after enrichment.** Always run post-enrichment cross-checks (Step 7.5) before outputting. Clearing a wrong value is always better than keeping it.

## Guardrails inherited from skills

The orchestrator inherits all guardrails from the 25 individual skills:

- **Never overwrite good data.** 23 of 25 skills skip products with existing values. Only `title` (score < 60) and `description` (< 150 chars) may modify existing values.
- **Metric over imperial.** Size and unit_pricing always prefer metric when both are present.
- **No false positives on multipack.** Dosage counts (60 capsules, 100 tablets) are NOT multipacks. Furniture terms (3-Sitzer, 4-teilig, 5-Zonen) are NOT multipacks. The `-teilig`/`-delig` pattern is excluded — it means "consisting of X parts" (modular), not "X identical items". Only scan titles, not descriptions.
- **No false positives on is_bundle.** The word "set" is excluded as a single-word trigger (too ambiguous: "Messer-Set", "gesetzliche Garantie", "festgesetzt"). Description scanning is disabled for bundles. Only unambiguous multi-word patterns or numbered patterns ("3er-Set") trigger a bundle flag.
- **No false positives on unit_pricing.** Length units (cm, mm, m) and area units (m², sqm) are EXCLUDED from unit pricing. These are product dimensions, not consumable measures. Only weight (g/kg/mg), volume (ml/cl/l), and count (stuks/pieces) are valid unit pricing units.
- **Generated content matches feed language.** A Dutch feed never gets English generated text. A German feed never gets French generated text. This applies to 7 supported languages: NL, EN, DE, FR, IT, ES, PT.
- **Extracted color and material match feed language.** Color and material output use the detected `output_lang` to select the correct vocabulary. A German feed outputs "Schwarz", not "Black" or "Zwart".
- **Technical values always English.** `condition`, `gender`, `age_group`, `adult`, `is_bundle`, `identifier_exists`, `size_system`, `size_type` — always English enum values regardless of feed language.
- **Post-enrichment validation catches cross-attribute errors.** No individual skill can detect that a kids product got an adult-only GPC, or that a supplement got size_system=EU. The orchestrator's validation step (Step 7.5) runs 7 cross-checks after all skills complete and auto-fixes or clears contradictory values.
