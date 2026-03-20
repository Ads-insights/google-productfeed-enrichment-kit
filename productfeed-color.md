---
name: productfeed-color
description: Extract and fill missing color attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [color] values. Also triggers when user mentions "color attribute", "kleur invullen", "feed color", "missing colors in feed", or asks to enrich/complete/fix color data in a product feed. Use this skill even if the user just says "fill in the colors" or "fix my feed colors" with an uploaded file.
---

# Feed Color Extractor

Extract color values from product titles, descriptions, and other available fields to fill empty `color` attributes in a Google Shopping product feed.

## Why this matters

The `[color]` attribute is marked as high-impact in Google Shopping. Missing color data leads to:
- Disapproved products in apparel/fashion categories
- Missed search queries containing color terms (e.g., "zwarte sneakers", "schwarze Schuhe", "chaussures noires")
- Lower ad relevance and reduced impression share
- Poor variant grouping when `item_group_id` is used

## Multi-language support

This skill supports 7 output languages: NL, EN, DE, FR, IT, ES, PT.

**How it works:**
1. The skill recognizes color words from ALL supported languages as input (a German feed can contain "schwarz", "black", or even "noir" — all are recognized)
2. Each recognized color maps to a **canonical color key** (e.g., `BLACK`)
3. The canonical key is then output in the detected feed language (e.g., `BLACK` → "Schwarz" for DE, "Black" for EN, "Zwart" for NL)

**Language detection:** When run standalone, the skill detects the feed language using the same `detect_output_language()` function as the orchestrator. When run via the orchestrator, it receives `output_lang` as a parameter.

## Workflow

### Step 1: Load and inspect the feed

Read the uploaded file (CSV, TSV, or Excel). Identify which columns are present. The color column may be named any of: `color`, `kleur`, `Color`, `colour`, `g:color`, `color [color]`, `farbe`, `couleur`, `colore`. Also locate source columns for extraction: `title`, `description`, `product_type`, `google_product_category`, `image_link`.

```python
import pandas as pd

# Detect file type and load
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

# Normalize column names to lowercase for matching
df.columns = [c.strip().lower() for c in df.columns]
```

Report to the user:
- Total products in feed
- How many have an empty/missing color value
- Which source columns are available for extraction

### Step 2: Language detection

When run standalone (not via orchestrator), detect the output language:

```python
import re

LANGUAGE_SIGNALS = {
    'nl': r'\b(?:de|het|een|van|voor|met|uit|bij|ook|niet|maar|deze|naar)\b',
    'en': r'\b(?:the|a|an|for|with|and|from|this|that|not|but|also|into)\b',
    'de': r'\b(?:der|die|das|ein|eine|und|für|mit|von|aus|auf|ist|den|dem|nicht|auch|oder)\b',
    'fr': r'\b(?:le|la|les|un|une|des|et|pour|avec|dans|sur|qui|est|pas|mais|sont|cette)\b',
    'it': r'\b(?:il|la|le|un|una|gli|dei|per|con|che|non|sono|nel|dal|alla|questo|questa)\b',
    'es': r'\b(?:el|la|los|las|un|una|y|para|con|del|por|que|no|se|este|esta|más)\b',
    'pt': r'\b(?:o|a|os|as|um|uma|e|para|com|que|não|se|do|da|no|na|por|mais)\b',
}

def detect_output_language(df, title_col):
    """Detect feed language from titles. Same logic as orchestrator."""
    # Check feed label / language column first
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
    
    # Statistical detection from titles
    sample = ' '.join(df[title_col].dropna().head(100).tolist()).lower()
    scores = {}
    for lang, pattern in LANGUAGE_SIGNALS.items():
        scores[lang] = len(re.findall(pattern, sample))
    scores = {k: v for k, v in scores.items() if v > 0}
    if not scores:
        return 'en'
    return max(scores, key=scores.get)
```

### Step 3: Color vocabulary — multi-language

The architecture: each input word maps to a **canonical key**. Each canonical key has an output word per language. This separates recognition (input) from output (language-dependent).

