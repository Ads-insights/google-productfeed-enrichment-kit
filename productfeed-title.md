---
name: productfeed-title
description: Optimize and generate product titles for Google Shopping product feeds. Improves existing titles to match Google's best practices per vertical, or generates new titles when missing. Triggers when user uploads a product feed and wants to optimize [title] or [structured_title] values. Also triggers when user mentions "title optimization", "titel optimalisatie", "product titles", "improve titles", or asks about Shopping title best practices. Use this skill even if the user just says "optimize my titles" or "fix the product titles" with an uploaded file.
---

# Feed Title Optimizer

Optimize existing product titles or generate new ones following Google's best practices per product vertical.

## Why this matters

The `[title]` attribute is the single most impactful attribute in Google Shopping:
- The title is the #1 factor Google uses to match products to search queries
- Users see the title first — it determines whether they click
- Optimized titles consistently drive 30-250% more clicks (industry case studies)
- Only the first ~70 characters are visible on most screens
- Google uses title content to determine ad relevance and auction placement

## Output attribute

This skill outputs to the regular `title` column in the supplemental feed. The supplemental feed title overwrites the primary feed title in Google Merchant Center.

**Note:** Google has a `structured_title` attribute with an AI-disclosure flag (`trained_algorithmic_media`) intended for AI-generated titles. However, via a supplemental feed this is not practically enforceable. If a merchant explicitly wants to be compliant with this policy, they can rename the output column to `structured_title` and prefix each value with `trained_algorithmic_media:`. For most use cases, the regular `title` column works fine.

## Two modes of operation

**This skill is unique: it may modify existing values, not just fill empty ones.**

**Mode 1: Generate** — When title is empty, create a title from available attributes.
**Mode 2: Optimize** — When title exists but doesn't follow best practices for its vertical, improve it.

Before optimizing, the skill analyzes each title against Google's recommended structure for that product's vertical. Only titles that score poorly get rewritten.

## Google's title formula per vertical

The most important detail FIRST (first 70 chars are critical):

**Apparel & Accessories:**
`Brand + Gender + Product Type + Attributes (Color, Size, Material)`
Example: "Nike Heren Air Max 90 Sneakers - Zwart/Wit - Maat 43"

**Electronics:**
`Brand + Product Type + Model + Key Spec`
Example: "Samsung Galaxy S24 Ultra 256GB Smartphone Titanium Blue"

**Home & Garden / Furniture:**
`Brand + Product Type + Key Feature + Material/Color`
Example: "IKEA Kallax Stellingkast Eikenhout 77x77cm"

**Health & Beauty:**
`Brand + Product Line + Product Type + Size/Variant`
Example: "L'Oréal Paris Revitalift Anti-Rimpel Dagcrème 50ml"

**Food & Beverages:**
`Brand + Product Name + Flavor/Variant + Size`
Example: "Douwe Egberts Aroma Rood Koffiepads 36 stuks"

**Sports & Outdoors:**
`Brand + Product Type + Key Feature + Variant`
Example: "Bosch Serie 6 Vaatwasser Vrijstaand RVS"

**Baby & Kids:**
`Brand + Product Type + Age Range + Key Feature`
Example: "Maxi-Cosi Pebble 360 Autostoel i-Size 0-15 maanden"

**Books & Media:**
`Title + Author/Artist + Format + Edition`
Example: "Harry Potter en de Steen der Wijzen - J.K. Rowling - Paperback"

**Arts & Crafts / Memorial / Specialty:**
`Brand/Collection + Product Type + Key Feature + Size/Variant`
Example: "KitchenAid Artisan Standmixer 4.8L Rood"

**Generic/Other:**
`Brand + Product Type + Key Differentiator + Variant`

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

Report: total products, how many have empty titles, average title length, title quality distribution.

### Step 2: Title quality scoring

Score each existing title to determine if it needs optimization.

