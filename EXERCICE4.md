# EXERCICE 4

## 4.1 Requête complexe
EXPLAIN ANALYZE
SELECT
    tb.startYear,
    COUNT(*) AS nombre_films,
    AVG(tr.average_rating) AS moyenne_notes
FROM title_basics tb
JOIN title_ratings tr ON tb.tconst = tr.tconst
WHERE tb.startYear BETWEEN 1990 AND 2000
  AND tb.title_type = 'movie'
GROUP BY tb.startYear
ORDER BY moyenne_notes DESC;

"Sort  (cost=275667.30..275667.63 rows=131 width=44) (actual time=793.438..800.537 rows=11 loops=1)"
"  Sort Key: (avg(tr.average_rating)) DESC"
"  Sort Method: quicksort  Memory: 25kB"
"  ->  Finalize GroupAggregate  (cost=275596.61..275662.70 rows=131 width=44) (actual time=791.055..800.502 rows=11 loops=1)"
"        Group Key: tb.startyear"
"        ->  Gather Merge  (cost=275596.61..275658.44 rows=262 width=44) (actual time=790.705..800.409 rows=33 loops=1)"
"              Workers Planned: 2"
"              Workers Launched: 2"
"              ->  Partial GroupAggregate  (cost=274596.59..274628.18 rows=131 width=44) (actual time=766.709..769.455 rows=11 loops=3)"
"                    Group Key: tb.startyear"
"                    ->  Sort  (cost=274596.59..274604.08 rows=2995 width=10) (actual time=766.348..767.290 rows=10346 loops=3)"
"                          Sort Key: tb.startyear"
"                          Sort Method: quicksort  Memory: 720kB"
"                          Worker 0:  Sort Method: quicksort  Memory: 703kB"
"                          Worker 1:  Sort Method: quicksort  Memory: 701kB"
"                          ->  Parallel Hash Join  (cost=256008.90..274423.65 rows=2995 width=10) (actual time=594.568..763.569 rows=10346 loops=3)"
"                                Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"
"                                ->  Parallel Seq Scan on title_ratings tr  (cost=0.00..16685.11 rows=658911 width=16) (actual time=0.034..51.532 rows=527129 loops=3)"
"                                ->  Parallel Hash  (cost=255731.08..255731.08 rows=22226 width=14) (actual time=591.241..591.243 rows=17113 loops=3)"
"                                      Buckets: 65536  Batches: 1  Memory Usage: 2976kB"
"                                      ->  Parallel Bitmap Heap Scan on title_basics tb  (cost=11279.50..255731.08 rows=22226 width=14) (actual time=73.081..581.861 rows=17113 loops=3)"
"                                            Recheck Cond: ((startyear >= 1990) AND (startyear <= 2000))"
"                                            Rows Removed by Index Recheck: 1307723"
"                                            Filter: ((title_type)::text = 'movie'::text)"
"                                            Rows Removed by Filter: 269309"
"                                            Heap Blocks: exact=16949 lossy=23533"
"                                            ->  Bitmap Index Scan on idx_title_basics_startyear  (cost=0.00..11266.16 rows=841373 width=0) (actual time=64.415..64.415 rows=859264 loops=1)"
"                                                  Index Cond: ((startyear >= 1990) AND (startyear <= 2000))"
"Planning Time: 0.437 ms"
"JIT:"
"  Functions: 63"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 4.341 ms (Deform 1.766 ms), Inlining 0.000 ms, Optimization 2.142 ms, Emission 44.314 ms, Total 50.798 ms"
"Execution Time: 801.558 ms"

## 4.2 Analyse du plan complexe
### 1. Identifiez les différentes étapes du plan (scan, hash, agrégation, tri)
Scan (lecture des données):
    Bitmap Index Scan sur idx_title_basics_startyear
    Parallel Bitmap Heap Scan sur title_basics
    Parallel Seq Scan sur title_ratings

Hash (jointure) :
    Parallel Hash sur la table title_bascis
    Parallel Hash Join entre title_basics et title_ratings (efficace pour gros volumes).
Agrégation :
    Partial GroupAggregate : agrégation partielle parallèle groupée par tb.startyear.
    Gather Merge (fusion des resultats partiels)
    Finalize GroupAggregate : agrégation finale pour une seule moyenne
Tri final :
    quicksort : pour avoir l'ordre décroissant
### 2. Pourquoi l'agrégation est-elle réalisée en deux phases ("Partial" puis "Finalize")?
Partial GroupAggregate : chaque worker va gérer une partie des données en parallèle.
Finalize GroupAggregate : combine les resultats obtenus par chaque worker
### 3. Comment sont utilisés les index existants?
index sur startyear : pour récupérer rapidement toutes les lignes qui sont entre 1990 et 2000
l'index sur average_rating n'est pas utilisé
### 4. Le tri final est-il coûteux? Pourquoi?
Non, il n'est pas couteux: 
"Sort  (cost=275667.30..275667.63 rows=131 width=44) (actual time=793.438..800.537 rows=11 loops=1)"
"  Sort Key: (avg(tr.average_rating)) DESC"
"  Sort Method: quicksort  Memory: 25kB"
on voit :
qu'il ne concerne que 11 lignes 
il a un cout de 800 (le 1er bitmap a un coup de 11266)
qu'il ne prend que 25kB en mémoire.

