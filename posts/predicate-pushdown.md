# Gap in PostgreSQL predicate pushdown

Recently at work, there was a slow query that caused daily CPU spikes.
It minimally reproduces with the latest version of postgres docker image
with the following sequence of statements:

```
CREATE TABLE test (a int PRIMARY KEY, b int);
INSERT INTO test SELECT t, t FROM generate_series(1, 1000000) t;

CREATE TABLE test2(a int PRIMARY KEY, c int);
INSERT INTO test2 SELECT t, t FROM generate_series(1, 1000000) t;

ANALYZE;

WITH cte AS (
	SELECT a, sum(a) AS s FROM test GROUP BY a
)
SELECT *
  FROM cte
  LEFT JOIN test2 ON cte.a = test2.a
 ORDER BY cte.s DESC
 LIMIT 3;
```
Conceptually, we can see that the we only need the first three rows from the
CTE where those rows can be determined very fast via the index on `test.a`.

However, running the above with `EXPLAIN (ANALYZE TRUE, TIMING TRUE)` yields:
```
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Limit  (cost=99313.66..99313.67 rows=3 width=20) (actual time=8768.987..8768.995 rows=3 loops=1)                                                               |
|   ->  Sort  (cost=99313.66..101813.66 rows=1000000 width=20) (actual time=8768.980..8768.981 rows=3 loops=1)                                                   |
|         Sort Key: (sum(test.a)) DESC                                                                                                                           |
|         Sort Method: top-N heapsort  Memory: 25kB                                                                                                              |
|         ->  Merge Left Join  (cost=0.85..86388.85 rows=1000000 width=20) (actual time=0.686..8393.873 rows=1000000 loops=1)                                    |
|               Merge Cond: (test.a = test2.a)                                                                                                                   |
|               ->  GroupAggregate  (cost=0.42..40980.43 rows=1000000 width=12) (actual time=0.331..2822.682 rows=1000000 loops=1)                               |
|                     Group Key: test.a                                                                                                                          |
|                     ->  Index Only Scan using test_pkey on test  (cost=0.42..25980.42 rows=1000000 width=4) (actual time=0.248..2119.261 rows=1000000 loops=1) |
|                           Heap Fetches: 0                                                                                                                      |
|               ->  Index Scan using test2_pkey on test2  (cost=0.42..30408.42 rows=1000000 width=8) (actual time=0.144..4889.683 rows=1000000 loops=1)          |
| Planning Time: 1.466 ms                                                                                                                                        |
| Execution Time: 8769.803 ms                                                                                                                                    |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
Fetching a few rows followed by a left join on indexed rows should definitely NOT take more than a second,
but the query planner fails to capture that.

It runs fast (as it should) if we make a small modification:
```
EXPLAIN (ANALYZE TRUE, TIMING TRUE)
 WITH cte AS (
     SELECT a, sum(a) AS s FROM test GROUP BY a LIMIT 3
 )
 SELECT *
   FROM cte
   LEFT JOIN test2 ON cte.a = test2.a
 ORDER BY cte.s DESC
```
which yields
```
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sort  (cost=25.90..25.91 rows=3 width=20) (actual time=14.714..14.726 rows=3 loops=1)                                                                   |
|   Sort Key: (sum(test.a)) DESC                                                                                                                          |
|   Sort Method: quicksort  Memory: 25kB                                                                                                                  |
|   ->  Nested Loop Left Join  (cost=0.85..25.88 rows=3 width=20) (actual time=14.271..14.285 rows=3 loops=1)                                             |
|         ->  Limit  (cost=0.42..0.55 rows=3 width=12) (actual time=13.888..13.893 rows=3 loops=1)                                                        |
|               ->  GroupAggregate  (cost=0.42..40980.43 rows=1000000 width=12) (actual time=13.886..13.891 rows=3 loops=1)                               |
|                     Group Key: test.a                                                                                                                   |
|                     ->  Index Only Scan using test_pkey on test  (cost=0.42..25980.42 rows=1000000 width=4) (actual time=13.865..13.868 rows=4 loops=1) |
|                           Heap Fetches: 0                                                                                                               |
|         ->  Index Scan using test2_pkey on test2  (cost=0.42..8.44 rows=1 width=8) (actual time=0.123..0.123 rows=1 loops=3)                            |
|               Index Cond: (a = test.a)                                                                                                                  |
| Planning Time: 2.811 ms                                                                                                                                 |
| Execution Time: 16.196 ms                                                                                                                               |
+---------------------------------------------------------------------------------------------------------------------------------------------------------+
```
This is what we ended up doing for the aforementioned slow query, except it was a lot more complicated
since there were many more joins in the query (we squeezed `LIMIT` clause for each of these tables).

Maybe I'll dig into the query planner in Postgres sometime to see why it fails to plan efficiently for the first case.
