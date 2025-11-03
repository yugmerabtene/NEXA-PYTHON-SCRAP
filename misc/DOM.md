# Cours complet — DOM + Web Scraping du DOM avec **CSS** et **XPath**

*Source d’exemples : `https://books.toscrape.com` (orthographe officielle “to**scrape**”)*

---

## Objectifs

* Comprendre le **DOM** (Document Object Model) : nœuds, éléments, attributs, texte, hiérarchie.
* Savoir localiser des éléments via **sélecteurs CSS** et via **XPath** (forces, limites, traductions).
* Mettre en pratique sur *Books to Scrape* : extraire des livres, suivre la pagination, et sauvegarder les données.
* Appliquer de bonnes pratiques : robustesse, propreté des sélecteurs, gestion des erreurs, politesse réseau.

---

## Pré-requis & installation

* Python 3.10+ conseillé.
* Bibliothèques :

  ```bash
  pip install requests beautifulsoup4 lxml
  ```

  * **requests** : HTTP client simple et fiable.
  * **beautifulsoup4** : parsing HTML + **CSS selectors** via `.select()` / `.select_one()`.
  * **lxml** : parsing rapide + **XPath** (via `lxml.html`).

---

# 1) Le DOM en 7 minutes

### 1.1 Qu’est-ce que le DOM ?

Le **DOM** est la représentation en **arbre** d’un document HTML. Chaque balise est un **nœud** (node).
Types de nœuds fréquents :

* **Document** (racine)
* **Element** (`<div>`, `<a>`, `<p>`, …)
* **Attribute** (`href`, `class`, `id`, …) attaché à un élément
* **Text** (le texte entre les balises)
* **Comment** (`<!-- ... -->`)

### 1.2 Hiérarchie

* **Parent / enfant** : `<ul>` (parent) → `<li>` (enfants)
* **Frères** (siblings) : deux `<li>` sous le même `<ul>`
* **Descendant** : enfant, petit-enfant, etc.

### 1.3 Pourquoi c’est crucial pour le scraping ?

Parce qu’on **cible** des nœuds précis dans l’arbre pour en extraire du texte / attributs.
Deux familles d’outils :

* **CSS** (rapide, naturel si vous connaissez le front)
* **XPath** (expressif, puissant pour naviguer partout et filtrer)

---

# 2) Trouver ses sélecteurs dans le navigateur

1. Ouvrez `https://books.toscrape.com`.
2. Clic droit → **Inspecter** (DevTools).
3. Survolez un bloc livre (dans la page d’accueil, chaque livre est un `<article class="product_pod">`).
4. Notez la structure :

   * `article.product_pod`
     └─ `h3 > a[title]` (titre + lien relatif vers la page livre)
     └─ `p.price_color` (prix)
     └─ `p.instock.availability` (disponibilité)
     └─ `p.star-rating` (note via la classe, ex : `star-rating Three`)

Ces observations guident la création de sélecteurs **CSS** et **XPath**.

---

# 3) Sélecteurs **CSS** — Rappels utiles

| But                               | CSS                            |
| --------------------------------- | ------------------------------ |
| Par balise                        | `h3`, `a`, `p`                 |
| Par classe                        | `.product_pod`, `.price_color` |
| Par id                            | `#main`                        |
| Descendant (quelque part dessous) | `article .price_color`         |
| Enfant direct                     | `article > h3 > a`             |
| Frère direct                      | `h3 + p`                       |
| Attribut exact                    | `a[title="Exact"]`             |
| Attribut contient                 | `a[href*="catalogue"]`         |
| Attribut commence                 | `a[href^="/"]`                 |
| Attribut finit                    | `img[src$=".jpg"]`             |
| Plusieurs classes                 | `p.instock.availability`       |
| N-ième enfant                     | `li:nth-child(3)`              |

> **Note** : BeautifulSoup prend en charge un large sous-ensemble des sélecteurs CSS classiques (pas tous les pseudo-sélecteurs avancés).

