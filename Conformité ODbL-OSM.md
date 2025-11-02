# Ce que tu dois faire (checklist)

1. **Afficher l’attribution** partout où les données OSM sont visibles ou consommées

   * Texte recommandé (FR) :

     > Données © OpenStreetMap contributeurs, sous licence ODbL 1.0.
   * Ajoute un lien vers la page copyright d’OSM ou vers l’ODbL. ([openstreetmap.org][1])

2. **Dire clairement la licence des données**

   * Indique que les données servies par ton API proviennent d’OSM et restent sous **ODbL 1.0** (mets le lien vers la licence). ([openstreetmap.org][1])

3. **Partage à l’identique (Share-Alike) si tu “publies” la base dérivée**

   * Ton fichier **SQLite** (extraction/normalisation OSM) est une **base dérivée**.
   * Si tu **la mets à disposition du public** (p. ex. par une API ou un téléchargement), tu dois **offrir cette base dérivée sous ODbL** et **fournir un accès raisonnable** (dump téléchargeable, format lisible machine, notice de licence). ([opendatacommons.org][2])

4. **Conserver les notices**

   * Garde dans ton dépôt/dossier un fichier **NOTICE/LICENCE** mentionnant OSM, ODbL et la date/zone de collecte (Nantes, requête Overpass, etc.). ([opendatacommons.org][2])

5. **Ne pas ajouter de restrictions incompatibles**

   * Tes CGU/API ne doivent pas retirer les libertés de l’ODbL (pas de DRM/limitations qui empêchent la réutilisation permise par la licence). ([opendatacommons.org][2])

6. **Si tu mélanges avec d’autres données**

   * Sépare structurellement tes données privées (ex. avis/notes) pour rester en **“base collective”** : la partie OSM reste sous ODbL, ta base séparée garde sa licence. Évite de “fusionner” au point que ce soit une seule base indivisible, sinon Share-Alike s’applique à l’ensemble adapté. ([osmfoundation.org][3])

---

# Où mettre l’attribution (pratique)

* **README** du projet (en haut ou section “Crédits/licence”). ([osmfoundation.org][4])
* **Page /about** ou **docs API** (visible sans interaction). ([osmfoundation.org][4])
* **Réponses API** (optionnel mais propre) : ajoute un champ `attribution` ou un en-tête `X-Attribution`. ([osmfoundation.org][4])

> Formulation prête à copier (FR) :
> **Attribution** — Données © OpenStreetMap contributeurs. Licence : [ODbL 1.0]. Plus d’infos : [openstreetmap.org/copyright].
> **Source** — Extrait via Overpass API, zone “Nantes”, traitement local (normalisation, déduplication).

(les liens pointent vers la page copyright OSM et vers la licence ODbL). ([openstreetmap.org][1])

---

# Quand le Share-Alike s’applique (dans ton cas)

* Tu **scrapes**, **normalises** et **sers** ces données via une **API publique** → c’est un **usage public d’une base dérivée**.
* Tu dois donc **proposer un export** (ex. `restaurants_nantes.sqlite` ou `.csv` + schéma) sous **ODbL 1.0**, avec la notice d’attribution et un historique minimal (date, requête Overpass). ([opendatacommons.org][2])

> Piste concrète : publie mensuellement un dump (`/exports/restaurants_nantes_YYYY-MM.sqlite`) + `LICENSE-ODbL.txt` + `ATTRIBUTION.md`. ([datasud.fr][5])

---

# Produced Work vs Base dérivée (rappel rapide)

* **Produced Work** (ex. image de carte, capture, tuile raster) : **attribution seule** suffit, pas d’obligation de publier la base. ([openstreetmap.org][1])
* **Base dérivée** (ex. ton **SQLite** retravaillé/extrait) **mise à disposition** : **attribution + partage à l’identique** (ODbL) **obligatoires**. ([opendatacommons.org][2])

---

# À mettre dans ton dépôt/projet (mini-kit conformité)

* `LICENSE-ODbL.txt` (texte de la licence). ([opendatacommons.org][2])
* `ATTRIBUTION.md` (texte d’attribution + liens + requête Overpass + date). ([openstreetmap.org][1])
* `exports/` (dernière base dérivée en téléchargement). ([data.ru.nl][6])
* Note dans `README.md` : “Les données OSM sont sous ODbL ; la base dérivée fournie ici est également sous ODbL.” ([openstreetmap.org][1])

---

# Références essentielles

* **ODbL 1.0 (texte)** — Open Data Commons / OSM : obligations d’attribution, de partage à l’identique et d’accès aux bases dérivées (art. 4.3–4.6). ([opendatacommons.org][2])
* **Attribution OSM (guidelines officielles)** — où/comment afficher l’attribution. ([osmfoundation.org][4])
* **Copyright OSM (page officielle)** — formule d’attribution, liens. ([openstreetmap.org][1])
* **FAQ OSMF (licence)** — précisions sur Produced Work, bases collectives, etc. ([osmfoundation.org][3])

