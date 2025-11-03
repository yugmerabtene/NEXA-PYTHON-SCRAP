

# 1) Définitions précises

* **Fetch** : l’**opération réseau** qui consiste à **demander** une ressource à une URL et à **recevoir** une réponse (statut HTTP, en-têtes, corps). C’est l’étape transport : DNS → TCP/TLS → requête HTTP → réponse.
* **Parse** : l’**opération d’analyse** du **corps** de la réponse (HTML, JSON, XML, CSV…) pour **en extraire** des valeurs structurées (champs, objets, lignes) conformes à un **schéma cible**.

> En pipeline : **fetch → parse → (validation/nettoyage → stockage)**. Le fetch ne comprend **aucune** logique métier d’extraction ; le parse ne fait **aucun** accès réseau.

---

# 2) **Fetch** en détail (récupération de la ressource)

## 2.1 Entrées et paramètres

* **URL** + **méthode** : `GET`, `HEAD`, `POST`, etc.
* **En-têtes** : `User-Agent` (identification), `Accept(-Language)`, `Referer`, `Cookie`, `Authorization` (Basic/Bearer), `If-None-Match`/`If-Modified-Since` (cache conditionnel).
* **Corps** (pour `POST`/`PUT`) : formulaire, JSON.
* **Réseau** : **timeout** (connexion/lecture), **proxy** (HTTP/HTTPS/SOCKS), **auth**, **TLS** (certificats), **réessais** avec **backoff** (429/5xx).
* **Politesse** : `DOWNLOAD_DELAY`, quotas, fenêtre horaire, rotation d’IP **si** licence le permet (sinon s’abstenir).

## 2.2 Résultats

* **Statut** HTTP (200, 301/302, 403, 404, 429, 5xx).
* **En-têtes réponse** : `Content-Type` (MIME), `Content-Encoding` (gzip/br), `Set-Cookie`, `ETag/Last-Modified`.
* **Corps** : octets bruts (à **décompresser** si nécessaire).
* **Métadonnées** : URL finale après redirections, durée, taille, adresse IP, HTTP/2 vs HTTP/1.1.

## 2.3 Gestion robuste des erreurs

* **Échecs réseau** : DNS, TCP, TLS, timeout de connexion/lecture.
* **Échecs applicatifs** : 4xx/5xx. Décider **si** on réessaie (ex. 429 → backoff exponentiel + jitter ; 404 → ne pas réessayer).
* **Redirections** : suivre dans la limite fixée ; rejeter boucles.
* **Cache** : exploiter `ETag`/`If-None-Match` pour diminuer la charge si autorisé.

## 2.4 Observabilité et conformité

* **Journaliser** : URL, statut, latence, taille, en-têtes clés, raison d’échec.
* **Limiter la pression** : concurrence contrôlée, délais, AutoThrottle (Scrapy).
* **Respecter** CGU/licence/robots et toute **réservation de droits** (TDM opt-out). Ne pas contourner les barrières (auth/CAPTCHA).

---

# 3) **Parse** en détail (analyse et extraction)

## 3.1 Pré-traitements

* **Décompression** (`gzip`, `br`) selon `Content-Encoding`.
* **Encodage** : détecter `charset` (`Content-Type`) ou heuristique (HTML meta, BOM) ; convertir en texte UTF-8.
* **Normalisation** : uniformiser espaces, sauts de ligne, entités HTML.

## 3.2 HTML (DOM)

* **Construction du DOM** (lxml/BeautifulSoup/Parsel).
* **Sélecteurs** :

  * **CSS** pour 80 % des cas (balises, classes, attributs, hiérarchie) ;
  * **XPath** si besoin de filtrer par **texte**, **remonter** au parent, atteindre **frères** ou exprimer des conditions avancées (*contains*, *starts-with*, *position()*).
* **Extraction** : texte (`::text`, `normalize-space()`), attributs (`::attr(href)` / `@href`), conversion de liens **relatifs** → **absolus** (prise en compte de `<base href>`).
* **Données structurées** : JSON-LD `application/ld+json`, Microdata, variables JavaScript contenant du JSON (regex ciblées + `json.loads`).
* **Fallbacks** : prévoir plusieurs sélecteurs (révisions de front fréquentes).

