# Chapitre-02 — Introduction au Web Scraping de pages dynamiques (3H30)
---

## 8 – Apprendre à récupérer les données générées par JavaScript avec des techniques de parsing **hybride** (HTML + JSON dans JS)

### 8.1 Comprendre les pages dynamiques & cibler les **requêtes réseau**

Certaines pages ne portent **pas** les données directement dans le HTML initial. Le navigateur exécute du **JavaScript** qui appelle une **API** (XHR/fetch) et insère la réponse (souvent **JSON**) dans le DOM. Méthode efficace pour un scraper :

1. **Ouvrir les DevTools** → onglet **Network** → filtre **XHR/Fetch** : observer l’URL appelée, la **méthode** (GET/POST), les **paramètres** (query/body), les **en-têtes** (auth, cookies), et la **réponse** (JSON).
2. **Rejouer** l’URL d’API **directement** depuis votre script (sans Selenium), quand c’est autorisé, pour récupérer un **JSON propre** (plus stable que scruter du HTML fragile). 

**Exemple (extrait du support) — appel d’une API JSON** :

```python
import requests

url = "https://api.exemple.com/produits"
response = requests.get(url)
data = response.json()

for produit in data["results"]:
    print(produit["nom"], produit["prix"])
```

**Décryptage détaillé** :

* `requests.get(url)` : émet une requête HTTP GET **comme un navigateur** (sans JS). Pensez à un `timeout` et un `User-Agent` explicite.
* `response.json()` : parse le **corps** en JSON (si `Content-Type: application/json`). Si le serveur renvoie du texte → utiliser `json.loads(response.text)`.
* **Structure des clés** : ici `data["results"]` ; vérifiez la **forme réelle** de la réponse (liste vs objet) dans l’onglet Network.
* **Erreurs & robustesse** : 401/403 (auth), 429 (rate-limit), 5xx (serveur). Utiliser `raise_for_status()` + **retries** exponentiels (backoff). 

### 8.2 Parsing **hybride** : extraire un **JSON** enfoui dans un `<script>` HTML

Quand il n’existe **pas** d’API séparée, le site peut **embarquer** les données dans un `<script>` (variable JS, JSON-LD, etc.). On récupère alors la page HTML puis on **détecte** et **extrait** le bloc JSON.

**Exemple (extrait du support)** :

```python
import re, json, requests
from bs4 import BeautifulSoup

url = "https://exemple.com/produits"
response = requests.get(url)
soup = BeautifulSoup(response.text, "lxml")

script = soup.find("script", string=re.compile("produits"))
json_str = re.search(r"\{.*\}", script.string).group(0)
data = json.loads(json_str)

for p in data["produits"]:
    print(p["nom"], p["prix"])
```

**Décryptage ligne par ligne** :

* `soup.find("script", string=re.compile("produits"))` : cherche **le script** dont le **contenu texte** contient “produits” (mot-clé pivot).
* `re.search(r"\{.*\}", script.string).group(0)` : capture **la 1ʳᵉ** séquence qui **commence** par `{` et **finit** par `}` (potentiellement **gourmande**).
* `json.loads(json_str)` : transforme la **chaîne** en **dictionnaire** Python.
* **Parcours** : `for p in data["produits"]` — on itère sur le tableau d’objets produit.

**Points pro & garde-fous** :

* **Robustesse regex** : utilisez des motifs **non gourmands** (`\{.*?\}`) ou mieux, ciblez une **clé** précise (`"produits"\s*:\s*\[`).
* **JSON-LD** : si le bloc est du **JSON-LD** (`<script type="application/ld+json">`), privilégiez `soup.find_all` sur ce `type` puis `json.loads`.
* **Encodage** : vérifier `response.encoding`/`apparent_encoding`.
* **Légalité** : respecter CGU/robots/RGPD, éviter tout **contournement** agressif.

---

## 9 – Organiser et stocker les données extraites en utilisant **SQLite** et insérer des données via **sqlite3** ou **SQLAlchemy**

> Objectif : passer d’un **flux brut** (JSON/API ou JSON dans `<script>`) à une **base locale** pour requêtes, exports et traçabilité. On vous montre **sqlite3** (standard library) puis **SQLAlchemy** (ORM). 

### 9.1 Création de la base & insertion simple (**sqlite3**)

**Exemple (extrait du support)** :

