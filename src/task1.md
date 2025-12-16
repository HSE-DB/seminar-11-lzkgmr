# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*  
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.023..0.023 rows=0 loops=1)
     Recheck Cond: (category IS NULL)
     ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.018..0.018 rows=0 loops=1)
           Index Cond: (category IS NULL)
   Planning Time: 0.425 ms
   Execution Time: 0.149 ms
   ```  

   
   *Объясните результат:*  
   BRIN хранит диапазонные сводки по страницам (min/max и т.п.), поэтому планировщик сначала берёт подходящие диапазоны через Bitmap Index Scan, 
   затем читает соответствующие heap-страницы через Bitmap Heap Scan и делает Recheck Cond (BRIN неточный, нужно перепроверять строки). 
   В результате строк нет (rows=0), поэтому всё заканчивается очень быстро.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

   7. Выполните поиск по категории и автору:
      ```sql
      EXPLAIN ANALYZE
      SELECT * FROM t_books 
      WHERE category = 'INDEX' AND author = 'SYSTEM';
      ```
   
      *План выполнения:*
      ```
      Bitmap Heap Scan on t_books  (cost=12.17..2413.62 rows=1 width=33) (actual time=13.958..13.959 rows=0 loops=1)
        Recheck Cond: ((category)::text = 'INDEX'::text)
        Rows Removed by Index Recheck: 150000
        Filter: ((author)::text = 'SYSTEM'::text)
        Heap Blocks: lossy=1225
        ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.17 rows=78430 width=0) (actual time=0.110..0.110 rows=12250 loops=1)
              Index Cond: ((category)::text = 'INDEX'::text)
      Planning Time: 0.278 ms
      Execution Time: 13.991 ms
      ```
   
      *Объясните результат (обратите внимание на bitmap scan):*  
      Используется только BRIN по category, потому что он даёт грубый отбор страниц. BRIN возвращает много “кандидатных” страниц, 
      поэтому Postgres строит bitmap, чтобы не ходить в heap по одной строке (это дешевле при большом числе совпадений).
      Heap Blocks: lossy=1225 означает, что bitmap “потерял точность” и хранит отметку “вся страница подходит”, 
      а не конкретные TID — поэтому появляется Rows Removed by Index Recheck: 150000 
      (при чтении страниц пришлось перепроверить много строк и почти всё отсеялось). 
      author='SYSTEM' применён как Filter, потому что индекс (в этом плане) по author не использовался, 
      и условие проверяется уже после чтения строк.

      8. Получите список уникальных категорий:
         ```sql
         EXPLAIN ANALYZE
         SELECT DISTINCT category 
         FROM t_books 
         ORDER BY category;
         ```
   
         *План выполнения:*
      ```   
      Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=21.869..21.871 rows=6 loops=1)
        Sort Key: category
        Sort Method: quicksort  Memory: 25kB
        ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=21.838..21.840 rows=6 loops=1)
              Group Key: category
              Batches: 1  Memory Usage: 24kB
              ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.150..6.355 rows=150000 loops=1)
      Planning Time: 1.841 ms
      Execution Time: 21.948 ms
      ```
   
      *Объясните результат:*   
      Уникальных категорий мало (в плане rows=6), но чтобы гарантированно найти все значения, нужно просмотреть всю таблицу -
      поэтому Seq Scan. Далее HashAggregate собирает DISTINCT по category, а Sort упорядочивает результат. 
      Индекс не помогает, потому что   
      а) BRIN не выдаёт значения “по порядку”,   
      б) дешевле один раз прочитать таблицу, чем делать множество выборок ради 6 значений.  

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ``` 
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=9.344..9.344 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=9.338..9.338 rows=0 loops=1)
           Filter: ((author)::text ~~ 'S%'::text)
           Rows Removed by Filter: 150000
   Planning Time: 1.090 ms
   Execution Time: 9.381 ms

   ```
   
   *Объясните результат:*   
   BRIN по author для префиксного поиска обычно плохо селективен (значения не “кластеризованы” по страницам), 
   поэтому планировщик выбирает Seq Scan. По факту совпадений нет (rows=0), значит все 150000 строк “удалены фильтром” 
   (Rows Removed by Filter: 150000)

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ``` 
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=37.705..37.706 rows=1 loops=1)
     ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=37.697..37.698 rows=1 loops=1)
           Filter: (lower((title)::text) ~~ 'o%'::text)
           Rows Removed by Filter: 149999
   Planning Time: 0.351 ms
   Execution Time: 37.736 ms

   ```
   
   *Объясните результат:*  
   Чтобы B-tree индекс использовался, условие должно быть индексируемым по правилам оператора и collation. 
   В DataGrip/PG часто LIKE (особенно с локалью/не-C collation) не даёт планировщику применить индекс для префикса, и он выбирает Seq Scan. 
   Плюс оценка селективности (rows=750) могла быть неточной, а реальных совпадений почти нет (rows=1).

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.17..2413.62 rows=1 width=33) (actual time=1.584..1.584 rows=0 loops=1)
     Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
     Rows Removed by Index Recheck: 8855
     Heap Blocks: lossy=73
     ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.17 rows=78430 width=0) (actual time=0.029..0.029 rows=730 loops=1)
           Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.308 ms
   Execution Time: 1.611 ms
   ```
   
   *Объясните результат:*  
   Теперь индекс отбирает страницы сразу по двум условиям (Index Cond включает оба), 
   поэтому кандидатов намного меньше: lossy=73 вместо 1225 и значительно меньше перепроверок (Rows Removed by Index Recheck: 8855). 
   Bitmap всё ещё lossy (BRIN всё ещё неточный), 
   но объём heap-чтений и recheck резко упал, поэтому Execution Time снизилось примерно с ~14 ms до ~1.6 ms.