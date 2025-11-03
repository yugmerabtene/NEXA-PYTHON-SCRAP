# 11.2 CSS vs XPath — résumé détaillé

## Idée générale

* **CSS Selectors** : syntaxe **courte** et **proche du front-end**. Idéal pour lister des cartes/éléments répétés, cibler par classes/ids, descendre dans l’arbre.
* **XPath** : **ultra-expressif**. Parfait pour les structures **complexes** (remonter vers un parent, cibler un `td` en fonction du `th` voisin, filtres sur le texte, axes avancés).
Dans Scrapy, les deux sont supportés : `response.css(...)` et `response.xpath(...)`. En interne, Scrapy compile les CSS en XPath, donc choisis surtout **lisibilité et robustesse**.

---

## Exemples rapides (Books to Scrape)

### 1) Titre des livres (listing)

**CSS (simple & lisible)**

```python
# Tous les titres sur la page de listing
response.css('article.product_pod h3 a::attr(title)').getall()
```

**XPath (équivalent robuste)**

```python
response.xpath('//article[contains(@class,"product_pod")]//h3/a/@title').getall()
```

### 2) Lien absolu vers la fiche livre

```python
# CSS → href relatifs ; on les absolu-ifie avec urljoin
hrefs = response.css('article.product_pod h3 a::attr(href)').getall()
links = [response.urljoin(h) for h in hrefs]  # important !

# XPath
hrefs = response.xpath('//article[contains(@class,"product_pod")]//h3/a/@href').getall()
links = [response.urljoin(h) for h in hrefs]
```

### 3) Prix & disponibilité

```python
# CSS
prices = response.css('article.product_pod p.price_color::text').getall()
stocks = [s.strip() for s in response.css('article.product_pod p.instock.availability::text').getall() if s.strip()]

# XPath
prices = response.xpath('//article[contains(@class,"product_pod")]//p[contains(@class,"price_color")]/text()').getall()
stocks = [s.strip() for s in response.xpath('//article[contains(@class,"product_pod")]//p[contains(@class,"instock")]/text()').getall() if s.strip()]
```

### 4) Rating (classe `star-rating Three/Four/...`)

```python
# CSS → on capture la 2e classe avec une regex
# ex: 'star-rating Three' → 'Three'
response.css('article.product_pod p.star-rating::attr(class)').re_first(r'star-rating\s+(\w+)')

# XPath (on récupère l’attribut class et on post-traite)
classes = response.xpath('//article[contains(@class,"product_pod")]//p[contains(@class,"star-rating")]/@class').getall()
# post-traitement Python : prendre le mot != "star-rating"
```

### 5) Détail page livre (table <th>/<td>) — **XPath recommandé**

```python
# Exemple : extraire le UPC depuis la table des specs
response.xpath('//table[contains(@class,"table-striped")]//tr[th/text()="UPC"]/td/text()').get()

# Variante normalisée (trim) pour n’importe quel champ :
# Price (incl. tax), Availability, Number of reviews, etc.
```

---

## Quand utiliser quoi ?

* **CSS (par défaut)**

  * Listings, cartes produit, attributs simples (classes/ids).
  * Lecture rapide et sélecteurs courts.
  * Exemple : `article.product_pod h3 a::attr(title)`

* **XPath (cas complexes)**

  * Remonter/aller latéralement (axes `parent`, `ancestor`, `following-sibling`…),
  * Filtrer sur le texte ou la structure (ex : `tr[th="UPC"]/td`),
  * Combinaisons précises de conditions (`contains()`, `starts-with()`, `normalize-space()`),
  * Exemple : `//tr[th="UPC"]/td/text()`

> **Règle pratique** : **CSS pour aller vite** sur les listes ; **XPath** dès que tu dois **remonter** dans l’arbre ou **croiser** les informations (tableaux, breadcrumbs, voisinage `th/td`, etc.).

---

## Combiner CSS & XPath (très utile)

Localiser **vite** avec CSS, puis affiner en XPath **relatif** :

```python
# 1) On prend une carte livre avec CSS
card = response.css('article.product_pod').get()

# 2) Puis on repart de la carte (sélecteur relatif) en XPath pour un filtre pointu
for sel in response.css('article.product_pod'):
    title = sel.xpath('.//h3/a/@title').get()  # XPath relatif au noeud courant
    price = sel.xpath('.//p[contains(@class,"price_color")]/text()').get()
```

---

## Traductions rapides (cheat-sheet)

| Besoin                    | CSS               | XPath                                 |
| ------------------------- | ----------------- | ------------------------------------- |
| Élément                   | `div`             | `//div`                               |
| Classe contient           | `.product_pod`    | `//*[contains(@class,"product_pod")]` |
| ID                        | `#main`           | `//*[@id="main"]`                     |
| Descendant (quelque part) | `div p`           | `//div//p`                            |
| Enfant direct             | `div > p`         | `//div/p`                             |
| Attr exact                | `a[title="X"]`    | `//a[@title="X"]`                     |
| Attr contient             | `a[href*="cat"]`  | `//a[contains(@href,"cat")]`          |
| Attr commence             | `a[href^="/"]`    | `//a[starts-with(@href,"/")]`         |
| Texte (Scrapy)            | `::text`          | `/text()` ou `normalize-space(.)`     |
| n-ième enfant*            | `li:nth-child(3)` | `parent/li[3]` (contexte requis)      |

* CSS `nth-child` ne se transpose pas 1:1 sans préciser le parent en XPath.

---

## Pièges & bonnes pratiques

* **CSS ne remonte pas** vers le parent (pas d’axe `parent`). Si besoin → **XPath**.
* **Liens relatifs** : toujours `response.urljoin(href)`.
* **Espaces/retours à la ligne** : `normalize-space()` (XPath) ou `.strip()` (Python).
* **`.get()` vs `.getall()`** : `.get()` = premier résultat ou `None`, `.getall()` = **liste**.
* **Robustesse classe** : côté XPath, préfère `contains(@class,"...")` (classes multiples).
* **Lisibilité** : choisis des sélecteurs qui survivront à de légères modifs d’HTML (évite les index “magiques”).

---

## Mini-pattern Scrapy (listing → détail)

```python
# 1) Listing (CSS lisible)
for card in response.css('article.product_pod'):
    title = card.css('h3 a::attr(title)').get()
    href  = response.urljoin(card.css('h3 a::attr(href)').get())
    price = card.css('p.price_color::text').get()
    yield response.follow(href, callback=self.parse_book, cb_kwargs={'title': title, 'price': price})

# 2) Détail (XPath précis)
def parse_book(self, response, title, price):
    upc = response.xpath('//tr[th="UPC"]/td/text()').get()
    category = response.xpath('normalize-space((//ul[@class="breadcrumb"]/li)[3]/a/text())').get()
    yield {
        'title': title,
        'price': price,
        'upc': upc,
        'category': category,
        'url': response.url
    }
```

