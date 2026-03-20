---
name: productfeed-material
description: Extract and fill missing material attributes in Google Shopping product feeds. Triggers when user uploads a product feed (CSV, TSV, Excel) and wants to populate empty [material] values. Also triggers when user mentions "material attribute", "materiaal invullen", "feed material", "missing materials in feed", or asks to enrich/complete/fix material data in a product feed. Use this skill even if the user just says "fill in the materials" or "fix my feed materials" with an uploaded file.
---

# Feed Material Extractor

Extract material values from product titles, descriptions, and other available fields to fill empty `material` attributes in a Google Shopping product feed.

## Why this matters

The `[material]` attribute is high-impact in Google Shopping. Missing material data leads to:
- Missed search queries containing material terms (e.g., "leren schoenen", "Lederschuhe", "chaussures en cuir")
- Lower ad relevance for material-specific searches
- Disapproved products in categories where material is required (apparel)
- Poor filtering experience on Shopping tab

## Multi-language support

This skill supports 7 output languages: NL, EN, DE, FR, IT, ES, PT.

**How it works:**
1. The skill recognizes material words from ALL supported languages as input
2. Each recognized material maps to a **canonical material key** (e.g., `COTTON`)
3. The canonical key is then output in the detected feed language (e.g., `COTTON` → "Baumwolle" for DE, "Cotton" for EN, "Katoen" for NL)

**Language detection:** When run standalone, the skill detects the feed language using the same `detect_output_language()` function as the orchestrator. When run via the orchestrator, it receives `output_lang` as a parameter.

## Workflow

### Step 1: Load and inspect the feed

Read the uploaded file (CSV, TSV, ZIP containing TSV, or Excel). The material column may be named any of: `material`, `materiaal`, `Material`, `g:material`, `matériau`, `materiale`. Also locate source columns: `title`/`titel`, `description`/`beschrijving`, `product_type`/`producttype`.

```python
import pandas as pd

# Detect file type and load (handle ZIP containing TSV)
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

Report: total products, how many have empty material, which source columns are available.

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
    scores = {}
    for lang, pattern in LANGUAGE_SIGNALS.items():
        scores[lang] = len(re.findall(pattern, sample))
    scores = {k: v for k, v in scores.items() if v > 0}
    if not scores:
        return 'en'
    return max(scores, key=scores.get)
```

### Step 3: Material vocabulary — multi-language

Same architecture as the color skill: input word → canonical key → localized output.

