# Exercice 2: Requêtes avec filtres multiples

### 2.1 Requête avec conditions multiples
```sql
EXPLAIN ANALYZE
SELECT *
FROM title_basics
WHERE title_type = 'movie' AND start_year = 1950;
```

* Output:
```sql
| QUERY PLAN |
| :--- |
| Gather  \(cost=1000.00..248242.17 rows=733 width=84\) \(actual time=11.236..504.468 rows=2009 loops=1\) |
|   Workers Planned: 2 |
|   Workers Launched: 2 |
|   -&gt;  Parallel Seq Scan on title\_basics  \(cost=0.00..247168.87 rows=305 width=84\) \(actual time=10.678..468.861 rows=670 loops=3\) |
|         Filter: \(\(\(title\_type\)::text = 'movie'::text\) AND \(start\_year = 1950\)\) |
|         Rows Removed by Filter: 3910950 |
| Planning Time: 0.110 ms |
| JIT: |
|   Functions: 6 |
|   Options: Inlining false, Optimization false, Expressions true, Deforming true |
|   Timing: Generation 1.598 ms \(Deform 0.773 ms\), Inlining 0.000 ms, Optimization 1.152 ms, Emission 15.145 ms, Total 17.895 ms |
| Execution Time: 504.959 ms |
```
