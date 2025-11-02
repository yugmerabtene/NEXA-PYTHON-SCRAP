# Cours complet SQLite (indépendant du projet)
---

## 1) Comprendre SQLite

* **Qu’est-ce que c’est ?**
  Un SGBD **embarqué** : pas de serveur séparé, la base tient dans **un fichier `.sqlite/.db`**. Parfait pour apps desktop/mobile, scripts, protos, tests, ou petites apps prod.
* **Forces** : simplicité de déploiement, rapidité locale, zéro admin serveur, ACID, transactions, intégrité.
* **Limites** : concurrence **écriture** limitée (un seul writer à la fois), pas de comptes utilisateurs/ACL internes (sécurité = OS/app), pas de sharding ni de réplication native.

**Outils** : binaire `sqlite3` (CLI), bibliothèques disponibles dans tous les langages (Python `sqlite3`, Go `database/sql`, etc.).

---

## 2) Modèle de stockage & architecture

* **Fichier unique** : pages B-Tree pour tables & index.
* **Journaux** : modes `DELETE` (par défaut historique), `WAL` (Write-Ahead Logging) recommandé pour **lectures et écritures concurrentes** (lectures non bloquées pendant l’écriture).
* **ACID** : transactions atomiques, cohérentes, isolées, durables.
* **Isolation** : par défaut “**SERIALIZABLE**” (sauf si `read_uncommitted=ON`).
* **Concurrence** : *lecteurs multiples*, *un seul écrivain* à la fois (mais rapide sur machine locale).

---

## 3) Types & règles de typage (spécificités SQLite)

* **Storage classes** : `NULL`, `INTEGER`, `REAL`, `TEXT`, `BLOB`.
* **Affinité de colonne** : `INTEGER`, `TEXT`, `NUMERIC`, `REAL`, `BLOB`. SQLite est **dynamiquement typé** : une colonne à affinité TEXT peut stocker un entier… mais **applique l’affinité** lors des opérations.
* **Tables STRICT** (recommandé quand dispo) : imposent le type déclaré, réduisent les surprises.
* **Dates/heures** : stocker en **TEXT ISO-8601** (`YYYY-MM-DD hh:mm:ss`), ou **INTEGER** (epoch), au choix ; pas de type “DATE” natif.

> Bonne pratique : choisissez une **convention** (ISO-8601 TEXT ou epoch INT) et tenez-vous-y.

---

## 4) Schéma & contraintes d’intégrité

### 4.1 Créer des tables

```sql
CREATE TABLE IF NOT EXISTS client (
  id         INTEGER PRIMARY KEY,   -- alias implicite ROWID
  email      TEXT    NOT NULL UNIQUE,
  nom        TEXT    NOT NULL,
  created_at TEXT    NOT NULL DEFAULT (strftime('%Y-%m-%d %H:%M:%f','now'))
);
```

* `INTEGER PRIMARY KEY` → alias du **ROWID** (clé rapide).
* `UNIQUE`, `NOT NULL`, `CHECK`, `FOREIGN KEY` : **toujours** préférer les contraintes **au plus près des données**.

### 4.2 Clés étrangères

```sql
PRAGMA foreign_keys = ON;           -- à activer à chaque connexion
CREATE TABLE commande (
  id        INTEGER PRIMARY KEY,
  client_id INTEGER NOT NULL REFERENCES client(id) ON DELETE CASCADE,
  total     REAL    NOT NULL CHECK(total >= 0)
);
```

> **Attention** : l’enforcement des **FK** dépend de `PRAGMA foreign_keys=ON` (souvent **off** par défaut selon les bindings).

### 4.3 Colonnes générées (dérivées)

```sql
CREATE TABLE produit (
  id       INTEGER PRIMARY KEY,
  prix_ht  REAL NOT NULL,
  tva      REAL NOT NULL CHECK(tva BETWEEN 0 AND 1),
  prix_ttc REAL GENERATED ALWAYS AS (prix_ht * (1 + tva)) VIRTUAL
);
```

* `VIRTUAL` (calcul à la lecture), `STORED` (stocké). Pratique pour dériver des champs (évite triggers).

---

## 5) Requêtes SQL essentielles

### 5.1 CRUD de base

```sql
-- INSERT
INSERT INTO client (email, nom) VALUES ('a@x.fr', 'Alice');

-- SELECT
SELECT id, email, nom FROM client WHERE email LIKE '%@x.fr' ORDER BY id DESC LIMIT 10 OFFSET 0;

-- UPDATE
UPDATE client SET nom = 'Alice B.' WHERE id = 1;

-- DELETE
DELETE FROM client WHERE id = 1;
```

### 5.2 UPSERT (résoudre les doublons proprement)

```sql
INSERT INTO client (id, email, nom)
VALUES (1, 'a@x.fr', 'Alice')
ON CONFLICT(id) DO UPDATE SET
  email = excluded.email,
  nom   = excluded.nom;
```