```python
# Canonical material key → output per language
MATERIAL_OUTPUT = {
    # Textiles
    'COTTON':      {'nl': 'Katoen',     'en': 'Cotton',       'de': 'Baumwolle',    'fr': 'Coton',         'it': 'Cotone',         'es': 'Algodón',       'pt': 'Algodão'},
    'POLYESTER':   {'nl': 'Polyester',   'en': 'Polyester',    'de': 'Polyester',    'fr': 'Polyester',     'it': 'Poliestere',     'es': 'Poliéster',     'pt': 'Poliéster'},
    'NYLON':       {'nl': 'Nylon',       'en': 'Nylon',        'de': 'Nylon',        'fr': 'Nylon',         'it': 'Nylon',          'es': 'Nylon',         'pt': 'Nylon'},
    'SILK':        {'nl': 'Zijde',       'en': 'Silk',         'de': 'Seide',        'fr': 'Soie',          'it': 'Seta',           'es': 'Seda',          'pt': 'Seda'},
    'WOOL':        {'nl': 'Wol',         'en': 'Wool',         'de': 'Wolle',        'fr': 'Laine',         'it': 'Lana',           'es': 'Lana',          'pt': 'Lã'},
    'LINEN':       {'nl': 'Linnen',      'en': 'Linen',        'de': 'Leinen',       'fr': 'Lin',           'it': 'Lino',           'es': 'Lino',          'pt': 'Linho'},
    'DENIM':       {'nl': 'Denim',       'en': 'Denim',        'de': 'Denim',        'fr': 'Jean',          'it': 'Denim',          'es': 'Mezclilla',     'pt': 'Denim'},
    'FLEECE':      {'nl': 'Fleece',      'en': 'Fleece',       'de': 'Fleece',       'fr': 'Polaire',       'it': 'Pile',           'es': 'Polar',         'pt': 'Fleece'},
    'VISCOSE':     {'nl': 'Viscose',     'en': 'Viscose',      'de': 'Viskose',      'fr': 'Viscose',       'it': 'Viscosa',        'es': 'Viscosa',       'pt': 'Viscose'},
    'ELASTANE':    {'nl': 'Elastaan',    'en': 'Elastane',     'de': 'Elasthan',     'fr': 'Élasthanne',    'it': 'Elastan',        'es': 'Elastano',      'pt': 'Elastano'},
    'CASHMERE':    {'nl': 'Kasjmier',    'en': 'Cashmere',     'de': 'Kaschmir',     'fr': 'Cachemire',     'it': 'Cashmere',       'es': 'Cachemira',     'pt': 'Caxemira'},
    'VELVET':      {'nl': 'Fluweel',     'en': 'Velvet',       'de': 'Samt',         'fr': 'Velours',       'it': 'Velluto',        'es': 'Terciopelo',    'pt': 'Veludo'},
    'SATIN':       {'nl': 'Satijn',      'en': 'Satin',        'de': 'Satin',        'fr': 'Satin',         'it': 'Raso',           'es': 'Satén',         'pt': 'Cetim'},
    'CHIFFON':     {'nl': 'Chiffon',     'en': 'Chiffon',      'de': 'Chiffon',      'fr': 'Mousseline',    'it': 'Chiffon',        'es': 'Gasa',          'pt': 'Chiffon'},
    'JERSEY':      {'nl': 'Jersey',      'en': 'Jersey',       'de': 'Jersey',       'fr': 'Jersey',        'it': 'Jersey',         'es': 'Jersey',        'pt': 'Jersey'},
    'TWEED':       {'nl': 'Tweed',       'en': 'Tweed',        'de': 'Tweed',        'fr': 'Tweed',         'it': 'Tweed',          'es': 'Tweed',         'pt': 'Tweed'},
    'CANVAS':      {'nl': 'Canvas',      'en': 'Canvas',       'de': 'Canvas',       'fr': 'Toile',         'it': 'Tela',           'es': 'Lona',          'pt': 'Lona'},
    'MESH':        {'nl': 'Mesh',        'en': 'Mesh',         'de': 'Mesh',         'fr': 'Maille',        'it': 'Rete',           'es': 'Malla',         'pt': 'Malha'},
    'MICROFIBRE':  {'nl': 'Microvezel',  'en': 'Microfibre',   'de': 'Mikrofaser',   'fr': 'Microfibre',    'it': 'Microfibra',     'es': 'Microfibra',    'pt': 'Microfibra'},
    'CHENILLE':    {'nl': 'Chenille',    'en': 'Chenille',     'de': 'Chenille',     'fr': 'Chenille',      'it': 'Ciniglia',       'es': 'Chenilla',      'pt': 'Chenille'},
    'BOUCLE':      {'nl': 'Bouclé',      'en': 'Boucle',       'de': 'Bouclé',       'fr': 'Bouclé',        'it': 'Bouclé',         'es': 'Bouclé',        'pt': 'Bouclé'},
    'ACRYLIC':     {'nl': 'Acryl',       'en': 'Acrylic',      'de': 'Acryl',        'fr': 'Acrylique',     'it': 'Acrilico',       'es': 'Acrílico',      'pt': 'Acrílico'},
    'POLYPROPYLENE': {'nl': 'Polypropyleen', 'en': 'Polypropylene', 'de': 'Polypropylen', 'fr': 'Polypropylène', 'it': 'Polipropilene', 'es': 'Polipropileno', 'pt': 'Polipropileno'},
    'MODAL':       {'nl': 'Modal',       'en': 'Modal',        'de': 'Modal',        'fr': 'Modal',         'it': 'Modal',          'es': 'Modal',         'pt': 'Modal'},
    'GORE_TEX':    {'nl': 'Gore-Tex',    'en': 'Gore-Tex',     'de': 'Gore-Tex',     'fr': 'Gore-Tex',      'it': 'Gore-Tex',       'es': 'Gore-Tex',      'pt': 'Gore-Tex'},
    # Leather & skins
    'LEATHER':     {'nl': 'Leer',        'en': 'Leather',      'de': 'Leder',        'fr': 'Cuir',          'it': 'Pelle',          'es': 'Cuero',         'pt': 'Couro'},
    'FAUX_LEATHER': {'nl': 'Kunstleer',  'en': 'Faux Leather', 'de': 'Kunstleder',   'fr': 'Simili Cuir',   'it': 'Ecopelle',       'es': 'Polipiel',      'pt': 'Couro Sintético'},
    'SUEDE':       {'nl': 'Suède',       'en': 'Suede',        'de': 'Wildleder',    'fr': 'Daim',          'it': 'Scamosciato',    'es': 'Ante',          'pt': 'Camurça'},
    'NUBUCK':      {'nl': 'Nubuck',      'en': 'Nubuck',       'de': 'Nubuk',        'fr': 'Nubuck',        'it': 'Nabuk',          'es': 'Nobuk',         'pt': 'Nubuck'},
    'SHEEPSKIN':   {'nl': 'Schapenvacht','en': 'Sheepskin',     'de': 'Schaffell',    'fr': 'Peau de Mouton','it': 'Pelle di Pecora','es': 'Piel de Oveja', 'pt': 'Pele de Ovelha'},
    # Metals
    'STAINLESS_STEEL': {'nl': 'RVS',     'en': 'Stainless Steel', 'de': 'Edelstahl', 'fr': 'Acier Inoxydable', 'it': 'Acciaio Inox', 'es': 'Acero Inoxidable', 'pt': 'Aço Inoxidável'},
    'STEEL':       {'nl': 'Staal',       'en': 'Steel',        'de': 'Stahl',        'fr': 'Acier',         'it': 'Acciaio',        'es': 'Acero',         'pt': 'Aço'},
    'ALUMINIUM':   {'nl': 'Aluminium',   'en': 'Aluminium',    'de': 'Aluminium',    'fr': 'Aluminium',     'it': 'Alluminio',      'es': 'Aluminio',      'pt': 'Alumínio'},
    'BRASS':       {'nl': 'Messing',     'en': 'Brass',        'de': 'Messing',      'fr': 'Laiton',        'it': 'Ottone',         'es': 'Latón',         'pt': 'Latão'},
    'COPPER':      {'nl': 'Koper',       'en': 'Copper',       'de': 'Kupfer',       'fr': 'Cuivre',        'it': 'Rame',           'es': 'Cobre',         'pt': 'Cobre'},
    'IRON':        {'nl': 'IJzer',       'en': 'Iron',         'de': 'Eisen',        'fr': 'Fer',           'it': 'Ferro',          'es': 'Hierro',        'pt': 'Ferro'},
    'CHROME':      {'nl': 'Chroom',      'en': 'Chrome',       'de': 'Chrom',        'fr': 'Chrome',        'it': 'Cromo',          'es': 'Cromo',         'pt': 'Cromo'},
    'TITANIUM':    {'nl': 'Titanium',    'en': 'Titanium',     'de': 'Titan',        'fr': 'Titane',        'it': 'Titanio',        'es': 'Titanio',       'pt': 'Titânio'},
    'NICKEL':      {'nl': 'Nikkel',      'en': 'Nickel',       'de': 'Nickel',       'fr': 'Nickel',        'it': 'Nichel',         'es': 'Níquel',        'pt': 'Níquel'},
    # Wood
    'WOOD':        {'nl': 'Hout',        'en': 'Wood',         'de': 'Holz',         'fr': 'Bois',          'it': 'Legno',          'es': 'Madera',        'pt': 'Madeira'},
    'BAMBOO':      {'nl': 'Bamboe',      'en': 'Bamboo',       'de': 'Bambus',       'fr': 'Bambou',        'it': 'Bambù',          'es': 'Bambú',         'pt': 'Bambu'},
    'OAK':         {'nl': 'Eikenhout',   'en': 'Oak',          'de': 'Eiche',        'fr': 'Chêne',         'it': 'Rovere',         'es': 'Roble',         'pt': 'Carvalho'},
    'WALNUT':      {'nl': 'Walnoothout', 'en': 'Walnut',       'de': 'Walnuss',      'fr': 'Noyer',         'it': 'Noce',           'es': 'Nogal',         'pt': 'Nogueira'},
    'TEAK':        {'nl': 'Teakhout',    'en': 'Teak',         'de': 'Teak',         'fr': 'Teck',          'it': 'Teak',           'es': 'Teca',          'pt': 'Teca'},
    'PINE':        {'nl': 'Grenenhout',  'en': 'Pine',         'de': 'Kiefer',       'fr': 'Pin',           'it': 'Pino',           'es': 'Pino',          'pt': 'Pinho'},
    'BEECH':       {'nl': 'Beukenhout',  'en': 'Beech',        'de': 'Buche',        'fr': 'Hêtre',         'it': 'Faggio',         'es': 'Haya',          'pt': 'Faia'},
    'MDF':         {'nl': 'MDF',         'en': 'MDF',          'de': 'MDF',          'fr': 'MDF',           'it': 'MDF',            'es': 'MDF',           'pt': 'MDF'},
    'PLYWOOD':     {'nl': 'Multiplex',   'en': 'Plywood',      'de': 'Sperrholz',    'fr': 'Contreplaqué',  'it': 'Compensato',     'es': 'Contrachapado', 'pt': 'Contraplacado'},
    'BIRCH':       {'nl': 'Berkenhout',  'en': 'Birch',        'de': 'Birke',        'fr': 'Bouleau',       'it': 'Betulla',        'es': 'Abedul',        'pt': 'Bétula'},
    'ASH':         {'nl': 'Essenhout',   'en': 'Ash',          'de': 'Esche',        'fr': 'Frêne',         'it': 'Frassino',       'es': 'Fresno',        'pt': 'Freixo'},
    'MAHOGANY':    {'nl': 'Mahoniehout', 'en': 'Mahogany',     'de': 'Mahagoni',     'fr': 'Acajou',        'it': 'Mogano',         'es': 'Caoba',         'pt': 'Mogno'},
    'MANGO_WOOD':  {'nl': 'Mangohout',   'en': 'Mango Wood',   'de': 'Mangobaum',    'fr': 'Bois de Manguier','it': 'Legno di Mango','es': 'Madera de Mango','pt': 'Madeira de Manga'},
    'CEDAR':       {'nl': 'Cederhout',   'en': 'Cedar',        'de': 'Zedernholz',   'fr': 'Cèdre',         'it': 'Cedro',          'es': 'Cedro',         'pt': 'Cedro'},
    'RUBBERWOOD':  {'nl': 'Rubberhout',  'en': 'Rubberwood',   'de': 'Gummibaumholz','fr': 'Hévéa',         'it': 'Legno di Hevea', 'es': 'Madera de Caucho','pt': 'Madeira de Seringueira'},
    # Glass & ceramics
    'GLASS':       {'nl': 'Glas',        'en': 'Glass',        'de': 'Glas',         'fr': 'Verre',         'it': 'Vetro',          'es': 'Vidrio',        'pt': 'Vidro'},
    'CRYSTAL':     {'nl': 'Kristalglas', 'en': 'Crystal Glass', 'de': 'Kristallglas','fr': 'Cristal',       'it': 'Cristallo',      'es': 'Cristal',       'pt': 'Cristal'},
    'CERAMIC':     {'nl': 'Keramiek',    'en': 'Ceramic',      'de': 'Keramik',      'fr': 'Céramique',     'it': 'Ceramica',       'es': 'Cerámica',      'pt': 'Cerâmica'},
    'PORCELAIN':   {'nl': 'Porselein',   'en': 'Porcelain',    'de': 'Porzellan',    'fr': 'Porcelaine',    'it': 'Porcellana',     'es': 'Porcelana',     'pt': 'Porcelana'},
    'EARTHENWARE': {'nl': 'Aardewerk',   'en': 'Earthenware',  'de': 'Steingut',     'fr': 'Faïence',       'it': 'Terracotta',     'es': 'Loza',          'pt': 'Faiança'},
    # Plastics & synthetics
    'PLASTIC':     {'nl': 'Kunststof',   'en': 'Plastic',      'de': 'Kunststoff',   'fr': 'Plastique',     'it': 'Plastica',       'es': 'Plástico',      'pt': 'Plástico'},
    'SILICONE':    {'nl': 'Siliconen',   'en': 'Silicone',     'de': 'Silikon',      'fr': 'Silicone',      'it': 'Silicone',       'es': 'Silicona',      'pt': 'Silicone'},
    'RUBBER':      {'nl': 'Rubber',      'en': 'Rubber',       'de': 'Gummi',        'fr': 'Caoutchouc',    'it': 'Gomma',          'es': 'Goma',          'pt': 'Borracha'},
    'PVC':         {'nl': 'PVC',         'en': 'PVC',          'de': 'PVC',          'fr': 'PVC',           'it': 'PVC',            'es': 'PVC',           'pt': 'PVC'},
    'ABS':         {'nl': 'ABS',         'en': 'ABS',          'de': 'ABS',          'fr': 'ABS',           'it': 'ABS',            'es': 'ABS',           'pt': 'ABS'},
    'POLYCARBONATE': {'nl': 'Polycarbonaat', 'en': 'Polycarbonate', 'de': 'Polycarbonat', 'fr': 'Polycarbonate', 'it': 'Policarbonato', 'es': 'Policarbonato', 'pt': 'Policarbonato'},
    'FOAM':        {'nl': 'Schuim',      'en': 'Foam',         'de': 'Schaum',       'fr': 'Mousse',        'it': 'Schiuma',        'es': 'Espuma',        'pt': 'Espuma'},
    # Natural materials
    'CORK':        {'nl': 'Kurk',        'en': 'Cork',         'de': 'Kork',         'fr': 'Liège',         'it': 'Sughero',        'es': 'Corcho',        'pt': 'Cortiça'},
    'JUTE':        {'nl': 'Jute',        'en': 'Jute',         'de': 'Jute',         'fr': 'Jute',          'it': 'Juta',           'es': 'Yute',          'pt': 'Juta'},
    'SISAL':       {'nl': 'Sisal',       'en': 'Sisal',        'de': 'Sisal',        'fr': 'Sisal',         'it': 'Sisal',          'es': 'Sisal',         'pt': 'Sisal'},
    'RATTAN':      {'nl': 'Rattan',      'en': 'Rattan',       'de': 'Rattan',       'fr': 'Rotin',         'it': 'Rattan',         'es': 'Ratán',         'pt': 'Vime'},
    'MARBLE':      {'nl': 'Marmer',      'en': 'Marble',       'de': 'Marmor',       'fr': 'Marbre',        'it': 'Marmo',          'es': 'Mármol',        'pt': 'Mármore'},
    'GRANITE':     {'nl': 'Graniet',     'en': 'Granite',      'de': 'Granit',       'fr': 'Granit',        'it': 'Granito',        'es': 'Granito',       'pt': 'Granito'},
    'STONE':       {'nl': 'Steen',       'en': 'Stone',        'de': 'Stein',        'fr': 'Pierre',        'it': 'Pietra',         'es': 'Piedra',        'pt': 'Pedra'},
    'CONCRETE':    {'nl': 'Beton',       'en': 'Concrete',     'de': 'Beton',        'fr': 'Béton',         'it': 'Cemento',        'es': 'Hormigón',      'pt': 'Betão'},
    'TERRAZZO':    {'nl': 'Terrazzo',    'en': 'Terrazzo',     'de': 'Terrazzo',     'fr': 'Terrazzo',      'it': 'Terrazzo',       'es': 'Terrazo',       'pt': 'Terraço'},
    'PAPER':       {'nl': 'Papier',      'en': 'Paper',        'de': 'Papier',       'fr': 'Papier',        'it': 'Carta',          'es': 'Papel',         'pt': 'Papel'},
    'CARDBOARD':   {'nl': 'Karton',      'en': 'Cardboard',    'de': 'Karton',       'fr': 'Carton',        'it': 'Cartone',        'es': 'Cartón',        'pt': 'Cartão'},
    # Composites
    'CARBON':      {'nl': 'Carbon',      'en': 'Carbon Fibre', 'de': 'Carbon',       'fr': 'Fibre de Carbone', 'it': 'Fibra di Carbonio', 'es': 'Fibra de Carbono', 'pt': 'Fibra de Carbono'},
    'FIBREGLASS':  {'nl': 'Glasvezel',   'en': 'Fibreglass',   'de': 'Glasfaser',    'fr': 'Fibre de Verre','it': 'Fibra di Vetro', 'es': 'Fibra de Vidrio','pt': 'Fibra de Vidro'},
    # Special
    'SOLID_WOOD':  {'nl': 'Massief Hout','en': 'Solid Wood',   'de': 'Massivholz',   'fr': 'Bois Massif',   'it': 'Legno Massello', 'es': 'Madera Maciza', 'pt': 'Madeira Maciça'},
    'SOLID_OAK':   {'nl': 'Massief Eiken','en': 'Solid Oak',   'de': 'Massiveiche',  'fr': 'Chêne Massif',  'it': 'Rovere Massello','es': 'Roble Macizo',  'pt': 'Carvalho Maciço'},
    'TEMPERED_GLASS': {'nl': 'Gehard Glas','en': 'Tempered Glass','de': 'Hartglas',   'fr': 'Verre Trempé',  'it': 'Vetro Temperato','es': 'Vidrio Templado','pt': 'Vidro Temperado'},
    'MEMORY_FOAM': {'nl': 'Traagschuim', 'en': 'Memory Foam',  'de': 'Memory Foam',  'fr': 'Mousse à Mémoire','it': 'Memory Foam', 'es': 'Espuma Viscoelástica', 'pt': 'Espuma Viscoelástica'},
    'POCKET_SPRUNG': {'nl': 'Pocketvering','en': 'Pocket Sprung','de': 'Taschenfederkern','fr': 'Ressorts Ensachés','it': 'Molle Insacchettate','es': 'Muelles Ensacados','pt': 'Molas Ensacadas'},
    'CAST_IRON':   {'nl': 'Gietijzer',   'en': 'Cast Iron',    'de': 'Gusseisen',    'fr': 'Fonte',         'it': 'Ghisa',          'es': 'Hierro Fundido','pt': 'Ferro Fundido'},
    'WROUGHT_IRON': {'nl': 'Smeedijzer', 'en': 'Wrought Iron', 'de': 'Schmiedeeisen','fr': 'Fer Forgé',     'it': 'Ferro Battuto',  'es': 'Hierro Forjado','pt': 'Ferro Forjado'},
}

# Input word → canonical key (all supported languages)
MATERIAL_INPUT = {
    # NL input
    'katoen': 'COTTON', 'katoenen': 'COTTON', 'polyester': 'POLYESTER', 'nylon': 'NYLON',
    'zijde': 'SILK', 'zijden': 'SILK', 'wol': 'WOOL', 'wollen': 'WOOL',
    'linnen': 'LINEN', 'denim': 'DENIM', 'jeans': 'DENIM', 'fleece': 'FLEECE',
    'viscose': 'VISCOSE', 'elastaan': 'ELASTANE', 'modal': 'MODAL',
    'kasjmier': 'CASHMERE', 'velours': 'VELVET', 'fluweel': 'VELVET', 'fluwelen': 'VELVET',
    'satijn': 'SATIN', 'satijnen': 'SATIN', 'chiffon': 'CHIFFON', 'jersey': 'JERSEY',
    'tweed': 'TWEED', 'canvas': 'CANVAS', 'mesh': 'MESH', 'microvezel': 'MICROFIBRE',
    'chenille': 'CHENILLE', 'bouclé': 'BOUCLE', 'boucle': 'BOUCLE',
    'acryl': 'ACRYLIC', 'polypropyleen': 'POLYPROPYLENE',
    'gore-tex': 'GORE_TEX', 'goretex': 'GORE_TEX',
    'leer': 'LEATHER', 'leren': 'LEATHER', 'kunstleer': 'FAUX_LEATHER', 'kunstleren': 'FAUX_LEATHER',
    'suède': 'SUEDE', 'nubuck': 'NUBUCK', 'schapenvacht': 'SHEEPSKIN',
    'rvs': 'STAINLESS_STEEL', 'edelstaal': 'STAINLESS_STEEL',
    'staal': 'STEEL', 'stalen': 'STEEL', 'aluminium': 'ALUMINIUM',
    'messing': 'BRASS', 'koper': 'COPPER', 'koperen': 'COPPER',
    'titanium': 'TITANIUM', 'ijzer': 'IRON', 'chroom': 'CHROME', 'nikkel': 'NICKEL',
    'hout': 'WOOD', 'houten': 'WOOD', 'bamboe': 'BAMBOO',
    'eiken': 'OAK', 'eikenhout': 'OAK', 'walnoot': 'WALNUT', 'walnoothout': 'WALNUT',
    'teak': 'TEAK', 'teakhout': 'TEAK', 'grenen': 'PINE', 'grenenhout': 'PINE',
    'beuken': 'BEECH', 'beukenhout': 'BEECH', 'mdf': 'MDF',
    'multiplex': 'PLYWOOD', 'berken': 'BIRCH', 'berkenhout': 'BIRCH',
    'essen': 'ASH', 'essenhout': 'ASH', 'mahonie': 'MAHOGANY', 'ceder': 'CEDAR',
    'glas': 'GLASS', 'glazen': 'GLASS', 'kristalglas': 'CRYSTAL', 'kristal': 'CRYSTAL',
    'keramiek': 'CERAMIC', 'keramisch': 'CERAMIC', 'keramische': 'CERAMIC',
    'porselein': 'PORCELAIN', 'aardewerk': 'EARTHENWARE',
    'kunststof': 'PLASTIC', 'plastic': 'PLASTIC',
    'siliconen': 'SILICONE', 'rubber': 'RUBBER', 'pvc': 'PVC', 'abs': 'ABS',
    'polycarbonaat': 'POLYCARBONATE', 'schuim': 'FOAM',
    'kurk': 'CORK', 'jute': 'JUTE', 'sisal': 'SISAL',
    'rattan': 'RATTAN', 'rotan': 'RATTAN', 'riet': 'RATTAN',
    'marmer': 'MARBLE', 'marmeren': 'MARBLE', 'graniet': 'GRANITE',
    'steen': 'STONE', 'stenen': 'STONE', 'beton': 'CONCRETE',
    'papier': 'PAPER', 'karton': 'CARDBOARD',
    'carbon': 'CARBON', 'koolstofvezel': 'CARBON', 'glasvezel': 'FIBREGLASS',
    'gietijzer': 'CAST_IRON', 'smeedijzer': 'WROUGHT_IRON',
    'traagschuim': 'MEMORY_FOAM', 'pocketvering': 'POCKET_SPRUNG',
    'terrazzo': 'TERRAZZO',
    # EN input
    'cotton': 'COTTON', 'silk': 'SILK', 'wool': 'WOOL', 'linen': 'LINEN',
    'rayon': 'VISCOSE', 'elastane': 'ELASTANE', 'spandex': 'ELASTANE', 'lycra': 'ELASTANE',
    'cashmere': 'CASHMERE', 'cashemere': 'CASHMERE', 'velvet': 'VELVET', 'satin': 'SATIN',
    'microfiber': 'MICROFIBRE', 'microfibre': 'MICROFIBRE',
    'acrylic': 'ACRYLIC', 'polypropylene': 'POLYPROPYLENE',
    'leather': 'LEATHER', 'suede': 'SUEDE', 'nubuck': 'NUBUCK', 'sheepskin': 'SHEEPSKIN',
    'steel': 'STEEL', 'aluminum': 'ALUMINIUM',
    'brass': 'BRASS', 'copper': 'COPPER', 'iron': 'IRON', 'chrome': 'CHROME', 'nickel': 'NICKEL',
    'wood': 'WOOD', 'wooden': 'WOOD', 'bamboo': 'BAMBOO',
    'oak': 'OAK', 'walnut': 'WALNUT', 'pine': 'PINE', 'beech': 'BEECH',
    'plywood': 'PLYWOOD', 'birch': 'BIRCH', 'ash': 'ASH', 'mahogany': 'MAHOGANY',
    'mango': 'MANGO_WOOD', 'cedar': 'CEDAR', 'rubberwood': 'RUBBERWOOD',
    'glass': 'GLASS', 'ceramic': 'CERAMIC', 'porcelain': 'PORCELAIN',
    'earthenware': 'EARTHENWARE', 'pottery': 'EARTHENWARE',
    'silicone': 'SILICONE', 'polycarbonate': 'POLYCARBONATE', 'foam': 'FOAM',
    'cork': 'CORK', 'marble': 'MARBLE', 'granite': 'GRANITE',
    'stone': 'STONE', 'concrete': 'CONCRETE',
    'paper': 'PAPER', 'cardboard': 'CARDBOARD',
    'fibreglass': 'FIBREGLASS', 'fiberglass': 'FIBREGLASS',
    # DE input
    'baumwolle': 'COTTON', 'seide': 'SILK', 'wolle': 'WOOL', 'leinen': 'LINEN',
    'viskose': 'VISCOSE', 'elasthan': 'ELASTANE', 'kaschmir': 'CASHMERE',
    'samt': 'VELVET', 'mikrofaser': 'MICROFIBRE',
    'leder': 'LEATHER', 'kunstleder': 'FAUX_LEATHER', 'wildleder': 'SUEDE',
    'nubuk': 'NUBUCK', 'schaffell': 'SHEEPSKIN',
    'edelstahl': 'STAINLESS_STEEL', 'stahl': 'STEEL', 'kupfer': 'COPPER',
    'eisen': 'IRON', 'chrom': 'CHROME', 'titan': 'TITANIUM',
    'holz': 'WOOD', 'hölzern': 'WOOD', 'bambus': 'BAMBOO',
    'eiche': 'OAK', 'walnuss': 'WALNUT', 'kiefer': 'PINE', 'buche': 'BEECH',
    'sperrholz': 'PLYWOOD', 'birke': 'BIRCH', 'esche': 'ASH', 'mahagoni': 'MAHOGANY',
    'zedernholz': 'CEDAR', 'mangobaum': 'MANGO_WOOD',
    'keramik': 'CERAMIC', 'porzellan': 'PORCELAIN', 'steingut': 'EARTHENWARE',
    'kunststoff': 'PLASTIC', 'silikon': 'SILICONE', 'gummi': 'RUBBER',
    'polycarbonat': 'POLYCARBONATE', 'polypropylen': 'POLYPROPYLENE',
    'kork': 'CORK', 'marmor': 'MARBLE', 'granit': 'GRANITE',
    'stein': 'STONE', 'beton': 'CONCRETE', 'papier': 'PAPER', 'karton': 'CARDBOARD',
    'glasfaser': 'FIBREGLASS', 'gusseisen': 'CAST_IRON', 'schmiedeeisen': 'WROUGHT_IRON',
    'schaum': 'FOAM', 'taschenfederkern': 'POCKET_SPRUNG',
    # FR input
    'coton': 'COTTON', 'soie': 'SILK', 'laine': 'WOOL', 'lin': 'LINEN',
    'polaire': 'FLEECE', 'élasthanne': 'ELASTANE', 'cachemire': 'CASHMERE',
    'velours': 'VELVET', 'mousseline': 'CHIFFON', 'toile': 'CANVAS',
    'maille': 'MESH', 'acrylique': 'ACRYLIC', 'polypropylène': 'POLYPROPYLENE',
    'cuir': 'LEATHER', 'daim': 'SUEDE',
    'acier': 'STEEL', 'laiton': 'BRASS', 'cuivre': 'COPPER', 'fer': 'IRON',
    'titane': 'TITANIUM',
    'bois': 'WOOD', 'bambou': 'BAMBOO', 'chêne': 'OAK', 'noyer': 'WALNUT',
    'pin': 'PINE', 'hêtre': 'BEECH', 'contreplaqué': 'PLYWOOD',
    'bouleau': 'BIRCH', 'frêne': 'ASH', 'acajou': 'MAHOGANY', 'cèdre': 'CEDAR',
    'verre': 'GLASS', 'céramique': 'CERAMIC', 'porcelaine': 'PORCELAIN',
    'faïence': 'EARTHENWARE', 'plastique': 'PLASTIC', 'caoutchouc': 'RUBBER',
    'liège': 'CORK', 'rotin': 'RATTAN', 'marbre': 'MARBLE', 'pierre': 'STONE',
    'béton': 'CONCRETE', 'carton': 'CARDBOARD',
    'fonte': 'CAST_IRON', 'mousse': 'FOAM',
    # IT input
    'cotone': 'COTTON', 'seta': 'SILK', 'lana': 'WOOL', 'lino': 'LINEN',
    'pile': 'FLEECE', 'velluto': 'VELVET', 'raso': 'SATIN',
    'pelle': 'LEATHER', 'ecopelle': 'FAUX_LEATHER', 'scamosciato': 'SUEDE',
    'acciaio': 'STEEL', 'ottone': 'BRASS', 'rame': 'COPPER', 'ferro': 'IRON',
    'titanio': 'TITANIUM', 'nichel': 'NICKEL', 'cromo': 'CHROME',
    'legno': 'WOOD', 'bambù': 'BAMBOO', 'rovere': 'OAK', 'noce': 'WALNUT',
    'pino': 'PINE', 'faggio': 'BEECH', 'compensato': 'PLYWOOD', 'betulla': 'BIRCH',
    'frassino': 'ASH', 'mogano': 'MAHOGANY', 'cedro': 'CEDAR',
    'vetro': 'GLASS', 'ceramica': 'CERAMIC', 'porcellana': 'PORCELAIN',
    'plastica': 'PLASTIC', 'gomma': 'RUBBER', 'silicone': 'SILICONE',
    'sughero': 'CORK', 'marmo': 'MARBLE', 'granito': 'GRANITE', 'pietra': 'STONE',
    'cemento': 'CONCRETE', 'carta': 'PAPER', 'cartone': 'CARDBOARD',
    'ghisa': 'CAST_IRON',
    # ES input
    'algodón': 'COTTON', 'seda': 'SILK', 'lana': 'WOOL', 'mezclilla': 'DENIM',
    'polar': 'FLEECE', 'terciopelo': 'VELVET', 'satén': 'SATIN', 'gasa': 'CHIFFON',
    'lona': 'CANVAS', 'malla': 'MESH', 'acrílico': 'ACRYLIC',
    'cuero': 'LEATHER', 'polipiel': 'FAUX_LEATHER', 'ante': 'SUEDE',
    'acero': 'STEEL', 'latón': 'BRASS', 'cobre': 'COPPER', 'hierro': 'IRON',
    'madera': 'WOOD', 'bambú': 'BAMBOO', 'roble': 'OAK', 'nogal': 'WALNUT',
    'haya': 'BEECH', 'contrachapado': 'PLYWOOD', 'abedul': 'BIRCH', 'fresno': 'ASH',
    'caoba': 'MAHOGANY', 'teca': 'TEAK',
    'vidrio': 'GLASS', 'cerámica': 'CERAMIC', 'porcelana': 'PORCELAIN',
    'plástico': 'PLASTIC', 'goma': 'RUBBER', 'silicona': 'SILICONE',
    'corcho': 'CORK', 'ratán': 'RATTAN', 'mármol': 'MARBLE',
    'piedra': 'STONE', 'hormigón': 'CONCRETE',
    # PT input
    'algodão': 'COTTON', 'lã': 'WOOL', 'linho': 'LINEN',
    'veludo': 'VELVET', 'cetim': 'SATIN',
    'couro': 'LEATHER', 'camurça': 'SUEDE',
    'aço': 'STEEL', 'cobre': 'COPPER',
    'madeira': 'WOOD', 'bambu': 'BAMBOO', 'carvalho': 'OAK', 'nogueira': 'WALNUT',
    'pinho': 'PINE', 'faia': 'BEECH', 'bétula': 'BIRCH', 'freixo': 'ASH',
    'mogno': 'MAHOGANY',
    'vidro': 'GLASS', 'cerâmica': 'CERAMIC',
    'plástico': 'PLASTIC', 'borracha': 'RUBBER',
    'cortiça': 'CORK', 'vime': 'RATTAN', 'mármore': 'MARBLE',
    'pedra': 'STONE', 'betão': 'CONCRETE',
}

# Multi-word materials → canonical key
MULTI_WORD_MATERIALS = {
    'stainless steel': 'STAINLESS_STEEL', 'acier inoxydable': 'STAINLESS_STEEL',
    'acciaio inossidabile': 'STAINLESS_STEEL', 'acciaio inox': 'STAINLESS_STEEL',
    'acero inoxidable': 'STAINLESS_STEEL', 'aço inoxidável': 'STAINLESS_STEEL',
    'faux leather': 'FAUX_LEATHER', 'vegan leather': 'FAUX_LEATHER',
    'pu leather': 'FAUX_LEATHER', 'pu-leather': 'FAUX_LEATHER',
    'pu leer': 'FAUX_LEATHER', 'pu-leer': 'FAUX_LEATHER',
    'simili cuir': 'FAUX_LEATHER',
    'crystal glass': 'CRYSTAL', 'kristallglas': 'CRYSTAL',
    'solid wood': 'SOLID_WOOD', 'massief hout': 'SOLID_WOOD', 'massivholz': 'SOLID_WOOD',
    'bois massif': 'SOLID_WOOD', 'legno massello': 'SOLID_WOOD', 'madera maciza': 'SOLID_WOOD',
    'solid oak': 'SOLID_OAK', 'massief eiken': 'SOLID_OAK', 'massiveiche': 'SOLID_OAK',
    'chêne massif': 'SOLID_OAK',
    'tempered glass': 'TEMPERED_GLASS', 'gehard glas': 'TEMPERED_GLASS', 'hartglas': 'TEMPERED_GLASS',
    'verre trempé': 'TEMPERED_GLASS', 'vetro temperato': 'TEMPERED_GLASS',
    'safety glass': 'TEMPERED_GLASS',
    'memory foam': 'MEMORY_FOAM', 'traagschuim': 'MEMORY_FOAM',
    'mousse à mémoire': 'MEMORY_FOAM',
    'pocket sprung': 'POCKET_SPRUNG', 'taschenfederkern': 'POCKET_SPRUNG',
    'ressorts ensachés': 'POCKET_SPRUNG',
    'mango wood': 'MANGO_WOOD', 'mangohout': 'MANGO_WOOD',
    'bois de manguier': 'MANGO_WOOD',
    'cast iron': 'CAST_IRON', 'gietijzer': 'CAST_IRON', 'gusseisen': 'CAST_IRON',
    'fonte': 'CAST_IRON',
    'wrought iron': 'WROUGHT_IRON', 'smeedijzer': 'WROUGHT_IRON',
    'schmiedeeisen': 'WROUGHT_IRON', 'fer forgé': 'WROUGHT_IRON',
    'carbon fiber': 'CARBON', 'carbon fibre': 'CARBON', 'koolstofvezel': 'CARBON',
    'fibre de carbone': 'CARBON',
    'recycled plastic': 'PLASTIC',
    'engineered wood': 'WOOD',
}

def _resolve_material(canonical_key, lang):
    """Resolve a canonical material key to the output string for the given language."""
    if canonical_key in MATERIAL_OUTPUT:
        return MATERIAL_OUTPUT[canonical_key].get(lang, MATERIAL_OUTPUT[canonical_key].get('en', canonical_key))
    return None
```