```python
# Canonical color key → output per language
# Only include colors where we have confident translations.
# If a language is missing for a color, fall back to English.
COLOR_OUTPUT = {
    'BLACK':       {'nl': 'Zwart',      'en': 'Black',     'de': 'Schwarz',    'fr': 'Noir',       'it': 'Nero',       'es': 'Negro',      'pt': 'Preto'},
    'WHITE':       {'nl': 'Wit',        'en': 'White',     'de': 'Weiß',       'fr': 'Blanc',      'it': 'Bianco',     'es': 'Blanco',     'pt': 'Branco'},
    'RED':         {'nl': 'Rood',       'en': 'Red',       'de': 'Rot',        'fr': 'Rouge',      'it': 'Rosso',      'es': 'Rojo',       'pt': 'Vermelho'},
    'BLUE':        {'nl': 'Blauw',      'en': 'Blue',      'de': 'Blau',       'fr': 'Bleu',       'it': 'Blu',        'es': 'Azul',       'pt': 'Azul'},
    'GREEN':       {'nl': 'Groen',      'en': 'Green',     'de': 'Grün',       'fr': 'Vert',       'it': 'Verde',      'es': 'Verde',      'pt': 'Verde'},
    'YELLOW':      {'nl': 'Geel',       'en': 'Yellow',    'de': 'Gelb',       'fr': 'Jaune',      'it': 'Giallo',     'es': 'Amarillo',   'pt': 'Amarelo'},
    'ORANGE':      {'nl': 'Oranje',     'en': 'Orange',    'de': 'Orange',     'fr': 'Orange',     'it': 'Arancione',  'es': 'Naranja',    'pt': 'Laranja'},
    'PURPLE':      {'nl': 'Paars',      'en': 'Purple',    'de': 'Lila',       'fr': 'Violet',     'it': 'Viola',      'es': 'Morado',     'pt': 'Roxo'},
    'PINK':        {'nl': 'Roze',       'en': 'Pink',      'de': 'Rosa',       'fr': 'Rose',       'it': 'Rosa',       'es': 'Rosa',       'pt': 'Rosa'},
    'BROWN':       {'nl': 'Bruin',      'en': 'Brown',     'de': 'Braun',      'fr': 'Marron',     'it': 'Marrone',    'es': 'Marrón',     'pt': 'Castanho'},
    'GREY':        {'nl': 'Grijs',      'en': 'Grey',      'de': 'Grau',       'fr': 'Gris',       'it': 'Grigio',     'es': 'Gris',       'pt': 'Cinzento'},
    'BEIGE':       {'nl': 'Beige',      'en': 'Beige',     'de': 'Beige',      'fr': 'Beige',      'it': 'Beige',      'es': 'Beige',      'pt': 'Bege'},
    'CREAM':       {'nl': 'Crème',      'en': 'Cream',     'de': 'Creme',      'fr': 'Crème',      'it': 'Crema',      'es': 'Crema',      'pt': 'Creme'},
    'GOLD':        {'nl': 'Goud',       'en': 'Gold',      'de': 'Gold',       'fr': 'Or',         'it': 'Oro',        'es': 'Dorado',     'pt': 'Dourado'},
    'SILVER':      {'nl': 'Zilver',     'en': 'Silver',    'de': 'Silber',     'fr': 'Argent',     'it': 'Argento',    'es': 'Plata',      'pt': 'Prateado'},
    'BRONZE':      {'nl': 'Brons',      'en': 'Bronze',    'de': 'Bronze',     'fr': 'Bronze',     'it': 'Bronzo',     'es': 'Bronce',     'pt': 'Bronze'},
    'TURQUOISE':   {'nl': 'Turquoise',  'en': 'Turquoise', 'de': 'Türkis',    'fr': 'Turquoise',  'it': 'Turchese',   'es': 'Turquesa',   'pt': 'Turquesa'},
    'BURGUNDY':    {'nl': 'Bordeaux',   'en': 'Burgundy',  'de': 'Bordeaux',   'fr': 'Bordeaux',   'it': 'Bordeaux',   'es': 'Burdeos',    'pt': 'Bordô'},
    'CORAL':       {'nl': 'Koraal',     'en': 'Coral',     'de': 'Koralle',    'fr': 'Corail',     'it': 'Corallo',    'es': 'Coral',      'pt': 'Coral'},
    'MAGENTA':     {'nl': 'Magenta',    'en': 'Magenta',   'de': 'Magenta',    'fr': 'Magenta',    'it': 'Magenta',    'es': 'Magenta',    'pt': 'Magenta'},
    'NAVY':        {'nl': 'Navy',       'en': 'Navy',      'de': 'Marineblau', 'fr': 'Bleu Marine','it': 'Blu Navy',   'es': 'Azul Marino','pt': 'Azul Marinho'},
    'MINT':        {'nl': 'Mint',       'en': 'Mint',      'de': 'Mint',       'fr': 'Menthe',     'it': 'Menta',      'es': 'Menta',      'pt': 'Menta'},
    'IVORY':       {'nl': 'Ivoor',      'en': 'Ivory',     'de': 'Elfenbein',  'fr': 'Ivoire',     'it': 'Avorio',     'es': 'Marfil',     'pt': 'Marfim'},
    'KHAKI':       {'nl': 'Khaki',      'en': 'Khaki',     'de': 'Khaki',      'fr': 'Kaki',       'it': 'Kaki',       'es': 'Caqui',      'pt': 'Cáqui'},
    'TAUPE':       {'nl': 'Taupe',      'en': 'Taupe',     'de': 'Taupe',      'fr': 'Taupe',      'it': 'Talpa',      'es': 'Topo',       'pt': 'Taupe'},
    'LILAC':       {'nl': 'Lila',       'en': 'Lilac',     'de': 'Flieder',    'fr': 'Lilas',      'it': 'Lilla',      'es': 'Lila',       'pt': 'Lilás'},
    'TEAL':        {'nl': 'Petrol',     'en': 'Teal',      'de': 'Petrol',     'fr': 'Sarcelle',   'it': 'Ottanio',    'es': 'Verde Azulado', 'pt': 'Azul Petróleo'},
    'OLIVE':       {'nl': 'Olijfgroen', 'en': 'Olive',     'de': 'Olivgrün',   'fr': 'Olive',      'it': 'Oliva',      'es': 'Oliva',      'pt': 'Azeitona'},
    'COGNAC':      {'nl': 'Cognac',     'en': 'Cognac',    'de': 'Cognac',     'fr': 'Cognac',     'it': 'Cognac',     'es': 'Coñac',      'pt': 'Cognac'},
    'CAMEL':       {'nl': 'Camel',      'en': 'Camel',     'de': 'Camel',      'fr': 'Camel',      'it': 'Cammello',   'es': 'Camello',    'pt': 'Camel'},
    'ANTHRACITE':  {'nl': 'Antraciet',  'en': 'Anthracite','de': 'Anthrazit',  'fr': 'Anthracite', 'it': 'Antracite',  'es': 'Antracita',  'pt': 'Antracite'},
    'ECRU':        {'nl': 'Ecru',       'en': 'Ecru',      'de': 'Ecru',       'fr': 'Écru',       'it': 'Ecru',       'es': 'Crudo',      'pt': 'Cru'},
    'OFFWHITE':    {'nl': 'Off-White',  'en': 'Off-White',  'de': 'Offwhite',  'fr': 'Blanc Cassé','it': 'Bianco Sporco','es': 'Blanco Roto','pt': 'Branco Sujo'},
    'INDIGO':      {'nl': 'Indigo',     'en': 'Indigo',    'de': 'Indigo',     'fr': 'Indigo',     'it': 'Indaco',     'es': 'Índigo',     'pt': 'Índigo'},
    'TERRACOTTA':  {'nl': 'Terracotta', 'en': 'Terracotta','de': 'Terrakotta', 'fr': 'Terracotta', 'it': 'Terracotta', 'es': 'Terracota',  'pt': 'Terracota'},
    'MUSTARD':     {'nl': 'Mosterdgeel','en': 'Mustard',   'de': 'Senfgelb',   'fr': 'Moutarde',   'it': 'Senape',     'es': 'Mostaza',    'pt': 'Mostarda'},
    'NATURAL':     {'nl': 'Naturel',    'en': 'Natural',   'de': 'Natur',      'fr': 'Naturel',    'it': 'Naturale',   'es': 'Natural',    'pt': 'Natural'},
    'RUST':        {'nl': 'Roest',      'en': 'Rust',      'de': 'Rostbraun',  'fr': 'Rouille',    'it': 'Ruggine',    'es': 'Óxido',      'pt': 'Ferrugem'},
    'OCHRE':       {'nl': 'Oker',       'en': 'Ochre',     'de': 'Ocker',      'fr': 'Ocre',       'it': 'Ocra',       'es': 'Ocre',       'pt': 'Ocre'},
    'COPPER':      {'nl': 'Koper',      'en': 'Copper',    'de': 'Kupfer',     'fr': 'Cuivre',     'it': 'Rame',       'es': 'Cobre',      'pt': 'Cobre'},
    'MULTICOLOURED': {'nl': 'Meerkleurig','en': 'Multicoloured','de': 'Mehrfarbig','fr': 'Multicolore','it': 'Multicolore','es': 'Multicolor','pt': 'Multicolor'},
}

# Input word → canonical key (all supported languages as input)
COLOR_INPUT = {
    # NL input
    'zwart': 'BLACK', 'wit': 'WHITE', 'witte': 'WHITE', 'rood': 'RED', 'rode': 'RED',
    'blauw': 'BLUE', 'blauwe': 'BLUE', 'groen': 'GREEN', 'groene': 'GREEN',
    'geel': 'YELLOW', 'gele': 'YELLOW', 'oranje': 'ORANGE', 'paars': 'PURPLE',
    'roze': 'PINK', 'bruin': 'BROWN', 'bruine': 'BROWN', 'grijs': 'GREY', 'grijze': 'GREY',
    'beige': 'BEIGE', 'crème': 'CREAM', 'creme': 'CREAM',
    'goud': 'GOLD', 'gouden': 'GOLD', 'zilver': 'SILVER', 'zilveren': 'SILVER',
    'brons': 'BRONZE', 'turkoois': 'TURQUOISE', 'bordeaux': 'BURGUNDY', 'bordeauxrood': 'BURGUNDY',
    'koraal': 'CORAL', 'magenta': 'MAGENTA', 'fuchsia': 'MAGENTA',
    'marineblauw': 'NAVY', 'donkerblauw': 'NAVY', 'mintgroen': 'MINT',
    'ivoor': 'IVORY', 'khaki': 'KHAKI', 'kaki': 'KHAKI', 'taupe': 'TAUPE',
    'lila': 'LILAC', 'petrol': 'TEAL', 'olijf': 'OLIVE', 'olijfgroen': 'OLIVE',
    'cognac': 'COGNAC', 'antraciet': 'ANTHRACITE', 'ecru': 'ECRU',
    'indigo': 'INDIGO', 'terra': 'TERRACOTTA', 'terracotta': 'TERRACOTTA',
    'mosterd': 'MUSTARD', 'naturel': 'NATURAL', 'oker': 'OCHRE', 'koper': 'COPPER',
    'meerkleurig': 'MULTICOLOURED',
    # EN input
    'black': 'BLACK', 'white': 'WHITE', 'red': 'RED', 'blue': 'BLUE', 'green': 'GREEN',
    'yellow': 'YELLOW', 'orange': 'ORANGE', 'purple': 'PURPLE', 'violet': 'PURPLE',
    'pink': 'PINK', 'brown': 'BROWN', 'grey': 'GREY', 'gray': 'GREY',
    'cream': 'CREAM', 'gold': 'GOLD', 'golden': 'GOLD', 'silver': 'SILVER',
    'bronze': 'BRONZE', 'turquoise': 'TURQUOISE', 'burgundy': 'BURGUNDY',
    'coral': 'CORAL', 'navy': 'NAVY', 'mint': 'MINT', 'ivory': 'IVORY',
    'teal': 'TEAL', 'olive': 'OLIVE', 'camel': 'CAMEL', 'anthracite': 'ANTHRACITE',
    'offwhite': 'OFFWHITE', 'off-white': 'OFFWHITE', 'mustard': 'MUSTARD',
    'natural': 'NATURAL', 'rust': 'RUST', 'ochre': 'OCHRE', 'copper': 'COPPER',
    'charcoal': 'ANTHRACITE', 'multicoloured': 'MULTICOLOURED', 'multicolored': 'MULTICOLOURED',
    # DE input
    'schwarz': 'BLACK', 'weiß': 'WHITE', 'weiss': 'WHITE', 'rot': 'RED', 'rote': 'RED', 'roter': 'RED',
    'blau': 'BLUE', 'blaue': 'BLUE', 'blauer': 'BLUE', 'grün': 'GREEN', 'grüne': 'GREEN',
    'gelb': 'YELLOW', 'gelbe': 'YELLOW', 'lila': 'LILAC', 'rosa': 'PINK',
    'braun': 'BROWN', 'braune': 'BROWN', 'grau': 'GREY', 'graue': 'GREY',
    'silber': 'SILVER', 'türkis': 'TURQUOISE', 'marineblau': 'NAVY',
    'elfenbein': 'IVORY', 'flieder': 'LILAC', 'anthrazit': 'ANTHRACITE',
    'senfgelb': 'MUSTARD', 'natur': 'NATURAL', 'ocker': 'OCHRE', 'kupfer': 'COPPER',
    'rostbraun': 'RUST', 'terrakotta': 'TERRACOTTA', 'olivgrün': 'OLIVE',
    'mehrfarbig': 'MULTICOLOURED',
    # FR input
    'noir': 'BLACK', 'noire': 'BLACK', 'blanc': 'WHITE', 'blanche': 'WHITE',
    'rouge': 'RED', 'bleu': 'BLUE', 'bleue': 'BLUE', 'vert': 'GREEN', 'verte': 'GREEN',
    'jaune': 'YELLOW', 'violet': 'PURPLE', 'violette': 'PURPLE', 'rose': 'PINK',
    'marron': 'BROWN', 'gris': 'GREY', 'grise': 'GREY', 'doré': 'GOLD', 'dorée': 'GOLD',
    'argent': 'SILVER', 'argenté': 'SILVER', 'ivoire': 'IVORY',
    'moutarde': 'MUSTARD', 'rouille': 'RUST', 'ocre': 'OCHRE', 'cuivre': 'COPPER',
    'naturel': 'NATURAL', 'multicolore': 'MULTICOLOURED',
    # IT input
    'nero': 'BLACK', 'nera': 'BLACK', 'bianco': 'WHITE', 'bianca': 'WHITE',
    'rosso': 'RED', 'rossa': 'RED', 'blu': 'BLUE', 'verde': 'GREEN',
    'giallo': 'YELLOW', 'gialla': 'YELLOW', 'arancione': 'ORANGE',
    'viola': 'PURPLE', 'marrone': 'BROWN', 'grigio': 'GREY', 'grigia': 'GREY',
    'oro': 'GOLD', 'argento': 'SILVER', 'turchese': 'TURQUOISE',
    'avorio': 'IVORY', 'senape': 'MUSTARD', 'ruggine': 'RUST',
    'naturale': 'NATURAL', 'ottanio': 'TEAL',
    # ES input
    'negro': 'BLACK', 'negra': 'BLACK', 'blanco': 'WHITE', 'blanca': 'WHITE',
    'rojo': 'RED', 'roja': 'RED', 'azul': 'BLUE', 'amarillo': 'YELLOW', 'amarilla': 'YELLOW',
    'naranja': 'ORANGE', 'morado': 'PURPLE', 'morada': 'PURPLE',
    'marrón': 'BROWN', 'dorado': 'GOLD', 'dorada': 'GOLD', 'plata': 'SILVER',
    'marfil': 'IVORY', 'mostaza': 'MUSTARD', 'óxido': 'RUST',
    'multicolor': 'MULTICOLOURED',
    # PT input
    'preto': 'BLACK', 'preta': 'BLACK', 'branco': 'WHITE', 'branca': 'WHITE',
    'vermelho': 'RED', 'vermelha': 'RED', 'castanho': 'BROWN', 'castanha': 'BROWN',
    'cinzento': 'GREY', 'cinzenta': 'GREY', 'dourado': 'GOLD', 'dourada': 'GOLD',
    'prateado': 'SILVER', 'prateada': 'SILVER', 'laranja': 'ORANGE',
    'roxo': 'PURPLE', 'roxa': 'PURPLE', 'amarelo': 'YELLOW', 'amarela': 'YELLOW',
    'ferrugem': 'RUST',

    # Composite prefixes — set to None, handled separately
    'donker': None, 'licht': None, 'helder': None,    # NL
    'dark': None, 'light': None, 'bright': None,       # EN
    'dunkel': None, 'hell': None,                       # DE
    'foncé': None, 'clair': None,                       # FR
    'scuro': None, 'chiaro': None,                      # IT
    'oscuro': None, 'claro': None,                      # ES/PT
}

# Multi-word color patterns → canonical key
MULTI_WORD_COLORS = {
    # EN
    'duck egg': 'DUCK_EGG_BLUE', 'duck egg blue': 'DUCK_EGG_BLUE',
    'army green': 'ARMY_GREEN', 'sky blue': 'SKY_BLUE', 'hot pink': 'HOT_PINK',
    'baby blue': 'BABY_BLUE', 'baby pink': 'BABY_PINK', 'royal blue': 'ROYAL_BLUE',
    'forest green': 'FOREST_GREEN', 'ice blue': 'ICE_BLUE', 'rose gold': 'ROSE_GOLD',
    'space grey': 'SPACE_GREY', 'space gray': 'SPACE_GREY',
    'midnight blue': 'MIDNIGHT_BLUE', 'jet black': 'JET_BLACK',
    'steel blue': 'STEEL_BLUE', 'burnt orange': 'BURNT_ORANGE',
    'dark grey': 'DARK_GREY', 'dark gray': 'DARK_GREY', 'light grey': 'LIGHT_GREY',
    'light gray': 'LIGHT_GREY', 'dark blue': 'DARK_BLUE', 'dark green': 'DARK_GREEN',
    'light blue': 'LIGHT_BLUE', 'light green': 'LIGHT_GREEN',
    'light wood': 'LIGHT_WOOD', 'dark wood': 'DARK_WOOD',
    'multi coloured': 'MULTICOLOURED', 'multi-coloured': 'MULTICOLOURED',
    'multi colored': 'MULTICOLOURED', 'multi-colored': 'MULTICOLOURED',
    # DE
    'dunkelblau': 'DARK_BLUE', 'dunkelgrün': 'DARK_GREEN', 'dunkelgrau': 'DARK_GREY',
    'dunkelbraun': 'DARK_BROWN', 'hellblau': 'LIGHT_BLUE', 'hellgrün': 'LIGHT_GREEN',
    'hellgrau': 'LIGHT_GREY', 'hellbraun': 'LIGHT_BROWN',
    # FR
    'bleu foncé': 'DARK_BLUE', 'bleu clair': 'LIGHT_BLUE', 'vert foncé': 'DARK_GREEN',
    'vert clair': 'LIGHT_GREEN', 'gris foncé': 'DARK_GREY', 'gris clair': 'LIGHT_GREY',
    'bleu marine': 'NAVY', 'blanc cassé': 'OFFWHITE',
    # IT
    'blu scuro': 'DARK_BLUE', 'blu chiaro': 'LIGHT_BLUE', 'verde scuro': 'DARK_GREEN',
    'verde chiaro': 'LIGHT_GREEN', 'grigio scuro': 'DARK_GREY', 'grigio chiaro': 'LIGHT_GREY',
    # ES
    'azul oscuro': 'DARK_BLUE', 'azul claro': 'LIGHT_BLUE', 'verde oscuro': 'DARK_GREEN',
    'verde claro': 'LIGHT_GREEN', 'gris oscuro': 'DARK_GREY', 'gris claro': 'LIGHT_GREY',
    'azul marino': 'NAVY', 'blanco roto': 'OFFWHITE',
}

# Multi-word color output (for colors not in the main COLOR_OUTPUT table)
MULTI_WORD_COLOR_OUTPUT = {
    'DUCK_EGG_BLUE':  {'nl': 'Eendenblauw',  'en': 'Duck Egg Blue',  'de': 'Enteneiblau',  'fr': 'Bleu Canard',   'it': 'Blu Uovo d\'Anatra', 'es': 'Azul Huevo de Pato', 'pt': 'Azul Ovo de Pato'},
    'ARMY_GREEN':     {'nl': 'Legergroen',    'en': 'Army Green',     'de': 'Armygrün',     'fr': 'Vert Armée',    'it': 'Verde Militare',     'es': 'Verde Militar',      'pt': 'Verde Militar'},
    'SKY_BLUE':       {'nl': 'Hemelsblauw',   'en': 'Sky Blue',       'de': 'Himmelblau',   'fr': 'Bleu Ciel',     'it': 'Azzurro Cielo',      'es': 'Azul Cielo',         'pt': 'Azul Céu'},
    'HOT_PINK':       {'nl': 'Felroze',       'en': 'Hot Pink',       'de': 'Knallrosa',    'fr': 'Rose Vif',      'it': 'Rosa Acceso',        'es': 'Rosa Fucsia',        'pt': 'Rosa Choque'},
    'BABY_BLUE':      {'nl': 'Babyblauw',     'en': 'Baby Blue',      'de': 'Babyblau',     'fr': 'Bleu Bébé',     'it': 'Azzurro Baby',       'es': 'Azul Bebé',          'pt': 'Azul Bebé'},
    'BABY_PINK':      {'nl': 'Babyroze',      'en': 'Baby Pink',      'de': 'Babyrosa',     'fr': 'Rose Bébé',     'it': 'Rosa Baby',          'es': 'Rosa Bebé',          'pt': 'Rosa Bebé'},
    'ROYAL_BLUE':     {'nl': 'Koningsblauw',  'en': 'Royal Blue',     'de': 'Königsblau',   'fr': 'Bleu Royal',    'it': 'Blu Reale',          'es': 'Azul Real',          'pt': 'Azul Real'},
    'FOREST_GREEN':   {'nl': 'Bosgroen',      'en': 'Forest Green',   'de': 'Waldgrün',     'fr': 'Vert Forêt',    'it': 'Verde Foresta',      'es': 'Verde Bosque',       'pt': 'Verde Floresta'},
    'ICE_BLUE':       {'nl': 'IJsblauw',      'en': 'Ice Blue',       'de': 'Eisblau',      'fr': 'Bleu Glacé',    'it': 'Azzurro Ghiaccio',   'es': 'Azul Hielo',         'pt': 'Azul Gelo'},
    'ROSE_GOLD':      {'nl': 'Roségoud',      'en': 'Rose Gold',      'de': 'Roségold',     'fr': 'Or Rose',       'it': 'Oro Rosa',           'es': 'Oro Rosa',           'pt': 'Ouro Rosa'},
    'SPACE_GREY':     {'nl': 'Space Grey',    'en': 'Space Grey',     'de': 'Space Grau',   'fr': 'Gris Sidéral',  'it': 'Grigio Siderale',    'es': 'Gris Espacial',      'pt': 'Cinzento Sideral'},
    'MIDNIGHT_BLUE':  {'nl': 'Nachtblauw',    'en': 'Midnight Blue',  'de': 'Nachtblau',    'fr': 'Bleu Nuit',     'it': 'Blu Notte',          'es': 'Azul Noche',         'pt': 'Azul Noite'},
    'JET_BLACK':      {'nl': 'Gitzwart',      'en': 'Jet Black',      'de': 'Tiefschwarz',  'fr': 'Noir Jais',     'it': 'Nero Corvino',       'es': 'Negro Azabache',     'pt': 'Preto Azeviche'},
    'STEEL_BLUE':     {'nl': 'Staalblauw',    'en': 'Steel Blue',     'de': 'Stahlblau',    'fr': 'Bleu Acier',    'it': 'Blu Acciaio',        'es': 'Azul Acero',         'pt': 'Azul Aço'},
    'BURNT_ORANGE':   {'nl': 'Gebrand Oranje','en': 'Burnt Orange',   'de': 'Branntorange', 'fr': 'Orange Brûlé',  'it': 'Arancio Bruciato',   'es': 'Naranja Quemado',    'pt': 'Laranja Queimado'},
    'DARK_GREY':      {'nl': 'Donkergrijs',   'en': 'Dark Grey',      'de': 'Dunkelgrau',   'fr': 'Gris Foncé',    'it': 'Grigio Scuro',       'es': 'Gris Oscuro',        'pt': 'Cinzento Escuro'},
    'LIGHT_GREY':     {'nl': 'Lichtgrijs',    'en': 'Light Grey',     'de': 'Hellgrau',     'fr': 'Gris Clair',    'it': 'Grigio Chiaro',      'es': 'Gris Claro',         'pt': 'Cinzento Claro'},
    'DARK_BLUE':      {'nl': 'Donkerblauw',   'en': 'Dark Blue',      'de': 'Dunkelblau',   'fr': 'Bleu Foncé',    'it': 'Blu Scuro',          'es': 'Azul Oscuro',        'pt': 'Azul Escuro'},
    'DARK_GREEN':     {'nl': 'Donkergroen',   'en': 'Dark Green',     'de': 'Dunkelgrün',   'fr': 'Vert Foncé',    'it': 'Verde Scuro',        'es': 'Verde Oscuro',       'pt': 'Verde Escuro'},
    'LIGHT_BLUE':     {'nl': 'Lichtblauw',    'en': 'Light Blue',     'de': 'Hellblau',     'fr': 'Bleu Clair',    'it': 'Azzurro',            'es': 'Azul Claro',         'pt': 'Azul Claro'},
    'LIGHT_GREEN':    {'nl': 'Lichtgroen',    'en': 'Light Green',    'de': 'Hellgrün',     'fr': 'Vert Clair',    'it': 'Verde Chiaro',       'es': 'Verde Claro',        'pt': 'Verde Claro'},
    'LIGHT_WOOD':     {'nl': 'Licht Hout',    'en': 'Light Wood',     'de': 'Helles Holz',  'fr': 'Bois Clair',    'it': 'Legno Chiaro',       'es': 'Madera Clara',       'pt': 'Madeira Clara'},
    'DARK_WOOD':      {'nl': 'Donker Hout',   'en': 'Dark Wood',      'de': 'Dunkles Holz', 'fr': 'Bois Foncé',    'it': 'Legno Scuro',        'es': 'Madera Oscura',      'pt': 'Madeira Escura'},
    'DARK_BROWN':     {'nl': 'Donkerbruin',   'en': 'Dark Brown',     'de': 'Dunkelbraun',  'fr': 'Marron Foncé',  'it': 'Marrone Scuro',      'es': 'Marrón Oscuro',      'pt': 'Castanho Escuro'},
    'LIGHT_BROWN':    {'nl': 'Lichtbruin',    'en': 'Light Brown',    'de': 'Hellbraun',    'fr': 'Marron Clair',  'it': 'Marrone Chiaro',     'es': 'Marrón Claro',       'pt': 'Castanho Claro'},
}

# Composite prefixes per language (for compound words like "donkerblauw", "dunkelblau")
COMPOSITE_PREFIXES = ['donker', 'licht', 'helder', 'dark', 'light', 'bright', 'dunkel', 'hell']

def _resolve_color(canonical_key, lang):
    """Resolve a canonical color key to the output string for the given language."""
    # Check main table first
    if canonical_key in COLOR_OUTPUT:
        return COLOR_OUTPUT[canonical_key].get(lang, COLOR_OUTPUT[canonical_key].get('en', canonical_key))
    # Check multi-word table
    if canonical_key in MULTI_WORD_COLOR_OUTPUT:
        return MULTI_WORD_COLOR_OUTPUT[canonical_key].get(lang, MULTI_WORD_COLOR_OUTPUT[canonical_key].get('en', canonical_key))
    return None
```

