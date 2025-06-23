# Exercice 2: Requêtes avec filtres multiples

### 2.1 Requête avec conditions multiples
```sql
EXPLAIN ANALYZE
SELECT *
FROM title_basics
WHERE title_type = 'movie' AND start_year = 1950;
