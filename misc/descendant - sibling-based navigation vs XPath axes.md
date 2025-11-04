## INTRODUCTION – Les deux styles de navigation XPath

En XPath, il existe deux grands modes de navigation dans un document HTML ou XML :

| Type de navigation                                                  | Description                                                                                                                                                             | Axes utilisés                                                                     |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| **Navigation complète (avec axes ascendantes)**                     | Permet de descendre, remonter et traverser le DOM dans toutes les directions. On utilise les axes `ancestor::`, `parent::`, `descendant::`, `following-sibling::`, etc. | `ancestor::`, `parent::`, `descendant::`, `child::`, `sibling::`                  |
| **Navigation descendante et latérale (descendant / sibling-based)** | Se limite à la descente dans le DOM et aux relations horizontales (frères). On ne remonte jamais vers les ancêtres.                                                     | `child::`, `descendant::`, `self::`, `preceding-sibling::`, `following-sibling::` |

En résumé :

* **Version avec axes complets :** on peut partir d’un élément bas et remonter pour trouver son parent ou son ancêtre, puis redescendre ailleurs.
* **Version descendante / sibling-based :** on part toujours d’un point haut (souvent `<article>` ou `<div>`) et on explore uniquement **vers le bas** ou **autour**.

---

## VERSION 1 – Code utilisant une navigation complète (avec `ancestor::`)

### Objectif

Partir du prix (`<p class="price_color">`) et remonter au parent `<article>` pour retrouver le titre et la description de chaque livre.

### Code complet

```python
import requests
from lxml import html

base_url = "https://books.toscrape.com/"
response = requests.get(base_url)
response.encoding = 'utf-8'
tree = html.fromstring(response.text)

prix_nodes = tree.xpath('//p[@class="price_color"]')
print(f"Nombre de prix trouvés : {len(prix_nodes)}")

for p in prix_nodes:
    prix = p.text.strip() if p.text else "Prix manquant"

    # Remonter à <article> (ancêtre) puis redescendre vers le titre
    titre_nodes = p.xpath('ancestor::article/h3/a/@title')
    titre = titre_nodes[0].strip() if titre_nodes else "Titre introuvable"

    # Récupérer le lien vers la page du livre
    lien_nodes = p.xpath('ancestor::article/h3/a/@href')
    lien = lien_nodes[0] if lien_nodes else None
    if not lien:
        continue
    if not lien.startswith("http"):
        lien = base_url + lien.replace("../", "")

    # Télécharger la page du livre
    r2 = requests.get(lien)
    r2.encoding = 'utf-8'
    page = html.fromstring(r2.text)

    # Extraire la description du produit
    desc_nodes = page.xpath('//div[@id="product_description"]/following-sibling::p[1]/text()')
    description = desc_nodes[0].strip() if desc_nodes else "Pas de description disponible"

    print(f"{titre} | {prix}\n{description}\n{'-'*80}")
```

### Explication détaillée

1. **Téléchargement et parsing**

   * `requests.get(base_url)` : télécharge la page d’accueil.
   * `html.fromstring(response.text)` : crée un arbre DOM utilisable par XPath.

2. **Recherche des prix**

   * `//p[@class="price_color"]` : cible chaque élément `<p>` qui contient un prix.
   * On parcourt ensuite cette liste.

3. **Remontée avec `ancestor::`**

   * `ancestor::article` : remonte jusqu’au conteneur `<article>` du livre.
   * `/h3/a/@title` : redescend à l’intérieur du même `<article>` pour lire le titre.
   * On obtient le titre et le lien même si le prix était dans une sous-division (`<div class="product_price">`).

4. **Chargement de la page individuelle**

   * Chaque lien renvoie vers une page produit contenant la description.
   * On utilise une nouvelle requête `requests.get(lien)` pour l’ouvrir.

5. **Récupération de la description**

   * `//div[@id="product_description"]/following-sibling::p[1]/text()` : cherche le paragraphe qui suit le bloc "Product Description".
   * `strip()` nettoie les espaces et retours à la ligne.

6. **Affichage**

   * Affiche le titre, le prix et la description de chaque livre.

### Sortie attendue

```
A Light in the Attic | £51.77
It's hard to imagine a world without A Light in the Attic...
--------------------------------------------------------------------------------
Tipping the Velvet | £53.74
An exuberant debut novel that follows the antics of a lesbian...
--------------------------------------------------------------------------------
```

Cette version exploite les axes **ascendants (ancestor)**, donc une **navigation complète**.

---

## VERSION 2 – Code utilisant une navigation descendante et latérale uniquement

### Objectif