### Step 4: Extraction algorithm

The extraction follows a strict cascade. Stop at the first match.

```python
from collections import Counter

def _scan_for_color(text, lang):
    """Scan a text string for color matches. Returns (color_output, match_type) or (None, None)."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None, None
    # Multi-word first
    for pattern, key in MULTI_WORD_COLORS.items():
        if pattern in text_lower:
            output = _resolve_color(key, lang)
            if output:
                return output, 'multi-word'
    # Single-word
    words = re.findall(r'[a-zà-ÿäöüß-]+', text_lower)
    for word in words:
        if word in COLOR_INPUT and COLOR_INPUT[word] is not None:
            output = _resolve_color(COLOR_INPUT[word], lang)
            if output:
                return output, 'single'
    # Composite (donkerblauw, dunkelblau, etc.)
    for word in words:
        for prefix in COMPOSITE_PREFIXES:
            if word.startswith(prefix) and len(word) > len(prefix):
                base = word[len(prefix):]
                if base in COLOR_INPUT and COLOR_INPUT[base] is not None:
                    # Check if this composite is already in MULTI_WORD_COLORS as a single word
                    # (e.g., "dunkelblau" → DARK_BLUE)
                    if word in MULTI_WORD_COLORS:
                        output = _resolve_color(MULTI_WORD_COLORS[word], lang)
                    elif word in COLOR_INPUT:
                        output = _resolve_color(COLOR_INPUT[word], lang)
                    else:
                        # Unknown composite — capitalize the raw input word as fallback
                        output = word.capitalize()
                    if output:
                        return output, 'composite'
    return None, None

def extract_multi_color(title, lang):
    """Detect color combinations like 'Zwart/Wit', 'Black/Gold', 'Schwarz/Weiß'."""
    title_lower = str(title).lower()

    # Pattern: "Color1/Color2" or "Color1 / Color2"
    slash_pattern = re.findall(r'([a-zà-ÿäöüß-]+)\s*/\s*([a-zà-ÿäöüß-]+)', title_lower)
    for c1, c2 in slash_pattern:
        if c1 in COLOR_INPUT and c2 in COLOR_INPUT:
            key1, key2 = COLOR_INPUT[c1], COLOR_INPUT[c2]
            if key1 and key2:
                out1 = _resolve_color(key1, lang)
                out2 = _resolve_color(key2, lang)
                if out1 and out2:
                    return f"{out1}/{out2}", 'title (multi-color)', 'high'

    # Pattern: "Color1 en/and/und/et/e/y Color2"
    and_pattern = re.findall(r'([a-zà-ÿäöüß-]+)\s+(?:en|and|und|et|e|y)\s+([a-zà-ÿäöüß-]+)', title_lower)
    for c1, c2 in and_pattern:
        if c1 in COLOR_INPUT and c2 in COLOR_INPUT:
            key1, key2 = COLOR_INPUT[c1], COLOR_INPUT[c2]
            if key1 and key2:
                out1 = _resolve_color(key1, lang)
                out2 = _resolve_color(key2, lang)
                if out1 and out2:
                    return f"{out1}/{out2}", 'title (multi-color)', 'high'

    return None, None, None

def extract_color(title, description='', product_type='', labels='',
                  google_category='', image_link='', bullet_points='', lang='en'):
    """Extract color using cascade. Returns (color, source, confidence).
    lang parameter controls the output language."""

    # --- PRIORITY 0: Multi-color in title ---
    color, source, conf = extract_multi_color(title, lang)
    if color:
        return color, source, conf

    # --- LAYER 1: Title (high confidence) ---
    color, _ = _scan_for_color(title, lang)
    if color:
        return color, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    color, _ = _scan_for_color(description, lang)
    if color:
        return color, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('google_category', google_category),
        ('labels', labels),
        ('bullet_points', bullet_points),
    ]:
        color, _ = _scan_for_color(source_text, lang)
        if color:
            return color, source_name, 'medium'

    # Check image_link filename (low confidence fallback)
    if pd.notna(image_link) and str(image_link).strip():
        img_filename = str(image_link).split('/')[-1].split('?')[0].lower()
        img_words = re.findall(r'[a-zà-ÿäöüß]+', img_filename)
        for word in img_words:
            if word in COLOR_INPUT and COLOR_INPUT[word] is not None:
                output = _resolve_color(COLOR_INPUT[word], lang)
                if output:
                    return output, 'image_link filename', 'low'

    # --- LAYER 4: No color found ---
    return None, 'none', 'unresolved'
```

