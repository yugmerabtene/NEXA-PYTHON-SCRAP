# 1) RGPD (données personnelles)

* **Oui, les données “publiques” restent des données personnelles** : si vous collectez des infos sur des personnes (nom, email, avis, profils…), vous devez avoir une **base légale** (souvent “intérêt légitime”), **informer** les personnes (art. 14 quand la collecte n’est pas faite auprès d’elles), respecter la **minimisation** et offrir les **droits** (accès, opposition, etc.). ([Légifrance][1])
* Lorsque la collecte est **à grande échelle** ou **susceptible de risque élevé**, une **AIPD/DPIA** peut s’imposer (listes CNIL). ([CNIL][2])

# 2) Droit d’auteur & **droit sui generis des bases de données** (UE + France)

* L’UE protège les **bases de données** (Directive 96/9/CE). En France, cela se traduit par le **CPI** : le **producteur** peut interdire l’**extraction** d’une partie substantielle, et même l’extraction **répétée** de parties non substantielles si cela excède l’usage normal (CPI **L342-1** / **L342-2**). Les atteintes sont **pénales** (jusqu’à 3 ans et 300 000 € ; **L343-4**). ([Légifrance][3])
* Jurisprudence clef CJUE : **British Horseracing Board c. William Hill** (notion d’extraction substantielle / répétée) ; **Ryanair c. PR Aviation** (si la base **n’est pas** protégée par le droit sui generis/copyright, le **contrat/CGU** peut interdire le scraping et rester opposable). ([Cour européenne de justice][4])

# 3) **Exceptions “TDM”** (fouille de textes et de données)

* La Directive **(UE) 2019/790** crée deux exceptions TDM :
  **Art. 3** (recherche publique) et **Art. 4** (autres usages, y compris commerciaux **si** le titulaire **n’a pas réservé ses droits**). La **réservation** doit être faite **de manière appropriée** et, pour les contenus mis en ligne, **en “machine-readable”** (métadonnées, conditions du site, etc.). Transposé en France (CPI, notamment **L122-5-3**). ([Légifrance][5])
* En pratique, les éditeurs peuvent signaler l’**opt-out TDM** via des mécanismes **machine-readable** (p. ex. **TDM Reservation Protocol/W3C**). C’est encore un écosystème en évolution. ([W3C][6])

# 4) **Contrats / CGU** des sites

* Même sans droit sui generis, les **CGU** peuvent interdire l’usage de robots ou l’extraction automatisée, et être **opposables** si acceptées (voir **Ryanair c. PR Aviation**). ([Cour européenne de justice][7])

# 5) **Accès non autorisé** (Code pénal)

* Le **scraping ne doit pas contourner** des barrières techniques d’accès (authentification, captchas, anti-bot stricts…). En France, l’**accès ou maintien frauduleux** dans un système (STAD) est pénalement réprimé (**art. 323-1 s.**). ([Légifrance][8])

# 6) **Open data & licences**

* Si la source est **ouverte** (ex. **Licence Ouverte Etalab 2.0**), la réutilisation est large mais sous **conditions d’attribution** (citer la source, etc.). Pour **OSM**, la licence **ODbL** impose **attribution** et **share-alike** sur les bases dérivées. ([alliance.numerique.gouv.fr][9])

---

## Check-list pratique (résumé actionnable)

1. **Identifier la nature des données** : personnelles ? (→ RGPD + info art. 14) ; base de données protégée ? (→ L342-1/L342-2). ([gdpr-info.eu][10])
2. **Lire la licence/CGU** de la source et vérifier un **opt-out TDM** éventuel (métadonnées/tdm-rep). ([EUR-Lex][11])
3. **Ne pas contourner** des mesures techniques d’accès (→ Code pénal 323-1 s.). ([Légifrance][8])
4. **Base légale RGPD** (souvent intérêt légitime) + **information art. 14**, **droits**, **sécurité**, **minimisation** ; faire une **AIPD** si nécessaire. ([European Commission][12])
5. **Open data** : respecter la **Licence Ouverte** / **ODbL** (attribution, partage à l’identique si ODbL). ([alliance.numerique.gouv.fr][9])