Ici on ne remonte jamais vers les ancêtres.
On part de l’élément `<article>` (bloc de livre) et on descend ou navigue entre ses enfants pour obtenir les données.

### Code complet

```python
import requests
from lxml import html

base_url = "https://books.toscrape.com/"
response = requests.get(base_url)
response.encoding = 'utf-8'
tree = html.fromstring(response.text)

articles = tree.xpath('//article[@class="product_pod"]')
print(f"Nombre d'articles trouvés : {len(articles)}")

for a in articles:
    # Descente dans le DOM (aucune remontée)
    titre_nodes = a.xpath('.//h3/a/@title')
    titre = titre_nodes[0].strip() if titre_nodes else "Titre introuvable"

    lien_nodes = a.xpath('.//h3/a/@href')
    lien = lien_nodes[0] if lien_nodes else None
    if not lien:
        continue
    if not lien.startswith("http"):
        lien = base_url + lien.replace("../", "")

    # Descendre encore pour atteindre le prix
    prix_nodes = a.xpath('.//p[@class="price_color"]/text()')
    prix = prix_nodes[0].strip() if prix_nodes else "Prix manquant"

    # Utiliser l'axe sibling pour trouver la disponibilité
    dispo_nodes = a.xpath('.//p[@class="price_color"]/following-sibling::p[@class="instock availability"]/text()')
    dispo = dispo_nodes[0].strip() if dispo_nodes else "Disponibilité inconnue"

    # Ouvrir la page du livre et extraire la description
    r2 = requests.get(lien)
    r2.encoding = 'utf-8'
    page = html.fromstring(r2.text)

    desc_nodes = page.xpath('//div[@id="product_description"]/following-sibling::p[1]/text()')
    description = desc_nodes[0].strip() if desc_nodes else "Pas de description disponible"

    print(f"{titre} | {prix} | {dispo}\n{description}\n{'-'*80}")
```

### Explication détaillée

1. **Point de départ**

   * On commence directement depuis `<article>` ; on n’aura donc jamais besoin de `ancestor::` ni `parent::`.

2. **Descente hiérarchique**

   * `.//h3/a/@title` : descend dans l’article jusqu’à `<a>` pour lire le titre.
   * `.//p[@class="price_color"]/text()` : descend dans la même arborescence pour trouver le prix.

3. **Navigation latérale (sibling)**

   * `following-sibling::p[@class="instock availability"]/text()` : part du `<p class="price_color">` et se déplace vers son frère `<p class="instock availability">`.

4. **Ouverture de la page individuelle**

   * Même méthode que la première version, mais cette fois l’origine du lien est descendante depuis `<article>`.

5. **Extraction de la description**

   * Même XPath que précédemment car sur les pages internes, la structure reste identique.

6. **Affichage final**

   * Titre, prix, disponibilité, et description.

### Sortie attendue

```
A Light in the Attic | £51.77 | In stock
It's hard to imagine a world without A Light in the Attic...
--------------------------------------------------------------------------------
Tipping the Velvet | £53.74 | In stock
An exuberant debut novel that follows the antics of a lesbian...
--------------------------------------------------------------------------------
```

Cette version illustre une **navigation descendante et sibling-based** :
on descend uniquement dans les balises enfants ou on se déplace entre des frères au même niveau, mais on ne remonte jamais vers les parents ou ancêtres.

---

## SYNTHÈSE COMPARATIVE

| Élément               | Version avec ancestor                                    | Version descendante/sibling                      |
| --------------------- | -------------------------------------------------------- | ------------------------------------------------ |
| Point de départ       | `<p class="price_color">`                                | `<article class="product_pod">`                  |
| Axe principal         | `ancestor::article` (ascendant)                          | `child::`, `descendant::`, `following-sibling::` |
| Direction du parcours | ascendante et descendante                                | uniquement descendante et latérale               |
| Portée                | peut retrouver des éléments situés plus haut dans le DOM | limitée au bloc courant                          |
| Performance           | légèrement plus lente (remontées dans le DOM)            | plus rapide, portée restreinte                   |
| Utilisation typique   | extraction à partir d’un élément spécifique              | scraping structuré de blocs complets             |

---

## Conclusion générale

* **La version avec `ancestor::`** est plus flexible : elle permet de partir de n’importe quel élément et de retrouver son contexte global.
* **La version descendante / sibling-based** est plus stricte : elle convient mieux quand la page a une structure stable et répétitive (comme une liste d’articles).

Ces deux approches sont donc **complémentaires**.
Dans un projet professionnel, on choisit souvent la version descendante pour la performance et la lisibilité, et la version avec axes complets pour gérer des structures HTML plus complexes ou irrégulières.