```python
import re

def detect_vertical(product_type, google_category, title):
    """Detect the product vertical for title formula selection."""
    all_text = f"{str(product_type)} {str(google_category)} {str(title)}".lower()

    verticals = {
        'apparel': ['clothing', 'apparel', 'shoes', 'kleding', 'schoenen', 'shirt',
                     'broek', 'jurk', 'jas', 'sneaker', 'accessoires',
                     'bekleidung', 'schuhe', 'hemd', 'hose', 'kleid', 'jacke',
                     'vêtements', 'chaussures', 'robe', 'veste',
                     'abbigliamento', 'scarpe', 'camicia', 'vestito',
                     'ropa', 'zapatos', 'camisa', 'vestido'],
        'electronics': ['electronics', 'phone', 'laptop', 'tablet', 'camera', 'audio',
                        'telefoon', 'computer', 'speaker', 'koptelefoon', 'tv',
                        'elektronik', 'telefon', 'lautsprecher', 'fernseher',
                        'électronique', 'téléphone', 'ordinateur',
                        'elettronica', 'telefono', 'altoparlante'],
        'home': ['home', 'garden', 'furniture', 'meubel', 'lamp', 'kussen', 'gordijn',
                 'huis', 'tuin', 'bed', 'bank', 'stoel', 'tafel', 'kast',
                 'möbel', 'garten', 'stuhl', 'tisch', 'schrank', 'lampe', 'vorhang',
                 'maison', 'jardin', 'meuble', 'chaise', 'table', 'armoire',
                 'casa', 'giardino', 'mobile', 'sedia', 'tavolo',
                 'hogar', 'jardín', 'mueble', 'silla', 'mesa'],
        'health_beauty': ['health', 'beauty', 'cosmetic', 'skincare', 'parfum',
                          'shampoo', 'crème', 'gezondheid', 'verzorging',
                          'gesundheit', 'kosmetik', 'pflege',
                          'santé', 'beauté', 'cosmétique', 'soin',
                          'salute', 'bellezza', 'cosmetico',
                          'salud', 'belleza', 'cosmético'],
        'food': ['food', 'beverage', 'voeding', 'drank', 'koffie', 'thee', 'wijn',
                 'lebensmittel', 'getränk', 'kaffee', 'tee', 'wein',
                 'alimentation', 'boisson', 'café', 'thé', 'vin',
                 'alimento', 'bevanda', 'caffè', 'vino'],
        'sports': ['sport', 'fitness', 'bike', 'fiets', 'outdoor', 'e-bike',
                   'fahrrad', 'vélo', 'bicicletta', 'bicicleta'],
        'baby': ['baby', 'toddler', 'peuter', 'kinder', 'kids',
                 'kleinkind', 'säugling', 'bébé', 'enfant',
                 'bambino', 'neonato', 'bebé', 'niño'],
        'books': ['book', 'boek', 'media', 'dvd', 'cd', 'vinyl',
                  'buch', 'bücher', 'livre', 'libro'],
        'specialty': ['candle', 'kaars', 'gift', 'cadeau', 'decoration', 'decoratie',
                      'kerze', 'geschenk', 'dekoration',
                      'bougie', 'décoration',
                      'candela', 'regalo', 'decorazione',
                      'vela', 'decoración'],
    }

    for vertical, keywords in verticals.items():
        if any(kw in all_text for kw in keywords):
            return vertical
    return 'generic'


def score_title(title, brand='', color='', size='', material='', vertical='generic'):
    """Score a title's quality. Returns (score 0-100, issues list)."""
    title_str = str(title).strip() if pd.notna(title) else ''
    if not title_str:
        return 0, ['empty']

    score = 50  # Start at 50, adjust up/down
    issues = []

    # Length checks
    length = len(title_str)
    if length < 25:
        score -= 20
        issues.append('too_short')
    elif length < 50:
        score -= 10
        issues.append('short')
    elif length >= 70:
        score += 10  # Good use of space

    # Brand presence (important for most verticals)
    brand_str = str(brand).strip().lower() if pd.notna(brand) else ''
    if brand_str and brand_str not in ['', 'nan']:
        if brand_str in title_str.lower():
            score += 10
        else:
            score -= 10
            issues.append('missing_brand')

    # Color presence (important for apparel, home)
    color_str = str(color).strip().lower() if pd.notna(color) else ''
    if color_str and color_str not in ['', 'nan']:
        if color_str in title_str.lower():
            score += 5
        elif vertical in ['apparel', 'home', 'specialty']:
            score -= 5
            issues.append('missing_color')

    # Size presence (important for apparel)
    size_str = str(size).strip().lower() if pd.notna(size) else ''
    if size_str and size_str not in ['', 'nan']:
        if size_str in title_str.lower() or any(s in title_str.lower() for s in size_str.split('/')):
            score += 5
        elif vertical == 'apparel':
            score -= 5
            issues.append('missing_size')

    # Quality red flags
    if title_str.isupper():
        score -= 20
        issues.append('all_caps')
    if re.search(r'[!]{2,}|[?]{2,}', title_str):
        score -= 15
        issues.append('excessive_punctuation')
    # Promotional text detection (NL, EN, DE, FR, IT, ES, PT)
    if re.search(r'\b(?:gratis|free shipping|korting|sale|aanbieding|beste prijs|discount|best price|lowest price|rabatt|angebot|versandkostenfrei|soldes|réduction|livraison gratuite|saldi|sconto|spedizione gratuita|rebajas|descuento|envío gratis|saldos|frete grátis)\b', title_str.lower()):
        score -= 20
        issues.append('promotional_text')
    # CTA detection (NL, EN, DE, FR, IT, ES, PT)
    if re.search(r'\b(?:koop nu|bestel|buy now|order now|shop now|add to cart|jetzt kaufen|jetzt bestellen|acheter maintenant|commander|acquista ora|compra ahora|comprar agora|compre agora)\b', title_str.lower()):
        score -= 20
        issues.append('cta_in_title')

    # Shop name / suffix noise
    if '|' in title_str:
        score -= 5
        issues.append('contains_separator')
    # Shop suffix detection (NL, EN, DE, FR, IT, ES, PT)
    if re.search(r'[-–—]\s*(?:kopen|webshop|online|shop|buy|order|store|kaufen|bestellen|acheter|boutique|acquista|negozio|comprar|tienda|loja)\s*$', title_str.lower()):
        score -= 10
        issues.append('shop_suffix')

    return max(0, min(100, score)), issues


OPTIMIZATION_THRESHOLD = 60  # Titles scoring below this get optimized
```