Parfait — voici la **grille de conformité UE/France “Web Scraping”** prête à utiliser projet par projet (format check-list). Elle couvre RGPD, droit des bases de données, exceptions TDM, CGU/contrats, accès technique, et licences Open Data.

---

# Grille de conformité — Web Scraping (UE/France)

## 0) Fiche d’identité du projet

* **Nom du projet** :
* **Cible(s) / URL(s)** :
* **Finalité(s)** (ex. veille prix, open data, R&D) :
* **Exposition** (interne / API publique / publication d’un dataset) :
* **Responsable de traitement** (si données perso) :
* **Pays d’hébergement / transferts hors UE** :

---

## 1) Nature des données & RGPD

| Critère                | Questions à trancher                                                | Preuves/éléments à collecter                          | Contrôles/mesures                           | Statut             | Notes |
| ---------------------- | ------------------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------- | ------------------ | ----- |
| Données personnelles ? | Identifient directement/indirectement une personne ?                | Exemples de champs, captures, dictionnaire de données | Minimisation (prendre le strict nécessaire) | ☐ Oui ☐ Non        |       |
| Base légale            | Intérêt légitime / consentement / autre ?                           | Test de mise en balance, registre                     | Notice d’info art.14, opt-out si possible   | ☐ OK ☐ À compléter |       |
| Information art.14     | Personnes informées si collecte indirecte ?                         | Texte d’information, emplacement                      | Page info/Privacy, FAQ                      | ☐ OK ☐ N/A         |       |
| DPIA (AIPD)            | Risque élevé / grande échelle ?                                     | Grille d’évaluation, score                            | Lancer DPIA si seuil atteint                | ☐ Oui ☐ Non        |       |
| Droits & rétention     | Droits (accès/opposition/suppression) opérables ? Durées définies ? | Politique de rétention, procédure                     | Purges planifiées, point de contact         | ☐ OK ☐ À définir   |       |
| Sécurité               | Accès, chiffrement, journaux, cloisonnement                         | Matrice d’accès, mesures techniques                   | Journalisation, durcissement, rate-limit    | ☐ OK ☐ À renforcer |       |

---

## 2) Droit des bases de données (UE/FR)

| Critère                    | Questions                                                 | Preuves                   | Contrôles                     | Statut          | Notes |
| -------------------------- | --------------------------------------------------------- | ------------------------- | ----------------------------- | --------------- | ----- |
| Base protégée ?            | La source est une **base de données** au sens juridique ? | Mentions légales, éditeur |                               | ☐ Oui ☐ Non     |       |
| Extraction substantielle ? | Volume/valeur **substantiel** ou **répété** ?             | Métriques d’extraction    | Limiter volumes, échantillons | ☐ Oui ☐ Non     |       |
| Usage normal               | Ne pas excéder l’usage normal de la base ?                | CGU/licence               | Seuils, cache, quotas         | ☐ OK ☐ À revoir |       |

---

## 3) Exceptions **TDM** (Text & Data Mining)

| Critère                             | Questions                                                 | Preuves                     | Contrôles                       | Statut                | Notes |
| ----------------------------------- | --------------------------------------------------------- | --------------------------- | ------------------------------- | --------------------- | ----- |
| Éligibilité TDM                     | Recherche (art.3) ou autre (art.4) ?                      | Statut organisme / finalité | Documenter l’exception invoquée | ☐ Art.3 ☐ Art.4 ☐ N/A |       |
| **Réservation de droits** (opt-out) | Le site a **réservé** ses droits TDM (machine-readable) ? | robots/meta/tdm-rep, CGU    | Respect de l’opt-out ⇒ renoncer | ☐ Oui ☐ Non           |       |

---

