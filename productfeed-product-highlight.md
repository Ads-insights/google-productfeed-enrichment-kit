---
name: productfeed-product-highlight
description: Generate product_highlight attributes for Google Shopping product feeds. Creates 4-6 short, scannable USP bullet points per product based on available feed data. Triggers when user uploads a product feed and wants to populate [product_highlight] values. Also triggers when user mentions "product highlights", "USP bullets", "product features", "kenmerken", or asks about product highlights. Use this skill even if the user just says "generate highlights" or "fill in the product highlights" with an uploaded file.
---

# Feed Product Highlight Generator

Generate `product_highlight` values — short, scannable bullet points that showcase the most important features of each product.

## Why this matters

The `[product_highlight]` attribute is recommended and high-impact in Google Shopping:
- Appears as bullet points beneath the product description in free listings
- Helps shoppers quickly scan key features and make purchase decisions
- Improves product relevance and matching to search queries
- Gives a competitive edge over listings without highlights (similar to Amazon bullet points)

## Google's specifications and rules

**Format requirements:**
- Minimum 2, maximum 100 highlights per product (Google recommends 4-6)
- Each highlight max 150 characters
- Separate multiple highlights with commas in the feed
- Must be in the same language as the feed

**Content rules — what TO do:**
- Write short, scannable sentence fragments (not full sentences)
- Focus on key features, specs, and benefits
- Answer the most common shopper questions about the product
- Be factual and specific ("100% biologisch katoen" not "geweldig materiaal")

**Content rules — what NOT to do:**
- No promotional text (no "Gratis verzending!", "Beste prijs!")
- No pricing, shipping, or delivery information
- No SEO keywords or keyword stuffing
- No comparisons to other products ("beter dan merk X")
- No company history or policies
- No ALL CAPS or gimmicky symbols
- No duplicate information within the highlights
- No references to categorization systems
- No information about compatible products or accessories

## Workflow

### Step 1: Load and inspect the feed

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

Report: total products, how many already have highlights, which source columns are available for generation.

### Step 2: Gather all available product data

For each product, collect ALL available information to generate highlights from:

```python
def gather_product_context(row, col_map):
    """Collect all available data for a product into a context dict."""
    context = {}
    for key, possible_cols in col_map.items():
        for col in possible_cols:
            if col in row.index:
                val = str(row[col]).strip() if pd.notna(row[col]) else ''
                if val and val.lower() not in ['', 'nan', 'none']:
                    context[key] = val
                    break
    return context

COL_MAP = {
    'title': ['title', 'titel', 'product name'],
    'description': ['description', 'beschrijving', 'product description'],
    'brand': ['brand', 'merk', 'manufacturer'],
    'color': ['color', 'kleur', 'colour'],
    'material': ['material', 'materiaal'],
    'size': ['size', 'grootte', 'maat'],
    'pattern': ['pattern', 'patroon'],
    'product_type': ['product_type', 'producttype'],
    'google_category': ['google_product_category', 'google productcategorie'],
    'condition': ['condition', 'staat'],
    'weight': ['product_weight', 'verzendgewicht', 'shipping weight', 'product weight'],
    'bullet1': ['product bullet point 1'],
    'bullet2': ['product bullet point 2'],
    'bullet3': ['product bullet point 3'],
    'labels': ['labels', 'aangepast label 1', 'custom_label_0'],
}
```

### Step 3: Highlight generation strategy

This is a GENERATIVE skill — unlike extraction skills, it creates new text. The approach:

**Strategy 1: Assemble from known attributes (deterministic)**
If the feed has structured data (material, color, size, weight, etc.), compose highlights directly from these attributes. This is the most reliable method.

**Strategy 2: Extract key phrases from description (semi-deterministic)**
If a description exists, scan for feature-like phrases and convert them into highlight bullets.

**Strategy 3: Generate from title + product type (AI-assisted)**
When minimal data is available, use the title and product type to infer reasonable highlights. Flag these as lower confidence.

