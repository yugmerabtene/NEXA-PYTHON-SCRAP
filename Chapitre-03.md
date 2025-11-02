# Chapitre-03 — Automatiser le scraping à grande échelle (3H30)
---

## 10 – Utiliser **Scrapy** : structure d’un projet (**Spider, Item, Pipeline**)

### 10.1 Introduction

**Scrapy** est un framework Python orienté **projet** pour le crawling/extraction : moteur **asynchrone** performant, planification de requêtes, files d’attente d’URLs, extraction via **CSS/XPath**, et transformation/stockage via **pipelines**. Résultat : un scraping **stable**, **scalable** et **structuré** (séparation claire spider ↔ items ↔ pipelines). 

**Pourquoi Scrapy plutôt que `requests`/`BeautifulSoup` ?**

* Moteur réseau **asynchrone** → forte **concurrence** contrôlée.
* **Suivi d’URLs** intégré (gestion du `dupefilter`, redirections).
* **Middlewares** (retries, UA/proxy), **pipelines** (nettoyage, DB), **export** natif (JSON/CSV/XML/SQLite). 

---

### 10.2 Installation

```bash
pip install scrapy
scrapy version  # vérifier l’installation
```

*(Gardez un venv propre par projet.)* 

---

### 10.3 Création d’un projet

```bash
scrapy startproject books_scraper
cd books_scraper
```

Arborescence standard (ce qui va où, et pourquoi) :

```
books_scraper/
├─ scrapy.cfg                # point d’entrée Scrapy (projet)
└─ books_scraper/
   ├─ __init__.py
   ├─ items.py               # schémas des données (Items)
   ├─ middlewares.py         # hooks requêtes/réponses (UA, proxys, retries…)
   ├─ pipelines.py           # post-traitement (nettoyer, valider, DB, export)
   ├─ settings.py            # configuration centrale (UA, délais, pipelines…)
   └─ spiders/               # vos crawlers
      └─ __init__.py
```

**Intérêt pédagogique** : architecture **durable**, composants **remplaçables** (ex. exporter vers SQLite aujourd’hui, PostgreSQL demain) sans changer la logique d’extraction. 

---

### 10.4 Création d’un **Spider**

Un **spider** définit :

* **`name`** (identifiant CLI),
* **`start_urls`** (point(s) de départ),
* **`parse()`** (logique d’extraction + navigation). 

```python
import scrapy

class BooksSpider(scrapy.Spider):
    name = "books"  # utilisé dans `scrapy crawl books`
    start_urls = ["https://books.toscrape.com/catalogue/page-1.html"]

    def parse(self, response):
        # 1) Extraction : on cible une carte produit
        for article in response.css('article.product_pod'):
            yield {
                'titre': article.css('h3 a::attr(title)').get(),
                'prix': article.css('p.price_color::text').get(),
                'disponibilite': article.css('p.instock.availability::text').get().strip()
            }
        # 2) Pagination : on suit le lien "next"
        next_page = response.css('li.next a::attr(href)').get()
        if next_page:
            yield response.follow(next_page, callback=self.parse)
```

**Explications essentielles**

* `response.css('…')` → sélecteurs **CSS** ultra lisibles (voir §11).
* `.get()` renvoie **le premier** match (ou `None`) ; `.getall()` pour **la liste**.
* `yield {…}` émet un **item** (dict ici) vers le **pipeline** (cf. §12).
* `response.follow()` résout **automatiquement** les URLs relatives, et rejoue `parse()` → **toute la pagination** est couverte sans boucle `while`. 

> **Variante pro** : définir un **Item** (dans `items.py`) pour typer/valider vos champs, puis **Item Loader** et **processors** (normalisation prix/dates). *(Hors diapo, mais recommandé en prod.)*

---

### 10.5 Lancer un spider & exporter

```bash
scrapy crawl books -o livres.json
```

* `crawl books` → exécute le spider **`books`**.
* `-o livres.json` → **export** auto (supporte **JSON/CSV/XML/SQLite**).
  *(Astuce : paramétrer `FEEDS` dans `settings.py` pour des exports datés, encodage UTF-8, ordre des colonnes, etc.)* 

---

## 11 – Réaliser un **Scrapy Shell** avec des sélecteurs **CSS/XPath**

### 11.1 Le **Scrapy Shell**

Le shell est votre **laboratoire** : tester sélecteurs **CSS/XPath** et expressions **regex** directement sur une page avant d’écrire le spider.

```bash
scrapy shell https://books.toscrape.com/
```

Dans le shell :

```python
response.css('h3 a::attr(title)').getall()
response.xpath('//p[@class="price_color"]/text()').getall()
```

**Bénéfice** : boucler **très vite** jusqu’aux bons sélecteurs, puis copier-coller dans votre spider. *(Essayez aussi `view(response)` pour ouvrir le HTML localement.)* 

---

### 11.2 **CSS** vs **XPath** (quand utiliser quoi)

* **CSS** : concis, lisible, idéal 80 % des cas front ; pseudo-sélecteur `::text`/`::attr(src)` utile.
  `response.css('div.article h3 a::attr(title)').get()`
* **XPath** : plus **expressif** sur structures complexes, axes, conditions.
  `response.xpath('//div[@class="article"]//h3/a/@title').get()`

> Bon réflexe : **CSS** par défaut, **XPath** pour cas tordus. 

---

## 12 – Développer des **pipelines** de nettoyage & export (**.json, .csv, .db**)

