## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
   Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=9.727..70.176 rows=499957 loops=1)
      Recheck Cond: (category = 'A'::text)
      Heap Blocks: exact=8334
      ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=8.661..8.661 rows=499957 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.187 ms
    Execution Time: 83.703 ms
   ```
    
    *Объясните результат:*  
    ```
   План: Bitmap Index Scan находит все TID для category='A' (их ~500k), потом Bitmap Heap Scan читает нужные страницы таблицы. 
   Так как половина строк — это ‘A’, селективность низкая, приходится читать много heap-блоков (Heap Blocks: exact=8334). 
   До кластеризации строки ‘A’ и ‘B’ перемешаны, поэтому обращения к heap идут по множеству разных страниц, кэш/диск работают хуже, 
   отсюда большое время (Execution ~83.7 ms).
   ```

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
   [2025-12-16 16:25:55] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2025-12-16 16:25:56] completed in 527 ms
   ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
   Index Scan using test_cluster_cat_idx on test_cluster  (cost=0.42..14591.50 rows=499033 width=39) (actual time=0.097..56.818 rows=499957 loops=1)
      Index Cond: (category = 'A'::text)
    Planning Time: 0.577 ms
    Execution Time: 72.494 ms
   ```
    
    *Объясните результат:*  
    После CLUSTER строки физически упорядочены по индексу category, поэтому строки ‘A’ лежат более компактно. 
    Планировщик выбирает Index Scan (а не bitmap), потому что теперь чтение по индексу приводит к почти последовательному чтению страниц. 
    За счёт лучшей локальности чтения общая стоимость падает: Execution ~72.5 ms. 
    Условие то же, строк столько же, выигрыш именно из-за более “плотного” размещения данных.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*  
    **До**: ~83.7 ms, bitmap + много разбросанных heap-чтений (плохая локальность).  
    **После**: ~72.5 ms, индексный скан с более последовательным чтением (хорошая локальность).   
    **Итог**: кластеризация дала ускорение примерно на 11.2 ms (около 13–14%) для низкоселективного фильтра, потому что уменьшила “случайность” чтения heap-страниц.