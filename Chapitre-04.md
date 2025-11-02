# Chapitre-04 — Comprendre les enjeux juridiques et éthiques liés au scraping (3h30)
---

## 14 – Comprendre les aspects juridiques liés au RGPD et aux conditions d’utilisation d’un site

### 14.1 Cadre général (RGPD : quand votre scraping devient du “traitement de données personnelles”)

* **Le RGPD s’applique** dès qu’une donnée permet d’identifier directement ou indirectement une personne (ex. nom, email, identifiant, géolocalisation liée à une personne…). Un scraping est **illégal** s’il traite ces données **sans base légale** (consentement, intérêt légitime, obligation légale…) ni respect des principes. 
* **Principes clés** à respecter avant tout projet :

  1. **Finalité déterminée et légitime** : documenter pourquoi et comment vous collectez (et pour combien de temps).
  2. **Proportionnalité & minimisation** : ne collecter **que** ce qui est nécessaire (éviter les champs “au cas où”).
  3. **Sécurité & confidentialité** : protéger l’accès, le stockage, la transmission (chiffrement, contrôle d’accès, journalisation).
  4. **Transparence** : quand c’est pertinent, informer la personne concernée ou l’éditeur de la donnée (selon contexte). 
* **À retenir** : de la donnée “publique” n’est pas **libre de droit** ni libre de tout **traitement** au sens RGPD. L’exposition publique n’emporte pas renonciation aux droits. 

### 14.2 Conditions d’utilisation (CGU) & propriété intellectuelle

* Chaque site a des **CGU** qui encadrent l’accès, la reproduction et la réutilisation. Les ignorer peut engager votre **responsabilité civile** (voire pénale selon les juridictions / actes). 
* Les données peuvent être protégées par le **droit d’auteur** ou le **droit sui generis** des **bases de données** (directive 96/9/CE), surtout si l’extraction porte sur une **partie substantielle**. 
* **Bon réflexe** : vérifier **CGU** et **robots.txt** **avant** d’automatiser ; si c’est interdit → renoncer, négocier des droits, ou basculer vers une **API officielle**/Open Data. 

**Check-list “Légal-by-design” (à mettre en amont du projet)**

* Définir les **finalités** précises et la **base légale** (si données perso potentielles). 
* Appliquer la **minimisation** (ne prendre que les champs utiles). 
* Cartographier les **sources** (licences/CGU), consigner les **restrictions** de réutilisation. 
* Prévoir des **mesures de sécurité** et une **durée de conservation** limitée. 

---

## 15 – Débattre sur les enjeux sociétaux et de vie privée liés à l’Open Data

### 15.1 L’Open Data (opportunités & garde-fous)

* L’**Open Data** = données rendues librement accessibles par des institutions/entreprises (souvent sous licence). Réutilisation autorisée, mais **avec conditions** (attribution, périmètre, finalité). 
* Exigences usuelles : **anonymisation**, **traçabilité de l’origine**, **conformité juridique** (ne pas sortir du cadre de la licence). 

### 15.2 Risques de vie privée (corrélation & ré-identification)

* Même “publique”, une donnée peut devenir **sensible** si on la **croise** avec d’autres sources (profilage, habitudes, géolocalisation, inférences).
* Bonnes pratiques : **éviter les croisements abusifs**, toujours **minimiser**, **documenter la finalité** et vos critères d’agrégation. 

### 15.3 Responsabilité éthique du data engineer

* Responsabilité de respecter les **droits fondamentaux** :

  * **Équité** (pas de biais/discrimination), **loyauté** (ne pas tromper sur l’origine/les intentions), **transparence** (processus et traçabilité). 

**Atelier discussion (15–20 min)**

* Cas : un dataset d’avis publics sur des commerces — quelles **informations minimiser** ? quelles **métadonnées** conserver ? quelle **licence** appliquer ? Quels **risques de corrélation** avec d’autres sources ? (Objectif : formaliser une position éthique, pas seulement légale.) 

---

## 16 – Comprendre les enjeux éthiques liés à la transparence et aux consentements

### 16.1 Transparence & information

* Une collecte automatisée doit pouvoir être **expliquée** : **qui** collecte, **quoi**, **pourquoi**, **comment**, **combien de temps**, **avec qui** c’est partagé.
* Exemples d’actions : **mention explicite** si vous contrôlez la source, **documentation interne** du pipeline d’extraction, **respect** des délais/périmètres d’usage. 

### 16.2 Consentement

* Le **consentement explicite** est requis si vous traitez des **données personnelles identifiables** ; sans consentement, limiter aux données **anonymisées/agrégées** et vérifier la **base légale** (ex. intérêt légitime avec test de mise en balance). 

### 16.3 Éthique du scraping appliqué à l’IA

* Si les données servent à entraîner un modèle, vérifier : **origine** (licites), **non-discrimination** des datasets, **licences** (CC/Open License…), et bannir les **sources interdites** (réseaux sociaux sans API publique, médias avec clauses anti-extraction). 

**Mini-cadre “Éthique by default”**

* **Transparence** (notice claire), **Contrôle** (opt-out si possible), **Proportionnalité** (pas de sur-collecte), **Traçabilité** (journaliser les versions/sources), **Ré-évaluation périodique** (audit éthique). 

---

## 17 – Identifier les limites techniques au scraping

### 17.1 Mécanismes de protection courants (à connaître… pour ne **pas** les contourner)

* **CAPTCHA** (tests anti-bot), **Rate limiting** (quota/minute), **IP blocking** (blocage après abus), **User-Agent filtering**, **Honeypots** (pièges invisibles pour repérer les robots). 

### 17.2 Contournement éthique (ce qu’il **ne faut pas** faire)

* Le **contournement** d’un dispositif de protection **sans autorisation** viole généralement les CGU et peut constituer une **intrusion**.
* Interdits typiques : **simuler un humain pour tromper un CAPTCHA**, **changer d’IP** pour esquiver un blocage délibéré, **accéder à des zones non publiques**. 

### 17.3 Alternatives conformes (les bonnes voies)

* Utiliser les **API publiques/officielles** quand elles existent ;
* Privilégier l’**Open Data** et les datasets **documentés** ;
* Travailler dans le cadre d’un **partenariat**/convention ;
* Respecter le droit local (RGPD, **loi Informatique & Libertés**, **directive 96/9/CE** bases de données). 

**Check-list “Tech-compliance” pour vos scrapers**

* **Limiter la pression** (delays, quotas), **journaliser** les erreurs 429/403, **arrêter** proprement si “no-go” légal. 
* **User-Agent** explicite, **respect robots.txt/CGU**, basculer vers des **APIs** si disponible. 
* **Documenter** la source, la licence, la finalité et la durée de conservation. 

---

## Ce que vous devez maîtriser à l’issue du Chapitre-04

* **Qualifier** juridiquement votre cas d’usage (RGPD : finalité, minimisation, sécurité, transparence). 
* **Lire** et **respecter** CGU/licences & robots.txt ; identifier les situations à **risque** (propriété intellectuelle / base de données). 
* **Évaluer** l’impact sociétal/vie privée (corrélations, ré-identification) et formaliser des **garde-fous**. 
* **Choisir** des alternatives **conformes** (APIs, Open Data, partenariats) et **renoncer** aux contournements techniques. 
