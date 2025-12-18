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
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=25.358..112.001 rows=500108 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=24.150..24.150 rows=500108 loops=1)
    Index Cond: (category = 'A'::text)
    Planning Time: 0.905 ms
    Execution Time: 124.553 ms
    ```
    
    *Объясните результат:*
    Индекс скан нашел половину таблицы, поэтому хип скану пришлось читать много страниц и перепроверять строки, отсюда время выполнения 124ms. 

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    [2025-12-19 01:39:18] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2025-12-19 01:39:18] completed in 614 ms
    ```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=5545.98..20095.39 rows=497233 width=39) (actual time=15.530..68.060 rows=500108 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4168
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5421.67 rows=497233 width=0) (actual time=15.059..15.059 rows=500108 loops=1)
    Index Cond: (category = 'A'::text)
    Planning Time: 1.538 ms
    Execution Time: 81.655 ms
    ```
    
    *Объясните результат:*
    Видим, что план такой же, но в 2 раза уменьшилось кол-во Heap Blocks 8к->4к из-за кластеризации (строки с А лежат более компактно). Время выполнения уменьшилось.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    - Кол-во найденных строк одинаковое
    - Кластеризация уменьшила вдвое чтение кучи
    - Время выполнения упало с 124 до 81 ms