## 4) **CGU / Contrats**

| Critère                  | Questions                                 | Preuves                   | Contrôles                             | Statut      | Notes |
| ------------------------ | ----------------------------------------- | ------------------------- | ------------------------------------- | ----------- | ----- |
| Interdictions explicites | CGU interdisent robots/scraping ?         | Copie CGU, date           | Si interdit : demander licence/accord | ☐ Oui ☐ Non |       |
| API officielle           | Existe une API/licence de réutilisation ? | Docs/API, tarifs          | Basculer vers API si dispo            | ☐ Oui ☐ Non |       |
| Opposabilité             | Avez-vous accepté des CGU (clic/compte) ? | Traçabilité d’acceptation | Respect strict                        | ☐ Oui ☐ Non |       |

---

## 5) Accès & mesures techniques

| Critère              | Questions                                | Preuves               | Contrôles                            | Statut          | Notes |
| -------------------- | ---------------------------------------- | --------------------- | ------------------------------------ | --------------- | ----- |
| Barrières techniques | Auth/captcha/paywall/anti-bot présents ? | Captures Network/HTML | Ne **pas** contourner                | ☐ Oui ☐ Non     |       |
| Charge & politesse   | Quotas, délais, UA, heures creuses ?     | Paramètres crawler    | `User-Agent` clair, `delay`, backoff | ☐ OK ☐ À régler |       |

---

## 6) Licences **Open Data** / ODbL / Etalab

| Critère            | Questions                  | Preuves               | Contrôles                         | Statut            | Notes |
| ------------------ | -------------------------- | --------------------- | --------------------------------- | ----------------- | ----- |
| Source ouverte     | Licence Etalab/CC/ODbL ?   | Texte de licence      | Respect clauses (attribution, SA) | ☐ Oui ☐ Non       |       |
| Attribution        | Texte d’attribution prêt ? | Formulation/lien      | Affichage dans app/API/docs       | ☐ OK ☐ À faire    |       |
| Share-Alike (ODbL) | Base **dérivée** publiée ? | Décision d’exposition | Publier dump sous ODbL si oui     | ☐ Oui ☐ Non ☐ N/A |       |

---

## 7) Décision & conditions de go-live

* **Décision** : ☐ GO ☐ GO avec conditions ☐ NO-GO
* **Conditions bloquantes** (à lever avant mise en prod) :

  1. …
  2. …
* **Mesures techniques minimales** : UA dédié, `DOWNLOAD_DELAY`, `RETRY/Backoff`, quotas, logs, purge, sécurité.
* **Documentation** à archiver : CGU/licence capturée (date), analyse RGPD (base légale, art.14), évaluation TDM/opt-out, métriques d’extraction, modèle d’attribution.

---

## Annexes — modèles prêts à coller

**A) Mention d’attribution (Open Data générique)**

> Source : [Nom de la source] — Licence : [Nom de la licence]. Données adaptées et redistribuées selon les termes de la licence.

**B) Mention OSM/ODbL**

> Données © OpenStreetMap contributors, sous licence ODbL 1.0. Les bases dérivées mises à disposition sont publiées sous la même licence.

**C) Politique de politesse (extrait pour README)**

* `User-Agent` : *VotreNom-Scraper/1.0 ([contact@domaine.tld](mailto:contact@domaine.tld))*
* `DOWNLOAD_DELAY` : 1–2 s ; **backoff** sur 429/5xx ; pas de scraping aux heures de pointe du site.
* Arrêt automatique si opt-out TDM détecté.

---

## Mode d’emploi (rapide)

1. **Remplir** la grille pour chaque source ciblée.
2. **Joindre** copies de CGU/licences (horodatées) et captures Network.
3. **Classer** le risque (RGPD / DB / CGU / TDM / pénal) et **décider** GO/NO-GO.
4. **Implémenter** les conditions (techniques & légales) avant mise en prod.
5. **Re-valider** périodiquement (CGU/licences évoluent).


