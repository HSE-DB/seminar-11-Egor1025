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
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.078..0.079 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.068..0.068 rows=0 loops=1)
   Index Cond: (category IS NULL)
   Planning Time: 1.340 ms
   Execution Time: 0.171 ms
   ```
   
   *Объясните результат:*
   Индекс применился, но в связке с битмап, так как брин возвращает не точные вхождения, а набор диапазонов. Запрос стоит мало и выполнился быстро.

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
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=19.405..19.406 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.865..0.866 rows=12250 loops=1)
   Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.573 ms
   Execution Time: 19.470 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   В данном плане участвует только индекс по категориям, по авторам обычная фильтрация. Видим, что запрос выполнялся долго, хотя вес маленький. Это связано с тем, что скан нашел много кандидатов (быстро), которые пришлось перепроверять – по сути последовательное чтение значительной части таблицы.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=34.924..34.926 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=34.847..34.848 rows=6 loops=1)
   Group Key: category
   Batches: 1  Memory Usage: 24kB
   ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.056..12.596 rows=150000 loops=1)
   Planning Time: 0.463 ms
   Execution Time: 35.067 ms
   ```
   
   *Объясните результат:*
   Брин индекс не использовался, так как он не подходит для извлечения уникальных значений. Проще прочитать таблицу целиком, найти уникальные категории (всего 6) и отсортировать их. Итого стоимость и время большие.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=18.892..18.892 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=18.888..18.888 rows=0 loops=1)
   Filter: ((author)::text ~~ 'S%'::text)
   Rows Removed by Filter: 150000
   Planning Time: 1.845 ms
   Execution Time: 19.029 ms
   ```
   
   *Объясните результат:*
   Брин не использовали, так как он не подходит для текстов (можно, например, gin). Поэтому просто прошлись по всей таблице и проверили префикс у всех строк (стоимость и время большие).

10. Создайте индекс для регистронезависимого поиска: \
    Добавил `text_pattern_ops`
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books (LOWER(title) text_pattern_ops);
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
   -- без text_pattern_ops
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=30.998..30.999 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=30.992..30.993 rows=1 loops=1)
   Filter: (lower((title)::text) ~~ 'o%'::text)
   Rows Removed by Filter: 149999
   Planning Time: 1.625 ms
   Execution Time: 31.089 ms
   
   -- с text_pattern_ops
   Aggregate  (cost=1151.40..1151.41 rows=1 width=8) (actual time=0.240..0.242 rows=1 loops=1)
   ->  Bitmap Heap Scan on t_books  (cost=20.11..1149.53 rows=750 width=0) (actual time=0.230..0.232 rows=1 loops=1)
   Filter: (lower((title)::text) ~~ 'o%'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on t_books_lower_title_idx  (cost=0.00..19.92 rows=750 width=0) (actual time=0.163..0.163 rows=1 loops=1)
   Index Cond: ((lower((title)::text) ~>=~ 'o'::text) AND (lower((title)::text) ~<~ 'p'::text))
   Planning Time: 3.110 ms
   Execution Time: 0.414 ms
   ```
   
   *Объясните результат:*
   `text_pattern_ops` делает btree индекс пригодным для префиксного LIKE по тексту, и план переключается с полного скана на индексный битмап.

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
   Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=4.834..4.835 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8840
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.282..0.282 rows=730 loops=1)
   Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 3.527 ms
   Execution Time: 5.127 ms
   ```
   
   *Объясните результат:*
   Тут используется составной брин индекс, поэтому оба условия попали в Index и Recheck Cond. Запрос стал в 4 раза быстрее, чем на 7 шаге, при той же цене.