---

# 4) **XPath** — Rappels utiles

## 4.1 Bases

* **Nœuds** : `//a` (tous les `<a>` du document), `//div[@id="main"]`
* **Relatif** à un nœud courant : `.//a` (descendants `<a>` du nœud courant)
* **Prédicats** (`[...]`) pour filtrer : `//a[@class="btn"]`
* **Texte** : `normalize-space(.)`, `text()`
* **Attribut** : `@href`, `@title`

## 4.2 Axes fréquents

* `child::` (implicite) → `div/p` == `child::div/child::p`
* `descendant::` → `//`
* `parent::` → `..`
* `ancestor::`, `following-sibling::`, `preceding-sibling::`

## 4.3 Fonctions utiles

* `contains(@class, "product")`
* `starts-with(@href, "/")`
* `normalize-space()` (trim + espaces normalisés)

---

# 5) Étude de cas — Page liste (*Books to Scrape*)

## 5.1 Extraction **CSS** (page unique) — Titre, prix, dispo, rating, lien

```python
import csv
from urllib.parse import urljoin

import requests
from bs4 import BeautifulSoup

BASE_URL = "https://books.toscrape.com/"

def parse_rating_class(classes):
    """
    Contexte : <p class="star-rating Three"> signifie 3/5.
    On récupère la 2e classe quand elle existe (ex: 'Three', 'Four'...).
    """
    # classes est une liste de classes CSS. On cherche celle qui n'est pas 'star-rating'.
    for c in classes:
        if c != "star-rating":
            return c  # ex: 'Three'
    return None

def scrape_listing_page_css(url):
    """
    Scrape une page 'listing' avec des sélecteurs CSS.
    Retourne une liste de dicts {title, price, availability, rating, url}
    """
    # 1) Faire la requête HTTP (avec un petit header User-Agent par politesse)
    resp = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
    resp.raise_for_status()  # lève une exception si statut >= 400

    # 2) Parser le HTML avec BeautifulSoup (parser 'lxml' plus rapide/robuste)
    soup = BeautifulSoup(resp.text, "lxml")

    # 3) Sélectionner tous les blocs livre
    books = []
    for card in soup.select("article.product_pod"):
        # 3a) Titre + lien (dans h3 > a[title])
        a = card.select_one("h3 > a")
        title = a.get("title", "").strip()
        rel_link = a.get("href", "").strip()
        # Les liens sont souvent relatifs → on fabrique un absolu
        link = urljoin(url, rel_link)

        # 3b) Prix (ex: £53.74) dans p.price_color
        price = card.select_one("p.price_color").get_text(strip=True)

        # 3c) Disponibilité dans p.instock.availability (ex: 'In stock (19 available)')
        availability = card.select_one("p.instock.availability").get_text(strip=True)

        # 3d) Note via class (ex: <p class="star-rating Three">)
        rating_node = card.select_one("p.star-rating")
        rating = parse_rating_class(rating_node.get("class", []))

        books.append({
            "title": title,
            "price": price,
            "availability": availability,
            "rating": rating,
            "url": link,
        })

    # 4) Chercher la pagination 'next' (si on souhaitait enchaîner ensuite)
    next_rel = soup.select_one("li.next > a")
    next_url = urljoin(url, next_rel["href"]) if next_rel else None

    return books, next_url

if __name__ == "__main__":
    start_url = BASE_URL  # la home est un listing paginé
    data, next_page = scrape_listing_page_css(start_url)

    # Sauvegarde rapide CSV (une page pour l’exemple)
    with open("books_page1_css.csv", "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)

    print(f"Extraits (CSS) : {len(data)} livres. Prochaine page : {next_page}")
```

**Points clés (CSS)**

* `article.product_pod` cible chaque carte livre.
* `h3 > a` → titre/lien (attribut `title` et `href`).
* `p.price_color`, `p.instock.availability`, `p.star-rating` → infos directes.

---