### Step 5: Apply to the feed and generate output

```python
# Identify rows with missing color
color_col = next((c for c in df.columns if c in ['color', 'kleur', 'colour', 'g:color', 'farbe', 'couleur', 'colore']), None)

if color_col is None:
    color_col = 'color'
    df[color_col] = ''

# Detect language (when running standalone)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
output_lang = detect_output_language(df, title_col)  # Override if passed by orchestrator

# Track changes for reporting
results = {'filled': 0, 'unresolved': 0, 'already_had': 0, 'details': []}

for idx, row in df.iterrows():
    current_color = str(row[color_col]).strip() if pd.notna(row[color_col]) else ''

    if current_color and current_color.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue

    # Try extraction
    title = row.get('title', '')
    desc = row.get('description', '')
    ptype = row.get('product_type', '')

    color, source, confidence = extract_color(title, desc, ptype, lang=output_lang)

    if color:
        df.at[idx, color_col] = color
        results['filled'] += 1
        results['details'].append({
            'id': row.get('id', idx),
            'title': str(title)[:60],
            'color_found': color,
            'source': source,
            'confidence': confidence
        })
    else:
        results['unresolved'] += 1
        results['details'].append({
            'id': row.get('id', idx),
            'title': str(title)[:60],
            'color_found': None,
            'source': 'none',
            'confidence': 'unresolved'
        })
```