### 5.3 Agrégations & groupements

```sql
SELECT client_id, COUNT(*) AS nb_cmd, SUM(total) AS ca
FROM commande
GROUP BY client_id
HAVING SUM(total) > 1000
ORDER BY ca DESC;
```

### 5.4 JOINS

```sql
SELECT c.email, cmd.id, cmd.total
FROM client c
JOIN commande cmd ON cmd.client_id = c.id
WHERE cmd.total >= 50;
```

* Types : `INNER`, `LEFT` (droit utile pour “clients sans commande”), `CROSS`. (Pas de `RIGHT/FULL` natifs → simuler.)

---

## 6) CTE, récursif & fenêtres

### 6.1 CTE (WITH)

```sql
WITH top_clients AS (
  SELECT client_id, SUM(total) ca
  FROM commande
  GROUP BY client_id
  ORDER BY ca DESC
  LIMIT 10
)
SELECT c.id, c.email, t.ca
FROM top_clients t JOIN client c ON c.id = t.client_id;
```

### 6.2 Récursif (ex. arborescence catégories)

```sql
WITH RECURSIVE tree(id, parent_id, lvl) AS (
  SELECT id, parent_id, 0 FROM categorie WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, t.lvl+1
  FROM categorie c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

### 6.3 Fenêtres (window functions)

```sql
SELECT client_id, total,
       ROW_NUMBER() OVER(PARTITION BY client_id ORDER BY total DESC) AS rk
FROM commande;
```

> Super pour classements, LAG/LEAD, moyennes glissantes, etc.

---

## 7) Index & performance

### 7.1 Créer les bons index

```sql
CREATE INDEX IF NOT EXISTS idx_commande_client ON commande(client_id);
CREATE UNIQUE INDEX IF NOT EXISTS idx_client_email ON client(email);
CREATE INDEX IF NOT EXISTS idx_commande_total ON commande(total);
```

* Index = **accélère** `WHERE`/`JOIN`/`ORDER BY` mais **ralentit** les inserts/updates.
* **Cibler** les colonnes **filtrées** ou **jointes** fréquemment.

### 7.2 Index d’expression & partiel

```sql
-- Expression
CREATE INDEX IF NOT EXISTS idx_email_lower ON client( lower(email) );
-- Partiel (si utile) : seulement clients actifs
CREATE INDEX IF NOT EXISTS idx_cmd_total_pos ON commande(total) WHERE total > 0;
```

### 7.3 Lire les plans

```sql
EXPLAIN QUERY PLAN
SELECT * FROM commande WHERE client_id = 42 AND total > 100;
```

> Cherchez les **SCAN** complets non désirés ; ajustez les index.

---

## 8) Transactions & WAL

### 8.1 Transactions

```sql
BEGIN IMMEDIATE;     -- verrouille en écriture dès le début (évite surprise plus tard)
  -- plusieurs INSERT/UPDATE ici
COMMIT;
```

* `DEFERRED` (par défaut) : verrouillage quand nécessaire.
* `IMMEDIATE` : réserve l’écriture tout de suite.
* `EXCLUSIVE` : verrou plus strict.

### 8.2 WAL (Write-Ahead Logging)

```sql
PRAGMA journal_mode = WAL;      -- à faire une fois
PRAGMA synchronous  = NORMAL;   -- compromis perf/sûreté (FULL si durabilité stricte)
```

* **Avantage majeur** : **lectures non bloquées** pendant écriture.
* Penser au **checkpoint** (automatique en général) sur gros fichiers WAL.

### 8.3 Timeouts & busy handlers

```sql
PRAGMA busy_timeout = 5000;     -- attend jusqu’à 5s avant “database is locked”
```

---

## 9) Extensions & tables virtuelles

### 9.1 JSON1 (manipuler du JSON)

```sql
SELECT json_extract(payload, '$.client.email') AS email
FROM evenement
WHERE json_extract(payload, '$.type') = 'signup';
```

* Possibilité d’**indexer** sur une expression JSON :

```sql
CREATE INDEX idx_evt_email ON evenement( json_extract(payload, '$.client.email') );
```

### 9.2 FTS5 (Full-Text Search)

```sql
CREATE VIRTUAL TABLE doc USING fts5(title, body);
INSERT INTO doc (title, body) VALUES ('hello', 'search engines are fun');
SELECT * FROM doc WHERE doc MATCH 'search';
```

* Supporte opérateurs, rang, surlignage.

### 9.3 Autres (selon besoins)

`R*Tree` (spatial simple), `csv` (table virtuelle sur CSV), `dbstat`, etc.

---

## 10) Sécurité & bonnes pratiques

* **Paramétrer** côté app (anti-injection) : `?` (bindings) — **jamais** formater du SQL avec des chaînes.
* **Permissions du fichier** : OS (chmod/ACL), pas d’utilisateurs internes.
* **Chiffrement** : pas natif → utiliser **SQLCipher** ou chiffrer applicativement les colonnes sensibles.
* **Extensions** : chargement d’extensions **désactivé** par défaut dans de nombreux bindings (plus sûr).
* **Backups** : utiliser l’API de backup (`VACUUM INTO`, `.backup`), pas une copie brute en plein write.

---

## 11) Maintenance

* **Intégrité** : `PRAGMA integrity_check;`
* **Compactage** : `VACUUM;` (réduit la taille, réorganise — à faire hors charge)
* **Statistiques** : `ANALYZE;`
* **Migration** : ajouter colonnes ok ; pour des modifs lourdes, recréer table + copier (pattern “shadow table”).

---

## 12) Import/Export (CLI)

```sql
-- Dans le shell sqlite3
.mode csv
.import data.csv ma_table
.headers on
.output export.csv
SELECT * FROM ma_table;
.output stdout
```

* JSON : soit via requêtes `json_object/ json_group_array`, soit côté application.

---

## 13) Intégration Python (aperçu idiomatique)

```python
import sqlite3