### 12.1 Définition & cycle de vie

Les **pipelines** sont des classes appelées **après** le spider, pour **post-traiter** chaque item : **nettoyage**, **validation**, **conversion de types**, **déduplication**, **insertion en base**, **export**. Méthodes clés :

* `open_spider(self, spider)` → initialisation (connexion DB, fichiers, etc.)
* `process_item(self, item, spider)` → traitement **unitaire** (peut lever `DropItem`)
* `close_spider(self, spider)` → fermeture propre (flush/commit/close) 

### 12.2 Exemple de **pipeline SQLite** (explications ligne-par-ligne)

```python
import sqlite3

class SQLitePipeline:
    def open_spider(self, spider):
        self.conn = sqlite3.connect('livres.db')         # 1) Ouvre/crée la base locale
        self.cur  = self.conn.cursor()
        self.cur.execute("""                           # 2) Schéma idempotent
            CREATE TABLE IF NOT EXISTS livres (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                titre TEXT,
                prix  TEXT,
                disponibilite TEXT
            )
        """)

    def process_item(self, item, spider):
        # 3) Insertion paramétrée (sécurité/perfs)
        self.cur.execute("""
            INSERT INTO livres (titre, prix, disponibilite)
            VALUES (?, ?, ?)
        """, (item['titre'], item['prix'], item['disponibilite']))
        self.conn.commit()                               # 4) Commit (ou batching par N)
        return item

    def close_spider(self, spider):
        self.conn.close()                                # 5) Fermeture propre
```

Points pro :

* **Paramètres `?`** → évite l’injection SQL, meilleure planification.
* **Batching** : pour la perf, faites `commit()` toutes les **N** lignes (ex. 100/1000).
* **Types** : convertissez `prix` en **float/decimal** dans le pipeline pour un schéma propre. 

### 12.3 Activer le pipeline

Dans `settings.py` :

```python
ITEM_PIPELINES = {
    'books_scraper.pipelines.SQLitePipeline': 300,  # l’ordre compte (plus petit = plus tôt)
}
```

Vous pouvez **chaîner** plusieurs pipelines : nettoyage → validation → DB. 

### 12.4 Autres formats d’**export**

En CLI, Scrapy sait **écrire directement** en JSON/CSV/XML :

```bash
scrapy crawl books -o data.json
scrapy crawl books -o data.csv
scrapy crawl books -o data.xml
```

*(Astuce : `FEED_EXPORT_ENCODING='utf-8'`, `FEEDS` pour définir format/champs/overwrite.)* 

---

## 13 – Créer un spider pour **scraper plusieurs pages** d’un site

### 13.1 Navigation multi-pages (**pagination**)

Patron classique : extraire les **cartes** d’une page, **émettre** les items, **suivre** le lien `next`.

```python
def parse(self, response):
    for article in response.css('article.product_pod'):
        yield {
            'titre': article.css('h3 a::attr(title)').get(),
            'prix': article.css('p.price_color::text').get(),
            'disponibilite': article.css('p.instock.availability::text').get().strip()
        }

    next_page = response.css('li.next a::attr(href)').get()
    if next_page:
        yield response.follow(next_page, callback=self.parse)
```

**Pourquoi c’est robuste ?**

* `response.follow()` gère les **URLs relatives**.
* La **file d’attente** interne évite les doublons et gère l’ordre.
* Aucune boucle explicite → code **déclaratif** et **lisible**. 

> *Aller plus loin (optionnel)* : **`CrawlSpider`** + `Rule(LinkExtractor(...))` pour suivre des **modèles de liens** (catégories, paginations), et **`allowed_domains`** pour borner le champ d’exploration.

---

### 13.2 Structure **finale** du projet (attendue)

```
books_scraper/
├─ books_scraper/
│  ├─ items.py
│  ├─ pipelines.py
│  ├─ settings.py
│  └─ spiders/
│     └─ books_spider.py
├─ livres.db        # si pipeline SQLite activé
├─ livres.csv       # si export CSV demandé
└─ scrapy.cfg
```

*(Cette structure reflète exactement ce que vous obtenez après mise en place du spider, des pipelines et des exports.)* 

---

### 13.3 Bonnes pratiques (à paramétrer dans `settings.py`)

* **Limiter la pression** : `CONCURRENT_REQUESTS`, `DOWNLOAD_DELAY` (ex. 0.5–2 s), **AutoThrottle** pour s’adapter.
* **S’identifier proprement** : `USER_AGENT` personnalisé.
* **Respect** : `ROBOTSTXT_OBEY=True` par défaut, **CGU** du site, RGPD.
* **Robustesse** : `RETRY_TIMES`, `HTTPERROR_ALLOW_ALL=False`, **errbacks** pour journaliser les erreurs.
* **Qualité des données** : valider dans **pipelines**, **dropper** les items incomplets, indexer en base. 

---

## Résultats attendus en fin de chapitre

* Être capable de **créer un projet Scrapy**, d’implémenter un **spider** qui **pagine** proprement, et d’**exporter** les données.
* Savoir **tester** les sélecteurs avec le **Scrapy Shell** (CSS/XPath), choisir la bonne approche.
* Maîtriser la **chaîne Items → Pipelines → Exports/DB** et activer les **settings** essentiels (délais, UA, pipelines).
  *(Conforme au plan : structure projet, shell, pipelines, spider multi-pages.)* 

---
