# Chapitre-01 — Introduction au Web scraping de pages statiques (3H30)

## 1 – Introduction au Web Scraping

### 1.1 Définition

Le **Web Scraping** consiste à **extraire automatiquement** des données à partir de pages Web. L’idée : **télécharger** un document HTML (comme le ferait un navigateur) puis **analyser le DOM** pour récupérer le texte, les nombres, les liens, les images, etc., et enfin **structurer** ces données (CSV, JSON, base de données). Exemples d’usages : suivi de prix e-commerce, agrégation d’articles pour NLP, enrichissement d’une base relationnelle avec de l’open data. 

### 1.2 Aspects juridiques et RGPD (rappel essentiel)

Avant de scraper :

* **CGU & robots.txt** : consultez les conditions du site et son fichier `robots.txt`.
* **RGPD** : évitez les données personnelles ; sinon, justifiez la finalité, minimisez, sécurisez, informez.
* **Charge & éthique** : n’impactez pas le service (delays, user-agent clair).
  *(Les chap. 04 détaillent le cadre éthique & légal ; ici on garde le rappel opérationnel.)* 

---

## 2 – Comprendre le fonctionnement du Web

### 2.1 Les protocoles HTTP/HTTPS

Le scraping s’appuie sur **HTTP** : votre script joue le rôle d’un mini-navigateur et envoie des **requêtes** (GET/POST) à un **serveur** qui renvoie une **réponse** (statut, en-têtes, corps HTML). Points clés :

* **Méthodes** : `GET` (lecture), `POST` (envoi de formulaire/API), etc.
* **Statuts** : `200 OK`, `301/302` redirection, `403` interdit, `404` non trouvé, `429` trop de requêtes.
* **En-têtes** : `User-Agent`, `Accept-Language`, `Cookie`, `Referer`… utiles pour être toléré par le site.
* **HTTPS** : chiffré (TLS) — rien à changer côté scraping, mais attention aux **certificats** et à la **SNI** si vous utilisez des proxys.

### 2.2 Structure d’une page HTML

Une page comporte des balises hiérarchisées (DOM). Exemple minimal :

```html
<html>
  <head><title>Exemple produit</title></head>
  <body>
    <h1 class="titre">Ordinateur portable Lenovo</h1>
    <p class="prix">999.99 €</p>
    <a href="/acheter" id="achat">Acheter</a>
  </body>
</html>
```

Points clés :

* **Balises** : `<h1>`, `<p>`, `<a>`…
* **Attributs** : `class`, `id`, `href`…
* **Texte** : contenu entre les balises (ex. “999.99 €”).

> **Objectif scraping** : cibler les **sélecteurs** (id/class/structure) les plus **stables** pour extraire des éléments robustement. 

---

## 3 – Analyse d’une page avec l’inspecteur

Méthode de travail (indispensable avant de coder) :

1. Ouvrir la page dans le navigateur.
2. **Clic droit → Inspecter** : onglet **Elements** pour le DOM, **Network** pour les requêtes.
3. Identifier **balises/attributs** cibles (ex. `.price_color`, `h3 a[title]`).
4. Tester des sélecteurs **CSS/XPath** et **copier** ceux qui sont stables (éviter les selecteurs trop fragiles). 

---

## 4 – Extraire des données de pages statiques avec BeautifulSoup

> Outils retenus : **requests** (HTTP) + **BeautifulSoup** (parsing HTML) + **lxml** (parser rapide & tolérant). 

### 4.1 Installation (environnement propre)

```bash
python -m venv venv
source venv/bin/activate     # Windows: .\venv\Scripts\activate
pip install requests beautifulsoup4 lxml
```

**Pourquoi `lxml` ?** Plus rapide/robuste que `html.parser` dans la plupart des cas.

### 4.2 Téléchargement d’une page (et contrôles)