### Step 4: Extraction algorithm

The extraction follows a strict 4-layer cascade. Stop at the first match.

```python
from collections import Counter

def _scan_for_material(text, lang):
    """Scan a text string for material matches. Returns material or None."""
    text_lower = str(text).lower() if pd.notna(text) else ''
    if not text_lower or text_lower == 'nan':
        return None
    # Multi-word first
    for pattern, key in MULTI_WORD_MATERIALS.items():
        if pattern in text_lower:
            return _resolve_material(key, lang)
    # Single-word
    words = re.findall(r'[a-zà-ÿäöüß-]+', text_lower)
    for word in words:
        if word in MATERIAL_INPUT and MATERIAL_INPUT[word] is not None:
            return _resolve_material(MATERIAL_INPUT[word], lang)
    return None

def extract_material(title, description='', product_type='', labels='',
                     google_category='', bullet_points='', lang='en'):
    """Extract material using 4-layer cascade. Returns (material, source, confidence).
    lang parameter controls the output language."""

    # --- LAYER 1: Title (high confidence) ---
    mat = _scan_for_material(title, lang)
    if mat:
        return mat, 'title', 'high'

    # --- LAYER 2: Description (medium confidence) ---
    mat = _scan_for_material(description, lang)
    if mat:
        return mat, 'description', 'medium'

    # --- LAYER 3: Other columns (medium confidence) ---
    for source_name, source_text in [
        ('product_type', product_type),
        ('labels', labels),
        ('google_category', google_category),
        ('bullet_points', bullet_points),
    ]:
        mat = _scan_for_material(source_text, lang)
        if mat:
            return mat, source_name, 'medium'

    # --- LAYER 4: No material found ---
    return None, 'none', 'unresolved'
```