## 3.3 JSON / XML

* **JSON** : validation de schéma (clés obligatoires), gestion des **valeurs manquantes**, types (string → float/date), fusion d’objets imbriqués, pagination via **tokens**.
* **XML** : **namespaces**, XPath, CDATA, attributs vs contenu.

## 3.4 Post-traitements métier

* **Nettoyage / typage** : prix “1 099,00 €” → `1099.00` (decimal), dates ISO-8601, coordonnées float, unités normalisées.
* **Validation** : champs obligatoires, bornes, formats (regex), listes contrôlées (ex. codes pays).
* **Déduplication & clés** : définir une **clé naturelle** (ex. `(osm_type, osm_id)`), ignorer doublons ou **UPSERT**.
* **Qualité** : règles de cohérence (ex. lat/lon valides), métriques d’extraction (taux de champs non vides).

## 3.5 Sorties et responsabilités

* **Sortie du parse** : **items** structurés (dict/objets) **uniquement**.
* **Ce que le parse ne fait pas** : pas de requêtes réseau, pas d’écriture en base (ça relève des **pipelines**), pas de logique de planification (c’est le **scheduler**/spider).
* **Testabilité** : le parse doit pouvoir être testé **hors réseau** à partir d’un **snapshot** de réponse.

---

# 4) Articulation **fetch ↔ parse** (séparation des rôles)

* **Contrat d’interface** : le fetch remet une **Response** (statut, en-têtes, corps) ; le parse lit le corps et produit des **items** ou des **URLs à suivre** (dans Scrapy, `Request` avec `callback`).
* **Reproductibilité** : stocker des **réponses brutes** (échantillons) pour rejouer le parse après une évolution du site.
* **Idempotence** : relancer le parse sur la même réponse doit donner le **même** résultat.
* **Performance** : réduire la taille traitée (compression, cache légal), limiter le coût de parsing (sélecteurs efficaces, JSON direct quand possible).

---

# 5) Implémentations usuelles

## 5.1 Scrapy

* **Fetch** : objets `Request` planifiés par le moteur, middlewares (retries, UA, proxy, AutoThrottle).
* **Parse** : méthode `parse(response)` (ou autre callback) → CSS/XPath via `response.css/xpath` → `yield item` et/ou `yield response.follow(...)`.
* **Erreurs** : `errback` pour capturer statuts/échecs.
* **Après parse** : **pipelines** (nettoyage, validation, export/DB).

## 5.2 Python `requests`

* **Fetch** : `Session` (pool/keep-alive), timeouts séparés (connexion/lecture), `raise_for_status()` ;
* **Parse** : HTML via BeautifulSoup/lxml, JSON via `response.json()`.
* **Bonnes pratiques** : en-têtes explicites, retries avec backoff, `ETag` si autorisé.

## 5.3 API **Fetch** (JavaScript, navigateur / Node)

* **Fetch** : `fetch(url, {method, headers, body, signal})` → `Response`.
* **Lecture** : `response.ok`, `response.status`, `response.text()`, `response.json()`.
* **Points d’attention** : **CORS**, `credentials`, `AbortController` pour timeouts, lecture **streaming** si nécessaire.

---

# 6) Check-lists rapides

**Fetch (réseau)**

* [ ] `User-Agent` clair, délais/quota définis, retries + backoff
* [ ] Timeouts (connexion/lecture), TLS vérifié, proxy si besoin
* [ ] Gestion redirections, 304 via ETag si permis
* [ ] Journalisation (URL, statut, latence), respect CGU/licences/robots

**Parse (analyse)**

* [ ] Décompression & encodage corrects
* [ ] Sélecteurs CSS par défaut ; XPath si texte/parent/frères/conditions
* [ ] Normalisation (espaces, unités, dates), validation (formats, bornes)
* [ ] Clé d’unicité/UPSERT ; métriques de qualité ; tests hors ligne

---

## En résumé

* **Fetch** : tout ce qui concerne la **récupération** fiable et conforme de la ressource (transport, erreurs, performance, politesse).
* **Parse** : tout ce qui concerne l’**interprétation** déterministe du contenu pour **produire** des données propres, validées et normalisées.
  Cette séparation améliore la **robustesse**, la **testabilité** et la **conformité** de vos scrapers.