```python
import sqlite3

conn = sqlite3.connect("produits.db")
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS produits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nom TEXT,
    prix REAL
)
""")

for p in data["produits"]:
    cur.execute("INSERT INTO produits (nom, prix) VALUES (?, ?)", (p["nom"], p["prix"]))

conn.commit()
conn.close()
```

**Décryptage complet** :

* `sqlite3.connect("produits.db")` : ouvre/crée un **fichier** SQLite local (transportable, zéro serveur).
* `CREATE TABLE IF NOT EXISTS` : **idempotent** ; relance possible sans erreur.
* `prix REAL` : stocke le prix en **float** (pensez à la **monnaie** / **decimals** si exactitude requise).
* `VALUES (?, ?)` : **requêtes paramétrées** (sécurité & performances).
* `conn.commit()` : persiste les insertions ; **commit** fréquent si gros volumes.
* **Index** (à ajouter dans un vrai projet) : `CREATE INDEX idx_nom ON produits(nom)` pour accélérer la recherche. 

### 9.2 Lecture des données (**sqlite3**)

**Extrait du support** :

```python
conn = sqlite3.connect("produits.db")
cur = conn.cursor()

for row in cur.execute("SELECT * FROM produits"):
    print(row)

conn.close()
```

**À retenir** :

* Itération **streaming** (pas de `fetchall()` si volumineux).
* Filtrer/ordonner : `SELECT nom, prix FROM produits WHERE prix > ? ORDER BY prix DESC`.
* **WAL mode** (pour un scraper + API en parallèle) : `PRAGMA journal_mode=WAL;` (lecture/écriture concurrentes). 

### 9.3 Utilisation de **SQLAlchemy** (ORM)

**Extrait du support** :

```python
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.orm import declarative_base, Session

engine = create_engine("sqlite:///produits.db", echo=True)
Base = declarative_base()

class Produit(Base):
    __tablename__ = "produits"
    id = Column(Integer, primary_key=True)
    nom = Column(String)
    prix = Column(Float)

Base.metadata.create_all(engine)

with Session(engine) as session:
    for p in data["produits"]:
        produit = Produit(nom=p["nom"], prix=p["prix"])
        session.add(produit)
    session.commit()
```

**Décryptage détaillé** :

* `create_engine("sqlite:///produits.db")` : fabrique un **moteur** (driver+URL). `echo=True` logge le SQL (utile en dev).
* `declarative_base()` : base des **classes-modèles**.
* `class Produit(Base)` : modèle **Python** relié à une table SQL (`__tablename__`).
* `Column(...)` : mapping **type SQL ↔ type Python**.
* `Base.metadata.create_all` : crée la table à partir des **modèles** (si absente).
* **Unit of Work** : le **Session** agrège les insertions ; un seul `commit()` en fin de lot.
* **Évolutivité** : facile de passer de SQLite à **PostgreSQL** (changer l’URL).

### 9.4 Lecture (**SQLAlchemy**)

**Extrait du support** :

```python
with Session(engine) as session:
    for produit in session.query(Produit).all():
        print(produit.nom, produit.prix)
```

**Points clés** :

* `session.query(Produit).all()` : charge **tous** les enregistrements → préférez la **pagination** (`limit/offset`) en prod.
* Filtrage lisible : `session.query(Produit).filter(Produit.prix > 100)` …
* **Performance** : indexes, projections (ne demander que les colonnes utiles).

---

### Bonnes pratiques (rappel du support + compléments pro)

* **Toujours** vérifier s’il existe une **API publique/documentée** avant de scraper le HTML.
* Respecter **RGPD/robots/CGU** ; limiter la charge (delays, backoff sur 429).
* Gérer les **erreurs réseau** (timeouts, retries, journaux).
* En base : **schéma clair**, types adéquats, **index**, journaux d’**extraction** (`date_extraction`).
* Préparer l’**export** (CSV/JSON) et la **traçabilité** (ville, période, source). 

---

## Ce que vous savez faire à la fin du Chapitre-02

* Repérer et **rejouer** une **requête XHR** pour obtenir un **JSON propre**.
* Extraire un **JSON** encapsulé dans un **script** quand il n’y a pas d’API dédiée.
* **Structurer** vos données en **SQLite** avec `sqlite3` (simple) ou **SQLAlchemy** (propre/évolutif).
* Mettre en place des **bonnes pratiques** de robustesse, performance et conformité.