### Step 5: Apply and build supplemental feed

```python
material_col = next((c for c in df.columns if c in ['material', 'materiaal', 'g:material', 'matériau', 'materiale']), None)
if material_col is None:
    material_col = 'material'
    df[material_col] = ''

id_col = next((c for c in df.columns if c in ['id', 'unique merchant sku', 'merchant item id']), None)
title_col = next((c for c in df.columns if c in ['title', 'titel', 'product name']), None)
desc_col = next((c for c in df.columns if c in ['description', 'beschrijving', 'product description']), None)

# Detect language (when running standalone)
output_lang = detect_output_language(df, title_col)  # Override if passed by orchestrator

results = {'filled': 0, 'unresolved': 0, 'already_had': 0}

for idx, row in df.iterrows():
    current = str(row[material_col]).strip() if pd.notna(row[material_col]) else ''
    if current and current.lower() not in ['', 'nan', 'none']:
        results['already_had'] += 1
        continue
    material, source, confidence = extract_material(
        row.get(title_col, ''), row.get(desc_col, ''), lang=output_lang)
    if material:
        df.at[idx, material_col] = material
        results['filled'] += 1
    else:
        results['unresolved'] += 1

# Build supplemental feed
supplemental = pd.DataFrame({
    'id': df[id_col],
    'material': df[material_col].fillna('').replace('nan', '')
})

output_path = "/mnt/user-data/outputs/supplemental_feed_material.xlsx"
supplemental.to_excel(output_path, index=False)
```

### Step 6: Report

Present summary with fill rate before/after, sample of filled values for spot-checking, and list of unresolved products. Always report the detected output language.

## Important guardrails

- Never overwrite existing material values
- Always output to a new file, never modify the original
- Flag uncertainty honestly — leave blank rather than guess
- Watch for false positives: "glass" in "sunglasses" is not a material, "canvas" in "Canvas Sneaker" could be either
- Some products genuinely have no material (digital goods, services) — leave blank
- **Output language must match feed language.** Never output "Holz" in an English feed or "Wood" in a German feed.
- **When in doubt, leave empty.** A missing material is better than a wrong material in the wrong language.

## Output format

The output is ALWAYS a supplemental feed — never a full copy of the original feed.

- **Single attribute**: 2 columns → `id` + `material`
- **Multiple attributes** (combined with other feed-attribute skills): 1 `id` column + 1 column per attribute
- Column names are always the **English Google Merchant Center attribute names**
- Format: **Excel (.xlsx)** — user can upload to Google Drive where it automatically converts to a Google Sheet for use as supplemental feed
- Include ALL products (also empty ones, so the user can see what still needs manual work)
