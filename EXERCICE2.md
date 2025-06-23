# Exercice 2: Requêtes avec filtres multiples

### 2.1 Requête avec conditions multiples
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