```python
import requests
from bs4 import BeautifulSoup

url = "https://books.toscrape.com/catalogue/page-1.html"
r = requests.get(url, timeout=15)          # + timeout pour éviter les blocages
r.raise_for_status()                        # lève une erreur si 4xx/5xx
soup = BeautifulSoup(r.text, "lxml")
```

**Décryptage** :

* `requests.get` → **requête GET** ; ajoutez `headers={"User-Agent": "..."}`
  si besoin.
* `timeout` → ne pas bloquer indéfiniment.
* `raise_for_status()` → fail-fast en cas d’erreur HTTP.
* `BeautifulSoup(..., "lxml")` → parse le HTML en **DOM navigable**.

### 4.3 Extraction simple (titres de livres)

```python
titres = [h3.a["title"] for h3 in soup.find_all("h3")]
```

**Décryptage** :

* `soup.find_all("h3")` → liste de balises `<h3>`.
* `h3.a` → le lien `<a>` **sous** ce `<h3>`.
* `["title"]` → lit l’attribut `title`.
  **Robustesse** : si l’attribut peut manquer, préférez `h3.a.get("title")`.

### 4.4 Extraire plusieurs éléments (titre, prix, dispo)

```python
prix  = [p.get_text(strip=True) for p in soup.find_all("p", class_="price_color")]
dispo = [s.get_text(strip=True) for s in soup.find_all("p", class_="instock availability")]

for t, pz, d in zip(titres, prix, dispo):
    print(f"{t:60} | {pz:10} | {d}")
```

**Décryptage** :

* `find_all("p", class_="price_color")` → filtre sur **tag + class**.
* `get_text(strip=True)` → récupère le texte **nettoyé** (whitespace).
* `zip(...)` → **aligne** les listes pour afficher **ligne par ligne**.

> **Astuce** : pour **sélecteurs complexes**, utilisez `select("css > path")` (CSS) ou `select_one`. 

---

## 5 – Récupération de texte, liens, images

*(Page “Books to Scrape” en exemple.)* 

### 5.1 Récupération du texte

**Tout le texte de la page** :

```python
texte = soup.get_text(separator="\n", strip=True)
print(texte[:500])
```

**Pourquoi utile ?** Diagnostic rapide, recherche de mots clés, sanity check.

### 5.2 Récupération des liens (relatifs → absolus)

```python
from urllib.parse import urljoin

liens_absolus = [urljoin(url, a["href"])
                 for a in soup.find_all("a", href=True)]
```

**Décryptage** :

* `href=True` → ne garde que les `<a>` avec attribut `href`.
* `urljoin` → transforme `/catalogue/foo.html` en URL absolue.

### 5.3 Récupération des images (et téléchargement contrôlé)

```python
imgs = [urljoin(url, img.get("src", "")) for img in soup.find_all("img")]
# Pour télécharger: boucler, GET chaque URL, écrire en binaire dans un dossier "images/"
```

**Bonnes pratiques** :

* limiter le volume (ex. 5 premières images),
* vérifier `status_code`, **MIME type** (`image/jpeg`),
* nommer les fichiers de façon sûre (**pas** d’input direct dans les chemins).

**Rappels “bon usage”** : robots.txt, `User-Agent` explicite, **sleep** entre requêtes. 

---

## 6 – Navigation dans le DOM

### 6.1 Méthodes principales (BeautifulSoup)

* `find(tag, attrs={})` → **premier** match
* `find_all(tag, attrs={})` → **liste** des matches
* `select("CSS selector")` / `select_one` → la **puissance des sélecteurs CSS**
  Exemple :

```python
for a in soup.select("ul.nav > li > a"):
    print(a["href"], a.get_text(strip=True))
```

**CSS vs XPath** : BeautifulSoup gère nativement **CSS** ; si vous préférez XPath, passez par **lxml.html** ou **parsel** (Scrapy). 

### 6.2 Attributs et texte

```python
lien = soup.find("a", id="achat")
print(lien["href"])         # attribut
print(lien.get_text())      # texte
```

