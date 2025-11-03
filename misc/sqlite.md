# COURS COMPLET : SQLite avec Python

---

## 1. Introduction à SQLite

SQLite est un moteur de base de données intégré, sans serveur et léger.
Les données sont stockées dans un seul fichier `.db` sur le disque.
C’est une base idéale pour :

* Les petits projets ou prototypes
* Les applications locales
* Les tests ou l’enseignement
* Le web scraping

---

## 2. Vérifier que SQLite est disponible

```python
import sqlite3
print(sqlite3.sqlite_version)
```

### Explication détaillée :

* `import sqlite3` : importe le module intégré à Python qui permet de manipuler SQLite.
* `sqlite3.sqlite_version` : renvoie la version de SQLite utilisée par Python.
* `print(...)` : affiche la version dans le terminal.

SQLite est intégré nativement à Python (aucune installation supplémentaire n’est nécessaire).

---

## 3. Créer une base de données et une table

```python
import sqlite3

# Connexion à la base (créée si inexistante)
conn = sqlite3.connect("livres.db")

# Création du curseur
cur = conn.cursor()

# Création d’une table
cur.execute('''
CREATE TABLE IF NOT EXISTS livres (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    titre TEXT NOT NULL,
    auteur TEXT,
    prix REAL,
    stock INTEGER
)
''')

# Validation et fermeture
conn.commit()
conn.close()
```

### Explication détaillée :

1. `sqlite3.connect("livres.db")` : crée une nouvelle base appelée `livres.db` si elle n’existe pas déjà, ou s’y connecte sinon.
   Ce fichier contient toutes les tables et les données.

2. `conn.cursor()` : crée un objet curseur qui sert d’interface pour exécuter les requêtes SQL.

3. `cur.execute('CREATE TABLE ...')` : envoie la requête SQL à la base pour créer une table.

   * `IF NOT EXISTS` : empêche une erreur si la table existe déjà.
   * `id INTEGER PRIMARY KEY AUTOINCREMENT` : colonne identifiant unique auto-incrémentée.
   * `titre TEXT NOT NULL` : champ texte obligatoire.
   * `prix REAL` : champ numérique à virgule flottante.
   * `stock INTEGER` : champ entier.

4. `conn.commit()` : enregistre définitivement les changements.

5. `conn.close()` : ferme proprement la connexion à la base.

---

## 4. Insérer une donnée dans la table

```python
import sqlite3

conn = sqlite3.connect("livres.db")
cur = conn.cursor()

cur.execute("INSERT INTO livres (titre, auteur, prix, stock) VALUES (?, ?, ?, ?)",
            ("Les Misérables", "Victor Hugo", 12.5, 10))

conn.commit()
conn.close()
```

### Explication détaillée :

* `"INSERT INTO ... VALUES (...)"` : insère une nouvelle ligne dans la table `livres`.
* Les `?` servent de **paramètres sécurisés** appelés *placeholders* pour éviter les injections SQL.
* Les valeurs `("Les Misérables", "Victor Hugo", 12.5, 10)` remplacent les `?` dans l’ordre.
* `conn.commit()` : valide l’insertion.
* `conn.close()` : ferme la base.

---

## 5. Insérer plusieurs lignes à la fois

```python
livres = [
    ("1984", "George Orwell", 9.9, 5),
    ("Le Petit Prince", "Antoine de Saint-Exupéry", 8.5, 20),
    ("Harry Potter", "J.K. Rowling", 15.0, 8)
]

conn = sqlite3.connect("livres.db")
cur = conn.cursor()
cur.executemany("INSERT INTO livres (titre, auteur, prix, stock) VALUES (?, ?, ?, ?)", livres)
conn.commit()
conn.close()
```

### Explication détaillée :

* `livres` : liste de tuples, chaque tuple correspond à une ligne.
* `executemany()` : exécute la même requête pour chaque élément de la liste.
* `commit()` : enregistre les modifications.
* `close()` : ferme la base.

---

## 6. Lire les données

```python
conn = sqlite3.connect("livres.db")
cur = conn.cursor()
cur.execute("SELECT * FROM livres")
resultats = cur.fetchall()

for ligne in resultats:
    print(ligne)

conn.close()
```

### Explication détaillée :

* `"SELECT * FROM livres"` : récupère toutes les colonnes de la table `livres`.
* `fetchall()` : renvoie une liste contenant toutes les lignes sous forme de tuples.
* La boucle `for` parcourt chaque ligne et l’affiche.

Autres méthodes :

* `fetchone()` : récupère une seule ligne.
* `fetchmany(5)` : récupère un nombre défini de lignes.

---

## 7. Filtrer les données avec WHERE

```python
conn = sqlite3.connect("livres.db")
cur = conn.cursor()
cur.execute("SELECT titre, prix FROM livres WHERE prix > ? ORDER BY prix DESC", (10,))
for ligne in cur.fetchall():
    print(ligne)
conn.close()
```

### Explication détaillée :

* `SELECT titre, prix` : ne récupère que les colonnes `titre` et `prix`.
* `WHERE prix > ?` : filtre les lignes où le prix est supérieur à 10.
* `(10,)` : tuple contenant la valeur de remplacement pour `?`.
* `ORDER BY prix DESC` : trie les résultats par prix décroissant.