## 4.3 Indexation des colonnes de jointure
CREATE INDEX idx_title_basics_tconst ON title_basics (tconst);
CREATE INDEX idx_title_ratings_tconst ON title_ratings (tconst);

### 4.4 Analyse après indexation
"Sort  (cost=275667.30..275667.63 rows=131 width=44) (actual time=1394.035..1407.688 rows=11 loops=1)"
"  Sort Key: (avg(tr.average_rating)) DESC"
"  Sort Method: quicksort  Memory: 25kB"
"  ->  Finalize GroupAggregate  (cost=275596.61..275662.70 rows=131 width=44) (actual time=1392.271..1407.653 rows=11 loops=1)"
"        Group Key: tb.startyear"
"        ->  Gather Merge  (cost=275596.61..275658.44 rows=262 width=44) (actual time=1392.015..1407.607 rows=33 loops=1)"
"              Workers Planned: 2"
"              Workers Launched: 2"
"              ->  Partial GroupAggregate  (cost=274596.59..274628.18 rows=131 width=44) (actual time=1355.448..1357.134 rows=11 loops=3)"
"                    Group Key: tb.startyear"
"                    ->  Sort  (cost=274596.59..274604.08 rows=2995 width=10) (actual time=1355.208..1355.869 rows=10346 loops=3)"
"                          Sort Key: tb.startyear"
"                          Sort Method: quicksort  Memory: 711kB"
"                          Worker 0:  Sort Method: quicksort  Memory: 696kB"
"                          Worker 1:  Sort Method: quicksort  Memory: 716kB"
"                          ->  Parallel Hash Join  (cost=256008.90..274423.65 rows=2995 width=10) (actual time=1208.632..1352.481 rows=10346 loops=3)"
"                                Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"
"                                ->  Parallel Seq Scan on title_ratings tr  (cost=0.00..16685.11 rows=658911 width=16) (actual time=0.043..45.598 rows=527129 loops=3)"
"                                ->  Parallel Hash  (cost=255731.08..255731.08 rows=22226 width=14) (actual time=1205.108..1205.109 rows=17113 loops=3)"
"                                      Buckets: 65536  Batches: 1  Memory Usage: 2976kB"
"                                      ->  Parallel Bitmap Heap Scan on title_basics tb  (cost=11279.50..255731.08 rows=22226 width=14) (actual time=58.848..1195.238 rows=17113 loops=3)"
"                                            Recheck Cond: ((startyear >= 1990) AND (startyear <= 2000))"
"                                            Rows Removed by Index Recheck: 1307723"
"                                            Filter: ((title_type)::text = 'movie'::text)"
"                                            Rows Removed by Filter: 269309"
"                                            Heap Blocks: exact=16940 lossy=23663"
"                                            ->  Bitmap Index Scan on idx_title_basics_startyear  (cost=0.00..11266.16 rows=841373 width=0) (actual time=69.059..69.059 rows=859264 loops=1)"
"                                                  Index Cond: ((startyear >= 1990) AND (startyear <= 2000))"
"Planning Time: 1.570 ms"
"JIT:"
"  Functions: 63"
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 4.446 ms (Deform 1.687 ms), Inlining 0.000 ms, Optimization 2.202 ms, Emission 35.538 ms, Total 42.186 ms"
"Execution Time: 1409.359 ms"

AVANT:
"                          ->  Parallel Hash Join  (cost=256008.90..274423.65 rows=2995 width=10) (actual time=1208.632..1352.481 rows=10346 loops=3)"
"                                Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"
APRES
"                          ->  Parallel Hash Join  (cost=256008.90..274423.65 rows=2995 width=10) (actual time=594.568..763.569 rows=10346 loops=3)"
"                                Hash Cond: ((tr.tconst)::text = (tb.tconst)::text)"

Pas de changement majeur avec les indexes sur tconst

## .5 Analyse des résultats
### 1. Les index de jointure sont-ils utilisés? Pourquoi?
Non. pour quelques miliers de lignes (ici 2995), le sequenciel est moins couteux
### 2. Pourquoi le plan d'exécution reste-t-il pratiquement identique?
La seule nouveauté venant des index, et ceux ci n'apportant rien dans ce cas là : pas de changement.
### 3. Dans quels cas les index de jointure seraient-ils plus efficaces?
Si les tables d'avaient pas pu être filtrée en amont, et que le nombre de ligne à traité aurait été plus grand