### Step 6: Save as supplemental feed and report

The output is always a **supplemental feed** — not a copy of the full feed. This means:

- **Column 1**: `id` — the product ID (always the Google-recognized attribute name `id`)
- **Column 2**: `color` — the Google-recognized English attribute name, with the extracted or existing color value

Include **ALL products** (not just the ones that were enriched), so the user can see which products still have empty values and need manual attention.

**Important rules for the supplemental feed output:**
- Column names are ALWAYS the English Google Merchant Center attribute names: `id`, `color`, `material`, `gender`, etc.
- Never use localized column names in the output (no `kleur`, `Farbe`, `couleur`, etc.)
- Include all products, even if the attribute value is empty — leave the cell blank
- The output format is always **Excel (.xlsx)** — user can upload to Google Drive where it automatically becomes a Google Sheet, ready to use as supplemental feed.

```python
supplemental = pd.DataFrame({
    'id': df['id'],
    'color': df[color_col]
})

output_path = "/mnt/user-data/outputs/supplemental_feed_color.xlsx"
supplemental.to_excel(output_path, index=False)
```

Present the user with a summary:
```
=== Color Extraction Results ===
Total products:        {total}
Already had color:     {already_had}
Successfully filled:   {filled}  (high confidence: X, medium: Y)
Still empty:           {unresolved}
Fill rate before:      X%
Fill rate after:       Y%
Output language:       {output_lang}
```

