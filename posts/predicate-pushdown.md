# Gap in PostgreSQL predicate pushdown

Recently at work, there was a slow query that caused daily CPU spikes.
It minimally reproduces with the latest version of postgres docker image:

```
CREATE TABLE test (a int PRIMARY KEY, b int);

INSERT INTO test SELECT t, t FROM generate_series(1, 1000000) t;
CREATE TABLE test2(a int PRIMARY KEY, c int);
INSERT INTO test2 SELECT t, t FROM generate_series(1, 1000000) t;
CREATE TABLE test3(a int PRIMARY KEY, d int);
INSERT INTO test3 SELECT t, t FROM generate_series(1, 1000000) t;

ANALYZE;

WITH cte AS (
	SELECT a, sum(a) AS s FROM test GROUP BY a
)
SELECT *
  FROM cte
  LEFT JOIN test2 ON cte.a = test2.a
  LEFT JOIN test3 ON cte.a = test3.a
 ORDER BY cte.s DESC
 LIMIT 3;
```
Conceptually, we can see that the we only need the first three rows from the
CTE where those rows can be determined very fast via the index on `test.a`.

However, running the above with `EXPLAIN (ANALYZE TRUE, TIMING TRUE)` yields:
```
                                                                              QUERY PLAN                                                                              
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=149150.09..149150.10 rows=3 width=28) (actual time=6945.652..6945.660 rows=3 loops=1)
   ->  Sort  (cost=149150.09..151650.09 rows=1000000 width=28) (actual time=6726.300..6726.301 rows=3 loops=1)
         Sort Key: (sum(test.a)) DESC
         Sort Method: top-N heapsort  Memory: 25kB
         ->  Merge Left Join  (cost=1.27..136225.27 rows=1000000 width=28) (actual time=1.112..6196.168 rows=1000000 loops=1)
               Merge Cond: (test.a = test3.a)
               ->  Merge Left Join  (cost=0.85..90816.85 rows=1000000 width=20) (actual time=1.008..4484.440 rows=1000000 loops=1)
                     Merge Cond: (test.a = test2.a)
                     ->  GroupAggregate  (cost=0.42..45408.43 rows=1000000 width=12) (actual time=0.909..2270.211 rows=1000000 loops=1)
                           Group Key: test.a
                           ->  Index Only Scan using test_pkey on test  (cost=0.42..30408.42 rows=1000000 width=4) (actual time=0.339..1410.292 rows=1000000 loops=1)
                                 Heap Fetches: 0
                     ->  Index Scan using test2_pkey on test2  (cost=0.42..30408.42 rows=1000000 width=8) (actual time=0.083..1298.690 rows=1000000 loops=1)
               ->  Index Scan using test3_pkey on test3  (cost=0.42..30408.42 rows=1000000 width=8) (actual time=0.090..685.574 rows=1000000 loops=1)
 Planning Time: 5.843 ms
 JIT:
   Functions: 14
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 12.283 ms (Deform 0.350 ms), Inlining 0.000 ms, Optimization 46.561 ms, Emission 172.965 ms, Total 231.810 ms
 Execution Time: 9052.856 ms
(20 rows)
```
Fetching a few rows followed by two left joins on indexed rows should definitely NOT take more than a second,
but the query planner fails to capture that.

It runs fast if we make a small modification:
```
postgres@localhost:postgres> EXPLAIN (ANALYZE TRUE, TIMING TRUE)
   WITH cte AS (
         SELECT a, sum(a) AS s FROM test GROUP BY a LIMIT 3
        )
 SELECT *
   FROM cte
   LEFT JOIN test2 ON cte.a = test2.a
   LEFT JOIN test3 ON cte.a = test3.a
  ORDER BY cte.s DESC

+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sort  (cost=51.23..51.23 rows=3 width=28) (actual time=7.545..7.562 rows=3 loops=1)                                                                         |
|   Sort Key: (sum(test.a)) DESC                                                                                                                              |
|   Sort Method: quicksort  Memory: 25kB                                                                                                                      |
|   ->  Nested Loop Left Join  (cost=1.27..51.20 rows=3 width=28) (actual time=6.970..7.038 rows=3 loops=1)                                                   |
|         ->  Nested Loop Left Join  (cost=0.85..25.88 rows=3 width=20) (actual time=5.356..5.389 rows=3 loops=1)                                             |
|               ->  Limit  (cost=0.42..0.55 rows=3 width=12) (actual time=2.736..2.748 rows=3 loops=1)                                                        |
|                     ->  GroupAggregate  (cost=0.42..40980.43 rows=1000000 width=12) (actual time=2.734..2.745 rows=3 loops=1)                               |
|                           Group Key: test.a                                                                                                                 |
|                           ->  Index Only Scan using test_pkey on test  (cost=0.42..25980.42 rows=1000000 width=4) (actual time=2.525..2.531 rows=4 loops=1) |
|                                 Heap Fetches: 0                                                                                                             |
|               ->  Index Scan using test2_pkey on test2  (cost=0.42..8.44 rows=1 width=8) (actual time=0.859..0.859 rows=1 loops=3)                          |
|                     Index Cond: (a = test.a)                                                                                                                |
|         ->  Index Scan using test3_pkey on test3  (cost=0.42..8.44 rows=1 width=8) (actual time=0.495..0.495 rows=1 loops=3)                                |
|               Index Cond: (a = test.a)                                                                                                                      |
| Planning Time: 5.373 ms                                                                                                                                     |
| Execution Time: 9.032 ms                                                                                                                                    |
+-------------------------------------------------------------------------------------------------------------------------------------------------------------+
```
This is what we ended up doing for the aforementioned slow query, except it was a lot more complicated
since there were many more joins in the query (we squeezed `LIMIT` clause for each of these tables).