### Step 3: Title generation / optimization

```python
def build_optimized_title(title, brand='', color='', size='', material='',
                          product_type='', google_category='', description='',
                          vertical='generic'):
    """Build an optimized title following Google's formula for the detected vertical.
    Returns (new_title, action taken)."""

    title_str = str(title).strip() if pd.notna(title) else ''
    brand_str = str(brand).strip() if pd.notna(brand) else ''
    color_str = str(color).strip() if pd.notna(color) else ''
    size_str = str(size).strip() if pd.notna(size) else ''
    material_str = str(material).strip() if pd.notna(material) else ''
    ptype_str = str(product_type).strip() if pd.notna(product_type) else ''

    # If title is empty, build from scratch
    if not title_str or title_str.lower() in ['nan', 'none']:
        parts = []
        if brand_str and brand_str.lower() not in ['nan', 'none']:
            parts.append(brand_str)

        # Try to get a product name from product_type
        if ptype_str and ptype_str.lower() not in ['nan', 'none']:
            # Use the last level of the product_type hierarchy
            type_parts = ptype_str.split('>')
            parts.append(type_parts[-1].strip())

        if material_str and material_str.lower() not in ['nan', 'none']:
            parts.append(material_str)
        if color_str and color_str.lower() not in ['nan', 'none']:
            parts.append(color_str)
        if size_str and size_str.lower() not in ['nan', 'none']:
            parts.append(size_str)

        if len(parts) >= 2:
            return ' '.join(parts)[:150], 'generated'
        return '', 'insufficient_data'

    # If title exists, optimize it
    optimized = title_str

    # Step 1: Remove shop name suffixes (all languages)
    optimized = re.sub(r'\s*\|\s*[^|]+$', '', optimized)
    # NL suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Kopen|Webshop|Online kopen|Bestellen|Nu bestellen).*$', '', optimized, flags=re.I)
    # EN suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Buy now|Shop now|Order now|Buy online|Official store|Free delivery).*$', '', optimized, flags=re.I)
    # DE suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Jetzt kaufen|Jetzt bestellen|Online kaufen|Kostenloser Versand|Offizieller Shop).*$', '', optimized, flags=re.I)
    # FR suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Acheter maintenant|Commander|Acheter en ligne|Livraison gratuite|Boutique officielle).*$', '', optimized, flags=re.I)
    # IT suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Acquista ora|Compra ora|Acquista online|Spedizione gratuita|Negozio ufficiale).*$', '', optimized, flags=re.I)
    # ES suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Compra ahora|Comprar ahora|Comprar online|Envío gratis|Tienda oficial).*$', '', optimized, flags=re.I)
    # PT suffixes
    optimized = re.sub(r'\s*[-–—]\s*(?:Compre agora|Comprar agora|Comprar online|Frete grátis|Loja oficial).*$', '', optimized, flags=re.I)

    # Step 2: Remove promotional text (all languages, word-boundary safe)
    optimized = re.sub(r'\b(?:gratis verzending|free shipping|aanbieding|korting|discount|best price|lowest price|beste prijs|versandkostenfrei|rabatt|angebot|livraison gratuite|soldes|réduction|spedizione gratuita|saldi|sconto|envío gratis|rebajas|descuento|frete grátis|saldos)\b', '', optimized, flags=re.I)

    # Step 3: Fix ALL CAPS
    if optimized.isupper():
        optimized = optimized.title()

    # Step 4: Add missing brand at the start (if not already there)
    if brand_str and brand_str.lower() not in ['nan', 'none', '']:
        if brand_str.lower() not in optimized.lower():
            optimized = f"{brand_str} {optimized}"

    # Step 5: Add missing color (for relevant verticals)
    if color_str and color_str.lower() not in ['nan', 'none', '']:
        if color_str.lower() not in optimized.lower():
            if vertical in ['apparel', 'home', 'specialty', 'sports']:
                optimized = f"{optimized} {color_str}"

    # Step 6: Add missing size (for apparel)
    if size_str and size_str.lower() not in ['nan', 'none', '']:
        if size_str.lower() not in optimized.lower():
            if vertical == 'apparel':
                optimized = f"{optimized} {size_str}"

    # Step 7: Clean up whitespace
    optimized = re.sub(r'\s+', ' ', optimized).strip()

    # Step 8: Enforce 150 char limit, cut at word boundary
    if len(optimized) > 150:
        optimized = optimized[:150].rsplit(' ', 1)[0]

    if optimized != title_str:
        return optimized, 'optimized'
    return title_str, 'unchanged'
```

