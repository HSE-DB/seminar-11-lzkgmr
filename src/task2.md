## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
   Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.082..0.086 rows=1 loops=1)
      Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
      Heap Blocks: exact=1
      ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.058..0.059 rows=1 loops=1)
            Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)
    Planning Time: 1.060 ms
    Execution Time: 0.154 ms
    ```
    
    *Объясните результат:*  
    Используется Bitmap Index Scan по GIN-индексу t_books_fts_idx, 
    потому что условие to_tsvector(...) @@ to_tsquery(...) прямо соответствует индексу. 
    Дальше Bitmap Heap Scan читает нужные heap-страницы и делает Recheck Cond — для GIN это нормально: 
    индекс возвращает кандидатов, а точное совпадение проверяется на строке. 
    Heap Blocks: exact=1 значит, что реально прочитали всего одну страницу таблицы. Нашлась 1 строка, поэтому очень быстро.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
    Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.030..0.031 rows=1 loops=1)
      Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 0.373 ms
    Execution Time: 0.060 ms
    ```
     
     *Объясните результат:*  
     PRIMARY KEY создаёт B-tree индекс t_lookup_pk, поэтому план — Index Scan. 
     По условию равенства берётся ровно одна запись (rows=1), чтение минимальное, потому что индекс сразу ведёт к нужной строке. 
     Быстро и предсказуемо.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
    Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.177..0.178 rows=1 loops=1)
      Index Cond: ((item_key)::text = '0000000455'::text)
    Planning Time: 0.384 ms
    Execution Time: 0.229 ms
    ```
     
     *Объясните результат:*  
     План тот же (Index Scan по PK), потому что для точечного поиска по ключу кластеризация не меняет выбор плана: 
     всё равно идём через B-tree к конкретному TID. Отличие во времени здесь не про “кластеризацию лучше/хуже”, а про физику:
     в конкретном прогоне нужная страница могла оказаться не в кеше (или требовала больше I/O), поэтому actual time выше. 
     В целом для одиночного равенства кластеризация обычно почти не даёт выигрыша.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
    Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.058..0.058 rows=0 loops=1)
      Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.264 ms
    Execution Time: 0.077 ms
    ```
     
     *Объясните результат:*  
     Есть B-tree индекс t_lookup_value_idx, поэтому снова Index Scan: индекс быстро проверяет, есть ли ключ 'T_BOOKS'. 
     Результата нет (rows=0), но это всё равно дёшево: скан индекса на равенство быстрый, и к heap почти не приходится обращаться.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
    Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.049..0.049 rows=0 loops=1)
      Index Cond: ((item_value)::text = 'T_BOOKS'::text)
    Planning Time: 0.312 ms
    Execution Time: 0.079 ms
    ```
     
     *Объясните результат:*  
     Аналогично: Index Scan по t_lookup_clustered_value_idx, результат не найден (rows=0).
     Кластеризация по первичному ключу тут почти не влияет, потому что поиск идёт по отдельному индексу на item_value, и строк всё равно нет.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*  
     Обе таблицы показывают почти одинаковую производительность (~0.077–0.079 ms), потому что: 
    - запрос ищет по item_value через B-tree индекс, а кластеризация выполнена по PK (не по item_value), значит физический порядок строк не помогает;
    - совпадений нет, поэтому обход ограничивается в основном индексом, без заметного чтения heap.   
    Выигрыш от CLUSTER проявился бы скорее в запросах, которые возвращают много строк и читают их “пачками” в порядке кластерного индекса (например, диапазоны по item_key), а не в точечном равенстве по другому столбцу.