```python
import re

def detect_feed_language(df, title_col):
    """Detect feed language by checking titles for Dutch vs English words."""
    nl_words = {'de', 'het', 'een', 'van', 'voor', 'met', 'en', 'handgemaakt', 'handgemaakte', 'maat', 'kleur'}
    en_words = {'the', 'a', 'an', 'for', 'with', 'and', 'handmade', 'size', 'color', 'colour'}
    nl_score = 0
    en_score = 0
    sample = df[title_col].dropna().head(50)
    for title in sample:
        words = set(str(title).lower().split())
        nl_score += len(words & nl_words)
        en_score += len(words & en_words)
    return 'en' if en_score > nl_score else 'nl'

# Bilingual templates
TEMPLATES = {
    'nl': {
        'material': 'Gemaakt van {value}',
        'brand': 'Van het merk {value}',
        'size': 'Maat/formaat: {value}',
        'weight': 'Gewicht: {value}',
        'condition': 'Conditie: {value}',
        'category': 'Categorie: {value}',
        'quality_words': {
            'handgemaakt': 'Handgemaakt product',
            'handgemaakte': 'Handgemaakt met zorg en aandacht',
            'premium': 'Premium kwaliteit',
            'professioneel': 'Professionele kwaliteit',
            'biologisch': 'Biologisch gecertificeerd',
            'duurzaam': 'Duurzaam geproduceerd',
            'waterdicht': 'Waterdicht ontwerp',
            'draadloos': 'Draadloos te gebruiken',
            'oplaadbaar': 'Oplaadbare batterij',
            'inklapbaar': 'Inklapbaar voor eenvoudige opslag',
            'verstelbaar': 'Verstelbaar voor optimaal comfort',
            'ergonomisch': 'Ergonomisch ontworpen',
        },
        'filler_patterns': [
            r'^(?:klik|bestel|koop|bekijk|ontdek)',
            r'(?:gratis|korting|aanbieding|actie)',
            r'(?:wij|ons|onze|onze winkel)',
        ],
    },
    'en': {
        'material': 'Made from {value}',
        'brand': 'By {value}',
        'size': 'Size: {value}',
        'weight': 'Weight: {value}',
        'condition': 'Condition: {value}',
        'category': 'Category: {value}',
        'quality_words': {
            'handmade': 'Handmade product',
            'hand-made': 'Handcrafted with care',
            'premium': 'Premium quality',
            'professional': 'Professional grade',
            'organic': 'Certified organic',
            'sustainable': 'Sustainably produced',
            'waterproof': 'Waterproof design',
            'wireless': 'Wireless connectivity',
            'rechargeable': 'Rechargeable battery',
            'foldable': 'Foldable for easy storage',
            'collapsible': 'Collapsible for easy storage',
            'adjustable': 'Adjustable for optimal comfort',
            'ergonomic': 'Ergonomic design',
            'lightweight': 'Lightweight construction',
            'portable': 'Portable and easy to carry',
            'eco-friendly': 'Eco-friendly materials',
        },
        'filler_patterns': [
            r'^(?:click|order|buy|view|discover|shop)',
            r'(?:free shipping|discount|sale|deal|offer)',
            r'(?:we |our |our shop|our store)',
        ],
    },
}

def generate_highlights_from_attributes(context, lang='nl'):
    """Generate highlights from structured product data. Returns list of strings."""
    highlights = []
    t = TEMPLATES[lang]
    title = context.get('title', '')
    brand = context.get('brand', '')
    material = context.get('material', '')
    color = context.get('color', '')
    size = context.get('size', '')
    weight = context.get('weight', '')
    condition = context.get('condition', '')
    product_type = context.get('product_type', '')
    description = context.get('description', '')

    # Highlight 1: Material (if available)
    if material:
        highlights.append(t['material'].format(value=material))

    # Highlight 2: Key feature from title
    title_lower = title.lower()
    for word, highlight in t['quality_words'].items():
        if word in title_lower and highlight not in highlights:
            highlights.append(highlight)
            break

    # Highlight 3: Brand
    if brand and brand.lower() not in ['', 'nan', 'none']:
        highlights.append(t['brand'].format(value=brand))

    # Highlight 4: Size/weight
    if size:
        highlights.append(t['size'].format(value=size))
    if weight:
        highlights.append(t['weight'].format(value=weight))

    # Highlight 5: Condition
    if condition and condition.lower() != 'new':
        highlights.append(t['condition'].format(value=condition))

    # Highlight 6: Extract features from description
    if description:
        desc_highlights = extract_highlights_from_description(description, lang)
        for h in desc_highlights:
            if h not in highlights and len(highlights) < 6:
                highlights.append(h)

    # Highlight from existing bullet points in feed
    for bp_key in ['bullet1', 'bullet2', 'bullet3']:
        bp = context.get(bp_key, '')
        if bp and len(highlights) < 6:
            bp_clean = bp.strip()[:150]
            if bp_clean not in highlights:
                highlights.append(bp_clean)

    return highlights[:6]


def extract_highlights_from_description(description, lang='nl'):
    """Extract feature-like phrases from a product description."""
    highlights = []
    desc = str(description).strip()
    if not desc or desc.lower() == 'nan':
        return highlights

    fragments = re.split(r'[•·\n\r;]|\.\s+', desc)
    filler_patterns = TEMPLATES[lang]['filler_patterns']

    for fragment in fragments:
        fragment = fragment.strip()
        if 10 <= len(fragment) <= 150:
            is_filler = any(re.search(p, fragment.lower()) for p in filler_patterns)
            if not is_filler:
                highlights.append(fragment)

    return highlights[:3]


def generate_highlights(context, lang='nl'):
    """Main highlight generation function.
    Returns (highlights_string, source, confidence)."""

    highlights = generate_highlights_from_attributes(context, lang)
    t = TEMPLATES[lang]

    if not highlights:
        return '', 'none', 'unresolved'

    # Ensure minimum of 2 highlights (Google requirement)
    if len(highlights) < 2:
        product_type = context.get('product_type', '')
        if product_type:
            highlights.append(t['category'].format(value=product_type))

    if len(highlights) < 2:
        return '', 'insufficient data', 'unresolved'

    # Validate each highlight: max 150 chars, no promotional text, no internal commas
    promo_pattern = (r'(?:gratis|korting|actie|aanbieding|beste prijs)'
                     if lang == 'nl' else
                     r'(?:free shipping|discount|sale|deal|best price|lowest price)')
    validated = []
    for h in highlights:
        h = h.strip()
        # Strip commas — Google uses commas as the delimiter between highlights
        # in text/TSV feeds. A comma inside a highlight causes Google to split
        # it into separate (broken) entries.
        h = h.replace(',', ' -')
        h = re.sub(r'\s+', ' ', h).strip()
        if len(h) > 150:
            h = h[:147] + '...'
        if not re.search(promo_pattern, h.lower()):
            validated.append(h)

    if len(validated) < 2:
        return '', 'insufficient valid highlights', 'unresolved'

    # Join with comma separation (Google's expected format for text feeds)
    highlights_str = ', '.join(validated)

    # Determine confidence
    has_structured = bool(context.get('material') or context.get('description'))
    confidence = 'high' if has_structured else 'medium'
    source = 'attributes + description' if has_structured else 'title only'

    return highlights_str, source, confidence
```