---

## 8. Modifier et supprimer des données

```python
conn = sqlite3.connect("livres.db")
cur = conn.cursor()

cur.execute("UPDATE livres SET stock = stock - 1 WHERE id = ?", (1,))
cur.execute("DELETE FROM livres WHERE id = ?", (2,))

conn.commit()
conn.close()
```

### Explication détaillée :

* `UPDATE ... SET ... WHERE ...` : modifie des enregistrements existants.
  Ici, on décrémente la quantité du livre dont l’ID est 1.
* `DELETE FROM ... WHERE ...` : supprime le livre dont l’ID est 2.
* `commit()` : sauvegarde les changements.

---

## 9. Utiliser le contexte `with` pour plus de sécurité

```python
import sqlite3

with sqlite3.connect("livres.db") as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM livres")
    for ligne in cur.fetchall():
        print(ligne)
```

### Explication détaillée :

* Le mot-clé `with` crée un bloc de contexte sécurisé.
* Quand le bloc se termine :

  * `conn.close()` est automatiquement appelé.
  * `commit()` est exécuté automatiquement si aucune erreur ne s’est produite.
  * En cas d’erreur, un `rollback()` (annulation) est effectué.

---

## 10. Exporter les données vers un fichier CSV

```python
import csv, sqlite3

with sqlite3.connect("livres.db") as conn:
    cur = conn.cursor()
    cur.execute("SELECT * FROM livres")
    rows = cur.fetchall()

    with open("export.csv", "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["ID", "Titre", "Auteur", "Prix", "Stock"])
        writer.writerows(rows)

print("Export terminé.")
```

### Explication détaillée :

* `import csv` : module pour manipuler des fichiers CSV.
* `cur.execute("SELECT * FROM livres")` : récupère les données à exporter.
* `open("export.csv", "w", newline="", encoding="utf-8")` : ouvre ou crée un fichier CSV en écriture.
* `writer.writerow([...])` : écrit la première ligne (en-têtes).
* `writer.writerows(rows)` : écrit toutes les lignes de la base dans le fichier.
* `newline=""` : évite les sauts de lignes inutiles sous Windows.

---

## 11. Importer un fichier CSV dans la base

```python
import csv, sqlite3

with sqlite3.connect("livres.db") as conn:
    cur = conn.cursor()
    with open("import.csv", "r", encoding="utf-8") as f:
        reader = csv.reader(f)
        next(reader)  # saute la ligne d’en-tête
        cur.executemany("INSERT INTO livres (titre, auteur, prix, stock) VALUES (?, ?, ?, ?)", reader)
    conn.commit()
```

### Explication détaillée :

* `csv.reader(f)` : lit le contenu du fichier CSV ligne par ligne.
* `next(reader)` : ignore la première ligne (les en-têtes).
* `executemany()` : insère chaque ligne du CSV dans la base.
* `commit()` : valide les insertions.

---

## 12. Créer une fonction Python réutilisable dans SQLite

```python
def tva(prix):
    return round(prix * 1.2, 2)

conn = sqlite3.connect("livres.db")
conn.create_function("TTC", 1, tva)
cur = conn.cursor()

cur.execute("SELECT titre, TTC(prix) FROM livres")
for row in cur.fetchall():
    print(row)
```

### Explication détaillée :

* `def tva(prix)` : définit une fonction Python qui calcule le prix TTC (20 % de TVA).
* `conn.create_function("TTC", 1, tva)` : déclare la fonction Python `tva()` comme fonction SQL utilisable dans les requêtes.
  Le `1` signifie que la fonction prend un seul argument.
* `"SELECT titre, TTC(prix)"` : exécute une requête SQL appelant la fonction personnalisée.

---

## 13. Gestion des erreurs et exceptions

```python
import sqlite3

try:
    conn = sqlite3.connect("livres.db")
    cur = conn.cursor()
    cur.execute("SELECT * FROM livres")
except sqlite3.Error as e:
    print("Erreur SQLite :", e)
finally:
    conn.close()
```

### Explication détaillée :

* Le bloc `try` contient le code principal susceptible de générer une erreur.
* `except sqlite3.Error as e` capture toutes les erreurs liées à SQLite et affiche un message d’erreur.
* `finally` : s’exécute toujours, qu’il y ait une erreur ou non.
  Il est utilisé ici pour fermer la connexion.

---

## 14. Accéder aux résultats sous forme de dictionnaires

```python
conn = sqlite3.connect("livres.db")
conn.row_factory = sqlite3.Row
cur = conn.cursor()
cur.execute("SELECT * FROM livres")
for row in cur.fetchall():
    print(dict(row))
conn.close()
```

### Explication détaillée :

* `conn.row_factory = sqlite3.Row` : transforme chaque ligne en un objet de type `Row` (clé/valeur).
* `dict(row)` : convertit cet objet en dictionnaire Python.
  Exemple de sortie :

  ```python
  {'id': 1, 'titre': '1984', 'auteur': 'George Orwell', 'prix': 9.9, 'stock': 5}
  ```

