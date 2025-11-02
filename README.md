# Sommaire du module — Web Scraping & Web Mining

## Jour 1

### Chapitre-01 — Introduction au web scraping de pages **statiques** (3h30)

1. **Introduction au web scraping** (définitions, usages, limites)
2. **Fonctionnement du Web** (HTTP/HTTPS, méthodes, statuts, en-têtes)
3. **Analyse d’une page** avec l’inspecteur du navigateur (DOM, Network)
4. **Extraction avec BeautifulSoup** (requests + parsing HTML)
5. **Récupérer** texte, liens, images (URL absolues, encodage, MIME)
6. **Navigation dans le DOM** (find/find_all, select CSS, robustesse)
7. **Cas pratique** : scraper un site de presse/e-commerce simple (pagination, JSON/CSV/SQLite)

### Chapitre-02 — Introduction au web scraping de pages **dynamiques** (3h30)

8. **Données générées par JavaScript**
   • Rejouer les appels XHR/fetch (API JSON)
   • Parsing **hybride** : HTML + JSON (JSON-LD / variables JS)
9. **Organisation & stockage avec SQLite**
   • `sqlite3` (création, insert, index)
   • **SQLAlchemy** (modèles, sessions)
   • Exports (CSV/JSON), traçabilité

---

## Jour 2

### Chapitre-03 — **Automatiser** le scraping à grande échelle (3h30)

10. **Scrapy : structure d’un projet** (Spider, Item, Pipeline, settings)
11. **Scrapy Shell** : tester sélecteurs **CSS/XPath**, mise au point rapide
12. **Pipelines** : nettoyage, validation, déduplication, **exports** (.json, .csv, .db)
13. **Spider multi-pages** : pagination, suivi de liens, bonnes pratiques (UA, délais, throttling)

### Chapitre-04 — Enjeux **juridiques & éthiques** du scraping (3h30)

14. **RGPD & CGU** : finalité, minimisation, sécurité, droit des bases de données
15. **Open Data & vie privée** : corrélation, ré-identification, devoirs du data engineer
16. **Transparence & consentements** : communication, documentation, usages IA
17. **Limites techniques** : anti-bots (CAPTCHA, rate limit), alternatives conformes (API/partenariats)