conn = sqlite3.connect("app.db", isolation_level=None)  # autocommit off si None ? (selon version)
conn.execute("PRAGMA foreign_keys=ON")
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA synchronous=NORMAL")

with conn:  # transaction
    conn.executemany(
        "INSERT INTO client(email, nom) VALUES(?, ?)",
        [("a@x.fr","Alice"),("b@y.fr","Bob")]
    )

cur = conn.execute("SELECT id, email FROM client WHERE email LIKE ?", ("%@x.%",))
for row in cur:
    print(row)
conn.close()
```

**Points clés** : `with conn` = **transaction** ; `?` = **paramètres** ; `executemany` = **batch**.

---

## 14) Exercices (avec solutions)

### Ex.1 — Schéma & contraintes

**Énoncé** : créez `client(id PK, email UNIQUE NOT NULL, nom NOT NULL)` et `commande(id PK, client_id FK→client(id), total≥0, created_at par défaut maintenant)`. Activez FK.
**Solution** :

```sql
PRAGMA foreign_keys=ON;
CREATE TABLE client (
  id INTEGER PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  nom TEXT NOT NULL
);
CREATE TABLE commande (
  id INTEGER PRIMARY KEY,
  client_id INTEGER NOT NULL REFERENCES client(id) ON DELETE CASCADE,
  total REAL NOT NULL CHECK(total >= 0),
  created_at TEXT NOT NULL DEFAULT (strftime('%Y-%m-%d %H:%M:%f','now'))
);
```

### Ex.2 — Index & requêtes

**Énoncé** : créez l’index qui accélère `SELECT * FROM commande WHERE client_id=? AND total>=? ORDER BY created_at DESC LIMIT 20;`
**Solution** :

```sql
CREATE INDEX IF NOT EXISTS idx_cmd_client_total_date
ON commande(client_id, total, created_at DESC);
```

### Ex.3 — UPSERT

**Énoncé** : insérez un client `(id=1, email='a@x.fr', nom='Alice')`; si déjà là, mettez à jour `email` et `nom`.
**Solution** :

```sql
INSERT INTO client(id, email, nom)
VALUES (1, 'a@x.fr', 'Alice')
ON CONFLICT(id) DO UPDATE SET
  email = excluded.email,
  nom   = excluded.nom;
```

### Ex.4 — CTE + agrégations

**Énoncé** : top 5 des clients par CA (somme `total`).
**Solution** :

```sql
WITH ca AS (
  SELECT client_id, SUM(total) AS s FROM commande GROUP BY client_id
)
SELECT client_id, s FROM ca ORDER BY s DESC LIMIT 5;
```

### Ex.5 — Fenêtres

**Énoncé** : pour chaque client, numeroter ses commandes (ordre `created_at` décroissant).
**Solution** :

```sql
SELECT id, client_id, created_at,
       ROW_NUMBER() OVER(PARTITION BY client_id ORDER BY created_at DESC) AS rk
FROM commande;
```

### Ex.6 — JSON1

**Énoncé** : table `event(payload JSON)`, sortir tous les emails `$.user.email`.
**Solution** :

```sql
SELECT json_extract(payload, '$.user.email') AS email
FROM event
WHERE json_extract(payload,'$.user.email') IS NOT NULL;
```

---

## 15) Règles d’or (résumé)

* **Schéma clair**, contraintes **au plus près** (NOT NULL, CHECK, UNIQUE, FK).
* **WAL** pour scraper/API ou multi-lecteurs, `busy_timeout` non nul.
* **Index ciblés**, `EXPLAIN QUERY PLAN` pour valider.
* **Transactions** en batch, `executemany` côté appli.
* **Sécurité** : paramètres `?`, permissions fichier, chiffrement externe si besoin.
* **Maintenance** : `integrity_check`, `ANALYZE`, `VACUUM`, backups via API.