## 5.2 Extraction **XPath** (page unique) — mêmes champs

```python
import csv
from urllib.parse import urljoin

import requests
import lxml.html as lh

BASE_URL = "https://books.toscrape.com/"

def xpath_text(node, query):
    """
    Contexte : utilitaire pour récupérer du texte d'un sous-nœud XPath.
    - node : nœud lxml
    - query : XPath relatif (ex: './/p[@class="price_color"]/text()')
    Retourne une chaîne normalisée (strip + spaces).
    """
    found = node.xpath(query)
    if not found:
        return ""
    # found peut contenir des strings ou des éléments ; on garde les strings
    if isinstance(found[0], str):
        return " ".join(s.strip() for s in found if isinstance(s, str)).strip()
    else:
        return " ".join((t.strip() for t in found[0].itertext())).strip()

def scrape_listing_page_xpath(url):
    """
    Scrape une page 'listing' en XPath.
    Retourne (books, next_url)
    """
    resp = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
    resp.raise_for_status()

    doc = lh.fromstring(resp.text)
    doc.make_links_absolute(url)  # convertit tous les href/src en absolus

    books = []
    # Chaque livre : //article[contains(@class,"product_pod")]
    for card in doc.xpath('//article[contains(@class, "product_pod")]'):
        # Titre + lien : h3/a
        a = card.xpath('.//h3/a')[0]
        title = a.get("title", "").strip()
        link  = a.get("href", "").strip()

        price = xpath_text(card, './/p[contains(@class,"price_color")]/text()')
        availability = " ".join(card.xpath('.//p[contains(@class,"instock")]/text()')).strip()
        # Note via classe : récupérer la valeur de class et en extraire le mot après 'star-rating'
        classes = card.xpath('.//p[contains(@class,"star-rating")]/@class')
        rating = None
        if classes:
            # ex classes[0] == "star-rating Three"
            parts = classes[0].split()
            rating = next((p for p in parts if p != "star-rating"), None)

        books.append({
            "title": title,
            "price": price,
            "availability": availability,
            "rating": rating,
            "url": link,  # déjà absolu grâce à make_links_absolute
        })

    # Pagination 'next'
    next_link = doc.xpath('//li[@class="next"]/a/@href')
    next_url = urljoin(url, next_link[0]) if next_link else None

    return books, next_url

if __name__ == "__main__":
    start_url = BASE_URL
    data, next_page = scrape_listing_page_xpath(start_url)

    with open("books_page1_xpath.csv", "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)

    print(f"Extraits (XPath) : {len(data)} livres. Prochaine page : {next_page}")
```

**Points clés (XPath)**

* `//article[contains(@class,"product_pod")]` → robuste quand il y a plusieurs classes.
* `make_links_absolute(url)` évite de gérer `urljoin` à la main.
* Prédicats puissants (`contains()`) et `@class`, `@href`, etc.

---

# 6) Aller plus loin — Suivre la pagination & ouvrir la fiche produit

## 6.1 Boucle de pagination (version **CSS**)

```python
import csv
import time
from urllib.parse import urljoin

import requests
from bs4 import BeautifulSoup

BASE_URL = "https://books.toscrape.com/"

def listing_css_all_pages(start_url, delay=0.7):
    """
    Parcourt toutes les pages de listing en CSS et renvoie une liste de livres.
    delay : pause en secondes entre les pages (politesse).
    """
    session = requests.Session()
    session.headers.update({"User-Agent": "Mozilla/5.0"})

    url = start_url
    all_books = []

    while url:
        r = session.get(url, timeout=20)
        r.raise_for_status()
        soup = BeautifulSoup(r.text, "lxml")

        for card in soup.select("article.product_pod"):
            a = card.select_one("h3 > a")
            title = a.get("title", "").strip()
            link = urljoin(url, a.get("href", "").strip())
            price = card.select_one("p.price_color").get_text(strip=True)
            availability = card.select_one("p.instock.availability").get_text(strip=True)
            classes = card.select_one("p.star-rating").get("class", [])
            rating = next((c for c in classes if c != "star-rating"), None)

            all_books.append({
                "title": title, "price": price, "availability": availability,
                "rating": rating, "url": link
            })

        # Pagination
        next_rel = soup.select_one("li.next > a")
        url = urljoin(url, next_rel["href"]) if next_rel else None

        time.sleep(delay)  # politesse

    return all_books

if __name__ == "__main__":
    books = listing_css_all_pages(BASE_URL)
    with open("books_all_css.csv", "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=books[0].keys())
        writer.writeheader()
        writer.writerows(books)
    print(f"Total livres (CSS) : {len(books)}")
```

