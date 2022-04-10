# PostgreSQL RANK() function

The RANK() function assigns a rank to every row within a partition of a result set.

## The following illustrates the syntax of the RANK() function:

```
RANK() OVER (
  [PARTITION BY partition_expression, ... ]
  ORDER BY sort_expression [ASC | DESC], ...
)
```

In this syntax:

  First, the `PARTITION BY` clause distributes rows of the result set into partitions to which the `RANK()` function is applied.
  Then, the `ORDER BY` clause specifies the order of rows in each a partition to which the function is applied.

The `RANK()` function can be useful for creating top-N and bottom-N reports.

## PostgreSQL RANK() function examples

  - uses the `OVER`
  - uses the func `RANK()`

  ```
  (local@localhost:5432) [local] > SELECT id, name, RANK() OVER (ORDER BY score DESC) score_rank, score FROM companies LIMIT 7;
       name      | score_rank | score
  ---------------+------------+-------
  Copa Airlines  |          1 |   398
  Basecamp       |          2 |   385
  1000Bulbs.com  |          3 |   328
  3Play Media    |          4 |   324
  6 Pack Fitness |          5 |   244
  1000 Markets   |          6 |   232
  3jam           |          7 |   190
  (7 rows)

  Time: 6.887 ms
  ```
  - `PARTITION` within a partition of a result set

  ```
  (local@localhost:5432) [local] > select id, name, category_id, RANK() OVER (PARTITION BY category_id ORDER BY score DESC) score_rank, score from companies LIMIT 7;
       name      | category_id | score_rank | score
  ---------------+-------------+------------+-------
  Basecamp       |           1 |          1 |   385
  1000Bulbs.com  |           1 |          2 |   328
  3Play Media    |           2 |          1 |   324
  Copa Airlines  |           5 |          1 |   398
  6 Pack Fitness |             |          1 |   244
  1000 Markets   |             |          2 |   232
  3jam           |             |          3 |   190
  (7 rows)

  Time: 19.045 ms
  ```