### Step 4: Apply and build supplemental feed

```python
id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)
brand_col = next((c for c in df.columns if c in ['brand', 'merk', 'manufacturer']), None)
color_col = next((c for c in df.columns if c in ['color', 'kleur', 'colour']), None)
size_col = next((c for c in df.columns if c in ['size', 'grootte', 'maat']), None)
material_col = next((c for c in df.columns if c in ['material', 'materiaal']), None)
ptype_col = next((c for c in df.columns if c in ['product_type', 'producttype']), None)
gpc_col = next((c for c in df.columns if c in ['google_product_category', 'google productcategorie']), None)

results = {'generated': 0, 'optimized': 0, 'unchanged': 0, 'insufficient': 0}
title_values = []

for idx, row in df.iterrows():
    title = row.get(title_col, '')
    brand = row.get(brand_col, '')
    color = row.get(color_col, '')
    size = row.get(size_col, '')
    material = row.get(material_col, '')
    ptype = row.get(ptype_col, '')
    gpc = row.get(gpc_col, '')
    desc = row.get(desc_col, '')

    # Detect vertical
    vertical = detect_vertical(ptype, gpc, title)

    # Score current title
    score, issues = score_title(title, brand, color, size, material, vertical)

    if score >= OPTIMIZATION_THRESHOLD and str(title).strip().lower() not in ['', 'nan', 'none']:
        # Title is good enough — don't touch it
        title_values.append('')  # Empty = don't override in supplemental
        results['unchanged'] += 1
    else:
        # Title needs work
        new_title, action = build_optimized_title(
            title, brand, color, size, material, ptype, gpc, desc, vertical)

        if action == 'insufficient_data':
            title_values.append('')
            results['insufficient'] += 1
        elif action in ['generated', 'optimized']:
            title_values.append(new_title)
            results[action] += 1
        else:
            title_values.append('')
            results['unchanged'] += 1

supplemental = pd.DataFrame({
    'id': df[id_col],
    'title': title_values,
})

output_path = "/mnt/user-data/outputs/supplemental_feed_title.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 5: Report

Present a detailed summary:
- Total products analyzed
- Titles unchanged (already good): X (score ≥ 60)
- Titles optimized (improved): X
- Titles generated (was empty): X
- Titles with insufficient data: X
- Average title score before vs. after
- Show 15-20 before/after examples for spot-checking, grouped by action type
- List the most common issues found (missing brand, too short, promotional text, etc.)

## Important guardrails

- Only optimize titles that score below the threshold (60) — don't "fix" titles that are already good
- Never add promotional text (pricing, shipping, "gratis", "beste prijs")
- Never use ALL CAPS for emphasis
- Put the most important info in the first 70 characters
- Language must match the feed language
- Max 150 characters
- Always preserve the product's accuracy — a "better sounding" but less accurate title is worse
- Show before/after examples so the user can verify the changes make sense
- The scoring system is a guideline — always let the user decide which optimizations to keep

## Title quality scoring reference

| Score | Meaning | Action |
|---|---|---|
| 80-100 | Excellent title, follows best practices | Don't touch |
| 60-79 | Acceptable, minor improvements possible | Don't touch (threshold) |
| 40-59 | Needs improvement, missing key elements | Optimize |
| 20-39 | Poor, significant issues | Optimize |
| 0-19 | Very poor or empty | Generate / major rewrite |

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `title`
- Empty cells = title was already good (no override needed in supplemental feed)
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet
- Include ALL products (empty cells for unchanged titles)
- **Note**: For merchants who want to explicitly disclose AI-generated titles, the column can be renamed to `structured_title` with values prefixed by `trained_algorithmic_media:`. For most supplemental feed use cases this is not necessary.