## 6.2 Récupérer des détails en page produit (version **XPath**)

Pour chaque livre, on ouvre sa page détail et on collecte des colonnes supplémentaires (UPC, prix HT/TTC, stock, catégorie…).

```python
import time
from urllib.parse import urljoin

import requests
import lxml.html as lh

def book_details_xpath(book_url, session=None):
    """
    Ouvre la page détail d'un livre et extrait :
    - upc, price_excl_tax, price_incl_tax, tax, availability, num_reviews
    - category (via le fil d'Ariane)
    """
    sess = session or requests.Session()
    sess.headers.update({"User-Agent": "Mozilla/5.0"})
    r = sess.get(book_url, timeout=20)
    r.raise_for_status()

    doc = lh.fromstring(r.text)
    doc.make_links_absolute(book_url)

    # Le tableau 'table.table-striped' contient des paires th/td
    def td_for(th_text):
        # Cherche la cellule <td> dont <th> frère vaut th_text
        xp = f'//table[contains(@class,"table-striped")]//tr[th[text()="{th_text}"]]/td/text()'
        found = doc.xpath(xp)
        return found[0].strip() if found else ""

    upc = td_for("UPC")
    price_excl = td_for("Price (excl. tax)")
    price_incl = td_for("Price (incl. tax)")
    tax = td_for("Tax")
    availability = td_for("Availability")
    num_reviews = td_for("Number of reviews")

    # Catégorie dans le breadcrumb : le 3e <li> en général
    category = doc.xpath('normalize-space((//ul[@class="breadcrumb"]/li)[3]/a/text())')

    return {
        "upc": upc,
        "price_excl_tax": price_excl,
        "price_incl_tax": price_incl,
        "tax": tax,
        "availability_detail": availability,
        "num_reviews": num_reviews,
        "category": category,
    }

# Exemple d’enchaînement :
# - on suppose que vous avez déjà une liste 'books' avec la clé 'url'
if __name__ == "__main__":
    sample = "https://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html"
    print(book_details_xpath(sample))
```

---

# 7) Script “tout-en-un” (sélecteur **au choix** CSS *ou* XPath)

> **Astuce pédagogique** : ci-dessous, un seul script avec un “mode” pour illustrer **les deux approches**.
>
> * `MODE = "css"` → listing en CSS + détails en XPath (on mélange pour montrer les deux)
> * `MODE = "xpath"` → listing en XPath + détails en XPath