Also show a sample of 10-15 filled values so the user can spot-check, and list the products that remain empty.

## Edge cases to handle

- **Multiple colors in title**: e.g., "Sneaker Zwart/Wit" → use "Zwart/Wit" (preserve the combination, output in detected language)
- **Color codes instead of names**: e.g., "#000000" or "RAL 9010" → skip, flag for manual review
- **Brand names that look like colors**: e.g., "Blue Lagoon Parfum" where Blue Lagoon is the product name → use word position heuristics (colors near the end of a title are more likely actual product colors)
- **Abbreviations**: "blk" → "Black", "wht" → "White", "gry" → "Grey" (only when confident)
- **Mixed feeds**: Some products have colors, others don't. Only process empty rows.
- **Color in image_link filename**: Sometimes the color is encoded in the image URL (e.g., `/product-black-xl.jpg`). This is a low-confidence fallback — only use when title and description yield nothing, and flag confidence as "low".

## Important guardrails

- Never overwrite existing color values, even if they look "wrong"
- Always preserve the original file as-is; output to a new file
- Flag uncertainty honestly — a "medium" or "unresolved" tag is more valuable than a wrong color
- If < 30% of empty colors could be filled, warn the user that the feed may need manual enrichment or that titles/descriptions lack color information
- Products in categories where color is irrelevant (e.g., software, digital goods) should be skipped entirely
- **Output language must match feed language.** Never output "Schwarz" in an English feed or "Black" in a German feed.
- **When in doubt, leave empty.** A missing color is better than a wrong color in the wrong language.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `color`
- **Multiple attributes** (when combined with other feed-attribute skills): 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet for use as supplemental feed
- Include ALL products (also empty ones, so the user can see what still needs manual work)