### Step 4: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
ph_col = next((c for c in df.columns if c in ['product_highlight', 'product highlight']), None)

results = {'generated': 0, 'unresolved': 0, 'already_had': 0}
highlight_values = []

for idx, row in df.iterrows():
    if ph_col:
        current = str(row[ph_col]).strip() if pd.notna(row[ph_col]) else ''
        if current and current.lower() not in ['', 'nan', 'none']:
            results['already_had'] += 1
            highlight_values.append(current)
            continue

    context = gather_product_context(row, COL_MAP)
    highlights_str, source, confidence = generate_highlights(context)

    highlight_values.append(highlights_str)
    if highlights_str:
        results['generated'] += 1
    else:
        results['unresolved'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'product_highlight': highlight_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_product_highlight.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present summary with:
- Fill rate before/after
- Average number of highlights per product
- 10-15 examples showing generated highlights for spot-checking
- Products that couldn't get highlights (insufficient data)
- Flag any highlights that were generated from title-only (lower confidence)

## Important guardrails

- Never overwrite existing product_highlight values
- Minimum 2 highlights per product (Google requirement) — if you can't generate 2, leave empty
- Maximum 6 highlights recommended (max 100 technically allowed)
- Each highlight max 150 characters
- Language must match feed language (Dutch feed → Dutch highlights)
- Never include: promotional text, pricing, shipping info, company info, SEO keywords
- Only describe the product itself — no comparisons, no accessories, no compatible products
- Write as scannable fragments, not full marketing sentences
- "100% biologisch katoen, pre-shrunk" is good. "Ervaar het ongelooflijke comfort van ons premium materiaal" is bad.
- When data is sparse, it's better to generate fewer but accurate highlights than to pad with generic filler

## Quality principles

Product highlights should answer one of these shopper questions:
1. Wat is het gemaakt van? (materiaal, constructie)
2. Wat maakt het bijzonder? (handgemaakt, premium, biologisch)
3. Wat zijn de specificaties? (afmetingen, gewicht, capaciteit)
4. Van welk merk is het? (brand, herkomst)
5. Hoe gebruik je het? (draadloos, oplaadbaar, verstelbaar)
6. Voor wie is het? (leeftijd, gender, doelgroep)

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `product_highlight`
- Highlights are comma-separated within the single `product_highlight` column
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (also empty ones)
