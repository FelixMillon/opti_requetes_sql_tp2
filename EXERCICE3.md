# EXERCICE 3



## 3.1 Jointure avec filtres
EXPLAIN ANALYZE
SELECT tb.primary_title, tr.average_rating
FROM title_basics tb
JOIN title_ratings tr ON tb.tconst = tr.tconst
WHERE tb.startYear = 1994 AND tr.average_rating > 8.5;


"Gather  (cost=120749.99..139323.09 rows=745 width=26) (actual time=125.582..224.890 rows=735 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Parallel Hash Join  (cost=119749.99..138248.59 rows=310 width=26) (actual time=105.892..198.136 rows=245 loops=3)"
"        Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"
"        ->  Parallel Seq Scan on title_ratings tr  (cost=0.00..18332.39 rows=63321 width=16) (actual time=0.127..80.044 rows=50463 loops=3)"
"              Filter: (average_rating > 8.5)"
"              Rows Removed by Filter: 476666"
"        ->  Parallel Hash  (cost=119450.50..119450.50 rows=23959 width=30) (actual time=100.532..100.534 rows=22988 loops=3)"
"              Buckets: 131072 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 6016kB"
"              ->  Parallel Bitmap Heap Scan on title_basics tb  (cost=642.07..119450.50 rows=23959 width=30) (actual time=12.453..84.433 rows=22988 loops=3)"
"                    Recheck Cond: (startyear = 1994)"
"                    Heap Blocks: exact=9790"
"                    ->  Bitmap Index Scan on idx_title_basics_startyear  (cost=0.00..627.69 rows=57501 width=0) (actual time=7.102..7.103 rows=68964 loops=1)"
"                          Index Cond: (startyear = 1994)"
"Planning Time: 0.256 ms"
"JIT:"
"  Functions: 45"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 2.457 ms (Deform 1.236 ms), Inlining 0.000 ms, Optimization 1.639 ms, Emission 24.749 ms, Total 28.845 ms"
"Execution Time: 225.699 ms"

## 3.2 Analyse du plan de jointure
### 1. Quel algorithme de jointure est utilisé?:
On voit qu'il utilise le Parallel Hash 

### 2. Comment l'index sur start_Year est-il utilisé?
Bitmap Index Scan on idx_title_basics_startyear
Index Cond: (startyear = 1994)
Postgre créer un bitmap avec le champs indexé. (valeur 1994)
Ainsi il limite rapidement le nombre de champs à traiter.

### 3. Comment est traitée la condition sur average_Rating?
"        ->  Parallel Seq Scan on title_ratings tr  (cost=0.00..18332.39 rows=63321 width=16) (actual time=0.127..80.044 rows=50463 loops=3)"
"              Filter: (average_rating > 8.5)"
average_rating est traitée sequentiellement.

### 4. Pourquoi PostgreSQL utilise-t-il le parallélisme?
Les tables sont grandes. Tout executer en sequentiel serait trop couteux.


## 3.3 Indexation de la seconde condition
CREATE INDEX idx_title_ratings_average_rating ON title_ratings(average_rating);

## 3.4 Analyse après indexation
"Hash Join  (cost=123438.75..135833.31 rows=745 width=26) (actual time=177.258..279.510 rows=735 loops=1)"
"  Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"
"  ->  Bitmap Heap Scan on title_ratings tr  (cost=2850.20..14845.84 rows=151971 width=16) (actual time=10.934..69.536 rows=151388 loops=1)"
"        Recheck Cond: (average_rating > 8.5)"
"        Heap Blocks: exact=9715"
"        ->  Bitmap Index Scan on idx_title_ratings_average_rating  (cost=0.00..2812.21 rows=151971 width=0) (actual time=9.290..9.292 rows=151388 loops=1)"
"              Index Cond: (average_rating > 8.5)"
"  ->  Hash  (cost=119869.78..119869.78 rows=57501 width=30) (actual time=164.892..164.896 rows=68964 loops=1)"
"        Buckets: 131072 (originally 65536)  Batches: 1 (originally 1)  Memory Usage: 5201kB"
"        ->  Bitmap Heap Scan on title_basics tb  (cost=642.07..119869.78 rows=57501 width=30) (actual time=16.490..133.791 rows=68964 loops=1)"
"              Recheck Cond: (startyear = 1994)"
"              Heap Blocks: exact=20823"
"              ->  Bitmap Index Scan on idx_title_basics_startyear  (cost=0.00..627.69 rows=57501 width=0) (actual time=4.146..4.147 rows=68964 loops=1)"
"                    Index Cond: (startyear = 1994)"
"Planning Time: 6.098 ms"
"JIT:"
"  Functions: 15"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 0.720 ms (Deform 0.393 ms), Inlining 0.000 ms, Optimization 0.632 ms, Emission 7.794 ms, Total 9.146 ms"
"Execution Time: 280.474 ms"

Le parallelisme a disparu.
L'index average_rating est utilisé
Le temps d'execution est plus long

## 3.5 Analyse de l'impact

### 1. L'algorithme de jointure a-t-il changé?
Il s'agit toujours d'un Hash join, mais non parallelisé.
### 2. Comment l'index sur average_Rating est-il utilisé?
l'index sur average_rating est utilisé pour faire un bitmap et un heap pour filtrer les données.
### 3. Le temps d'exécution s'est-il amélioré? Pourquoi?
Non: avec l'ajout d'un nouvel index, les données peuvent être filtrées suffisamment pour que le parallelisme ne soit pas utile (rentable ?).
Resultat : c'est un peu plus long, mais ça doit moins demander au cpu.
### 4. Pourquoi PostgreSQL abandonne-t-il le parallélisme?
Le parallelisme offre des gains sur de très grandes quantitées de données. Une fois les deux filtres appliqués, il est possible que le parallelisme ne soit pas plus efficace que le calcule sequentiel.