```python
"""
Scraper Books to Scrape :
- Parcourt TOUTES les pages de listing (pagination)
- Extrait titre, prix, dispo, rating, url
- Ouvre chaque page livre pour extraire upc, prix HT/TTC, taxe, stock, nb avis, catégorie
- Sauvegarde CSV

Paramètre MODE : "css" ou "xpath"
"""

import csv
import time
from urllib.parse import urljoin

import requests
from bs4 import BeautifulSoup
import lxml.html as lh

BASE_URL = "https://books.toscrape.com/"
MODE = "css"  # changer en "xpath" pour la variante XPath
PAUSE = 0.6   # politesse entre pages

UA = {"User-Agent": "Mozilla/5.0"}

def parse_listing_css(doc_html, page_url):
    """Retourne (books, next_url) à partir d'un HTML de listing, en CSS."""
    soup = BeautifulSoup(doc_html, "lxml")
    items = []
    for card in soup.select("article.product_pod"):
        a = card.select_one("h3 > a")
        title = a.get("title", "").strip()
        link = urljoin(page_url, a.get("href", "").strip())
        price = card.select_one("p.price_color").get_text(strip=True)
        availability = card.select_one("p.instock.availability").get_text(strip=True)
        classes = card.select_one("p.star-rating").get("class", [])
        rating = next((c for c in classes if c != "star-rating"), None)
        items.append({"title": title, "price": price, "availability": availability, "rating": rating, "url": link})
    nxt = soup.select_one("li.next > a")
    next_url = urljoin(page_url, nxt["href"]) if nxt else None
    return items, next_url

def parse_listing_xpath(doc_html, page_url):
    """Retourne (books, next_url) à partir d'un HTML de listing, en XPath."""
    doc = lh.fromstring(doc_html)
    doc.make_links_absolute(page_url)
    items = []
    for card in doc.xpath('//article[contains(@class,"product_pod")]'):
        a = card.xpath('.//h3/a')[0]
        title = a.get("title", "").strip()
        link = a.get("href", "").strip()
        price = "".join(card.xpath('.//p[contains(@class,"price_color")]/text()')).strip()
        availability = " ".join(card.xpath('.//p[contains(@class,"instock")]/text()')).strip()
        classes = " ".join(card.xpath('.//p[contains(@class,"star-rating")]/@class'))
        rating = None
        if classes:
            parts = classes.split()
            rating = next((p for p in parts if p != "star-rating"), None)
        items.append({"title": title, "price": price, "availability": availability, "rating": rating, "url": link})
    next_href = doc.xpath('//li[@class="next"]/a/@href')
    next_url = urljoin(page_url, next_href[0]) if next_href else None
    return items, next_url

def book_details_xpath(book_url, session):
    """Détails de la page produit via XPath."""
    r = session.get(book_url, timeout=20)
    r.raise_for_status()
    doc = lh.fromstring(r.text)

    def td_for(th_text):
        xp = f'//table[contains(@class,"table-striped")]//tr[th[text()="{th_text}"]]/td/text()'
        v = doc.xpath(xp)
        return v[0].strip() if v else ""

    category = doc.xpath('normalize-space((//ul[@class="breadcrumb"]/li)[3]/a/text())')

    return {
        "upc": td_for("UPC"),
        "price_excl_tax": td_for("Price (excl. tax)"),
        "price_incl_tax": td_for("Price (incl. tax)"),
        "tax": td_for("Tax"),
        "availability_detail": td_for("Availability"),
        "num_reviews": td_for("Number of reviews"),
        "category": category,
    }

def run(mode="css", out_csv="books_full.csv"):
    session = requests.Session()
    session.headers.update(UA)

    url = BASE_URL
    rows = []
    while url:
        r = session.get(url, timeout=20)
        r.raise_for_status()

        if mode == "css":
            items, next_url = parse_listing_css(r.text, url)
        else:
            items, next_url = parse_listing_xpath(r.text, url)

        for it in items:
            # détails toujours en XPath (pour montrer la technique)
            details = book_details_xpath(it["url"], session)
            row = {**it, **details}
            rows.append(row)

        url = next_url
        time.sleep(PAUSE)

    # Sauvegarde CSV
    if rows:
        with open(out_csv, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=rows[0].keys())
            writer.writeheader()
            writer.writerows(rows)
    print(f"OK: {len(rows)} livres → {out_csv}")

if __name__ == "__main__":
    run(MODE, out_csv=f"books_full_{MODE}.csv")
```

---

# 8) CSS vs XPath — Quand choisir quoi ?

