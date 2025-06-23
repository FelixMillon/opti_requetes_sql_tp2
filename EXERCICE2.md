# Exercice 2: Requêtes avec filtres multiples

### 2.1 Requête avec conditions multiples
```sql
EXPLAIN ANALYZE
SELECT *
FROM title_basics
WHERE title_type = 'movie' AND start_year = 1950;

![image](https://github.com/user-attachments/assets/3f4dffaf-8ecb-484d-8e40-8835258b876f)
