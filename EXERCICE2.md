# Exercice 2: Requêtes avec filtres multiples

### 2.1 Requête avec conditions multiples

* Request
```sql
EXPLAIN ANALYZE
SELECT *
FROM title_basics
WHERE title_type = 'movie' AND start_year = 1950;
```

* Output:

| QUERY PLAN |
| :--- |
| Bitmap Heap Scan on title\_basics  \(cost=128.63..37010.34 rows=733 width=84\) \(actual time=0.690..6.011 rows=2009 loops=1\) |
|   Recheck Cond: \(start\_year = 1950\) |
|   Filter: \(\(title\_type\)::text = 'movie'::text\) |
|   Rows Removed by Filter: 6268 |
|   Heap Blocks: exact=3275 |
|   -&gt;  Bitmap Index Scan on idx\_title\_basics\_start\_year  \(cost=0.00..128.45 rows=11735 width=0\) \(actual time=0.341..0.342 rows=8277 loops=1\) |
|         Index Cond: \(start\_year = 1950\) |
| Planning Time: 0.108 ms |
| Execution Time: 6.096 ms |

### 2.2 Analyse du plan d'exécution
**_Quelle stratégie est utilisée pour le filtre sur start_Year?_**

    La stratégie qui est utilisée pour le filtre sur start_Year est une Bitmap Index Scan sur l'index idx_title_basics_start_year.

**_1. Comment est traité le filtre sur title_Type?_**

    Le filtre sur title_type = 'movie' est appliqué comme un Filter supplémentaire après le Bitmap Heap Scan

    Ce n'est pas utilisé dans l'index scan (ce qui est inefficace)

    Cela explique pourquoi 6268 lignes sont rejetées après avoir passé le filtre start_year

**_2. Combien de lignes passent le premier filtre, puis le second?_**

    Premier filtre (start_year) : 8277 lignes satisfont start_year = 1950 (rows=8277 dans le Bitmap Index Scan)

    Second filtre (title_type) : Seulement 2009 lignes satisfont les deux conditions (rows=2009 dans le Bitmap Heap Scan)

**_3. Quelles sont les limitations de notre index actuel?_**

    L'index existant (idx_title_basics_start_year) ne couvre que start_year

    PostgreSQL doit donc:

        - Utiliser l'index pour start_year

        - Charger toutes ces lignes en mémoire (Heap Blocks: exact=3275)

        - Filtrer manuellement pour title_type

    Cela est inefficace car 75% des lignes (6,268/8,277) sont rejetées après avoir été chargées

### 2.3 Index composite
```sql
CREATE INDEX idx_title_basics_type_year ON title_basics(title_type, start_year);
```

### 2.4 Analyse après index composite
* Output

| QUERY PLAN |
| :--- |
| Index Scan using idx\_title\_basics\_type\_year on title\_basics  \(cost=0.43..2344.94 rows=733 width=84\) \(actual time=0.043..1.873 rows=2009 loops=1\) |
|   Index Cond: \(\(\(title\_type\)::text = 'movie'::text\) AND \(start\_year = 1950\)\) |
| Planning Time: 0.127 ms |
| Execution Time: 1.977 ms |

* Analyse

Avant (avec index simple sur start_year) :

    Type de scan : Bitmap Heap Scan + Bitmap Index Scan

    Temps d'exécution : 6.096 ms

    Lignes examinées : 8,277 (dont 6,268 rejetées)

    Blocs lus : 3,275

Après (avec index composite (title_type, start_year)) :

    Type de scan : Index Scan direct (plus efficace)

    Temps d'exécution : 1.977 ms (3x plus rapide)

    Lignes examinées : exactement 2,089 (aucune rejetée)

    Pas de blocs superflus lus

Points clés d'amélioration :

    Changement radical de stratégie :

        Passage d'un Bitmap Scan à un Index Scan pur

        Plus besoin de vérification supplémentaire (Recheck Cond disparaît)

    Filtrage intégré dans l'index :

        La condition combinée title_type = 'movie' AND start_year = 1950 est évaluée directement dans l'index

        Visible dans la clause Index Cond du nouveau plan

    Gain de performance :

        Réduction de 67% du temps d'exécution (6.096 ms → 1.977 ms)

        Élimination du filtrage post-scan (0 ligne rejetée vs 6,268 avant)

    Optimisation des E/S :

        Lecture de seulement les blocs nécessaires

        Plus de lecture superflue de 3,275 blocs

### 2.5 Impact du nombre de colonnes
* Changed request
```sql
EXPLAIN ANALYZE
SELECT tconst, primary_title, start_year, title_type
FROM title_basics
WHERE title_type = 'movie' AND start_year = 1950;
```

Comparez les performances avec la requête de l'étape 2.4:

**_1. Le temps d'exécution a-t-il changé?_**

oui, (1.977 ms → 1.595 ms)

**_2. Pourquoi cette optimisation est-elle plus ou moins efficace que dans l'exercice 1?_**

Elle est moins efficace ici car on utilise déjà un index composite très performant. Dans l’exercice 1, l’indexation améliorait fortement les performances car on partait d’un scan séquentiel. Ici, l’essentiel du gain avait déjà été obtenu à l’étape 2.4.

**_3. Dans quel cas un "covering index" (index qui contient toutes les colonnes de la requête) serait idéal?_**

Un covering index est utile quand l’index contient toutes les colonnes nécessaires à la requête. Dans ce cas, PostgreSQL peut répondre sans lire la table, ce qui accélère encore l’exécution.

### 2.6 Analyse de l'amélioration globale
**_1. Quelle est la différence de temps d'exécution par rapport à l'étape 2.1?_**

Le temps d'exécution est passé d’environ 6 ms à moins de 2 ms, soit une réduction d’environ 67 %. Ce gain est principalement dû à l’utilisation d’un index composite qui permet un accès plus direct et plus sélectif aux lignes correspondantes.

**_2. Comment l'index composite modifie-t-il la stratégie?_**

L’index composite permet à PostgreSQL d’utiliser un Index Scan au lieu d’un Bitmap Heap Scan, ce qui évite de charger en mémoire un grand nombre de lignes à filtrer. La condition combinée title_type = 'movie' AND start_year = 1950 est directement appliquée via l’index.

**_3. Pourquoi le nombre de blocs lus ("Heap Blocks") a-t-il diminué?_**

Parce que l’Index Scan lit uniquement les lignes correspondant aux deux filtres. Contrairement au plan initial, PostgreSQL n’a plus besoin de lire toutes les lignes correspondant à start_year = 1950 pour ensuite filtrer title_type, ce qui réduit fortement les lectures inutiles.

**_4. Dans quels cas un index composite est-il particulièrement efficace?_**

Un index composite est très utile quand plusieurs colonnes sont utilisées ensemble dans une clause WHERE, en particulier avec des conditions d’égalité. Il permet de réduire les lectures disque et d’accélérer les requêtes filtrées sur ces colonnes.