| Critère              | **CSS**                                                                | **XPath**                                                                  |
| -------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Simplicité usuelles  | Très simple pour classes/ids, relations enfant/frère                   | Plus verbeux au début                                                      |
| Expressivité         | Limité pour remonter (pas de `parent`), filtres avancés moins naturels | Très puissant : axes (`ancestor`, `following-sibling`), fonctions, filtres |
| Lecture/maintenance  | Très lisible si on connaît le front                                    | Parfait pour cas complexes (tableaux, filtres par texte)                   |
| Disponibilité        | Direct dans BeautifulSoup (`.select`)                                  | Direct dans lxml (`.xpath`)                                                |
| Cas d’usage typiques | Listes, cartes produit, attributs simples                              | Tableaux de détails, breadcrumbs, filtrage texte fin                       |

**Règle pratique**

* **Commencez CSS** pour les listings simples.
* **Passez XPath** dès que vous devez :

  * remonter vers un parent / ancêtre,
  * cibler “le `td` dont le `th` voisin a tel texte”,
  * normaliser du texte ou utiliser des filtres complexes.

---

# 9) Table de traduction rapide CSS → XPath

| CSS                | XPath                                                    |
| ------------------ | -------------------------------------------------------- |
| `div`              | `//div`                                                  |
| `.class`           | `//*[contains(@class,"class")]`                          |
| `#id`              | `//*[@id="id"]`                                          |
| `div > p`          | `//div/p`                                                |
| `div p`            | `//div//p`                                               |
| `a[href*="cat"]`   | `//a[contains(@href,"cat")]`                             |
| `a[href^="/"]`     | `//a[starts-with(@href,"/")]`                            |
| `img[src$=".jpg"]` | `//img[substring(@src, string-length(@src)-3) = ".jpg"]` |
| `li:nth-child(3)`  | `(//li)[3]` (attention : pas le même contexte exact)     |

> **Note** : le `nth-child` ne se transpose pas 1:1 sans connaître le contexte (parent). On emploie souvent `(parent//li)[3]` ou `parent/li[3]`.

---

# 10) Robustesse & bonnes pratiques

* **Headers** : mettez un `User-Agent`.
* **Timeouts** : `timeout=20` minimum.
* **Retry/backoff** (optionnel) : si vous enchaînez beaucoup de pages.
* **Pauses** : `time.sleep(0.5~1s)` entre pages.
* **Sélecteurs stables** : préférez classes/structures stables à des indices fragiles.
* **Nettoyage** : `strip()`, `normalize-space()`.
* **Journalisation** : loggez le nombre d’items, les URLs visitées.
* **Évolutivité** : séparez “listing” et “détails”, gardez des fonctions pures testables.
* **Légalité & éthique (rappel bref)** : respectez les CGU, la charge du site, la confidentialité des données. *Books to Scrape* est un site didactique prévu pour l’exercice.

---

# 11) Exercices proposés (facultatif)

1. **Lister les catégories** (menu latéral), puis pour chaque catégorie, scrapez tous les livres via pagination.
2. **Convertir** le pipeline complet : version 100 % CSS, puis 100 % XPath.
3. **Exporter JSON Lines** (une ligne JSON par livre).
4. **Filtrer** les livres notés “Four” ou “Five” et prix > £20 (convertissez au float).

---

## FAQ rapide

**Q.** *Je n’obtiens pas le texte d’un nœud en XPath.*
**R.** Utilisez `normalize-space(string(.))` ou `//node/text()` si c’est un texte direct.

**Q.** *Pourquoi mon lien est cassé ?*
**R.** Les liens sont souvent **relatifs**. Utilisez `urljoin(base, href)` ou `doc.make_links_absolute(base)`.

**Q.** *CSS peut-il faire comme XPath pour `th→td` ?*
**R.** Difficile/fragile en CSS. En XPath : `//tr[th="UPC"]/td/text()` est idéal.

---

Si tu veux, je peux adapter ce cours **en version “TP pas à pas”** (objectifs, étapes, livrables) ou **faire une variante Scrapy** (avec `parsel`, sélecteurs CSS/XPath natifs et pipelines).