**Anti-plantage** : utilisez `get("href")` plutôt que `["href"]` si l’attribut peut manquer.

---

## 7 – Scraper un site de presse ou e-commerce simple

*(Support “Books to Scrape”)* 

### 7.1 Exercice 1 — Extraction de titres et prix (→ JSON)

**Cible** : récupérer les **20 premiers livres** (page 1) avec leur **titre** et **prix**, et **sauvegarder en JSON**.
**Attendu** : un fichier `livres.json` contenant des objets `{titre, prix}`.
**Points pédagogiques** :

* Identifier les **balises** (ex. `article.product_pod`) et **sous-éléments** (`h3 a[title]`, `p.price_color`).
* Nettoyer les valeurs (strip, currency).
* **Sérialiser** en JSON (UTF-8, `ensure_ascii=False`).
* **Contrôler** le nombre d’éléments (doit être 20 pour la page test).

### 7.2 TP — Scraper un e-commerce simple (multi-pages, SQLite + CSV)

**Objectif** :

* Parcourir **plusieurs pages** (pagination),
* Extraire **titre, prix, disponibilité, note**,
* Stocker en **SQLite** (table `livres`),
* **Exporter** en CSV à la fin.
  **Structure minimale** :

```
tp_scraping/
├─ main.py
├─ requirements.txt
├─ livres.db
└─ export.csv
```

**Explications clés** :

* **Pagination** : détecter `li.next a[href]`, reconstruire l’URL, boucler.
* **SQLite** : créer la table si elle n’existe pas, **insertion par lots** (`executemany`), commit final.
* **Export CSV** : requêter la table puis écrire `writerows(rows)` ; inclure l’**entête**.
* **Nettoyage** : éliminer les espaces non imprimables, uniformiser les **prix** (séparateur décimal), convertir la **note** (texte → numérique si besoin).
* **Idempotence** (bonus) : utiliser une **clé unique** (ex. URL produit) pour éviter les doublons ou vider la table avant chaque run. 

---

## Annexes pédagogiques (pour ancrer les réflexes de pro)

### A. Choisir des sélecteurs robustes

* Préférer des **classes sémantiques** stables à des classes “obfusquées” (par ex. `product_pod` > `x1a2b3`).
* Éviter les **index numériques** fragiles (`div[3] > p[2]`) au profit de sélecteurs par **attributs** (`a[title]`, `p.price_color`).
* **Tester** vos sélecteurs dans l’inspecteur ou un **Notebook** avant d’industrialiser.

### B. Gérer l’internationalisation & l’encodage

* `response.apparent_encoding` (chardet intégré à requests) pour diagnostiquer un encodage atypique.
* Normaliser les nombres (virgule/point) et les **espaces fines** dans les prix.
* Toujours travailler en **UTF-8** (fichiers, CSV, JSON).

### C. Rate-limit & session

* **`time.sleep()`** entre les requêtes (ex. 1–2 s).
* Regrouper des requêtes sur une **Session** (`requests.Session()`) pour réutiliser les connexions, gérer cookies & headers communs.
* **429 Too Many Requests** → allonger la pause ; **ne pas** contourner agressivement.

### D. Tests rapides de validation

* **Comptage** : nombre d’items attendu.
* **Sanity checks** : prix non vides, `href` valides (absolus), pas de `None` inattendus.
* **Contrôle visuel** : exporter **5–10 lignes** en CSV et ouvrir dans un tableur.

---

## Ce que l’étudiant doit savoir faire à la fin du chapitre

* **Repérer** dans l’inspecteur les **zones** à extraire (DOM, classes, attributs).
* **Télécharger** et **parser** proprement une page **statique** (requests + BeautifulSoup + lxml).
* **Extraire** textes, liens, images et **naviguer** dans le DOM (find/select).
* **Paginer** et **enregistrer** les résultats (JSON/CSV/SQLite).
* **Appliquer** des **bonnes pratiques** (delays, user-agent, robustesse des sélecteurs, RGPD/robots). 
