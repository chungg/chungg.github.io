---
layout: post
title:  "lateral join: optimising subqueries (for some cases)"
date:   2021-05-25 16:00:00 -0500
tags: sql postgres join
---

I'm not entirely sure how i found lateral joins. Maybe i was searching for reasons why my query
was slow. Maybe google watches my incognito/private search history and realised i don't know
anything. Regardless, lateral joins are awesome and can help simplify and optimise queries.

From PostgreSQL docs:

> allows (subqueries) to reference columns provided by preceding FROM items. (Without LATERAL,
> each subquery is evaluated independently and so cannot cross-reference any other FROM item.

which, in passing, is not entirely easy to decipher its meaning. I've also seen `LATERAL` joins
referred to as a correlated subquery which gives a bit more insight. 

To make it a bit more tangible, consider you have an `INVENTORY` table that captures how many items
of each product you have each day. There is also a `PRICE` table that captures the price of each
product on a given day because you're a capitalist.

```sql
              Table "public.inventory"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 person  | integer |           | not null |
 date    | date    |           | not null |
 product | integer |           | not null |
 count   | integer |           | not null |
Indexes:
    "idx_date" btree (date)
    "uniq_prod" UNIQUE, btree (person, product, date)


                    Table "public.price"
 Column  |       Type       | Collation | Nullable | Default
---------+------------------+-----------+----------+---------
 product | integer          |           | not null |
 date    | date             |           | not null |
 price   | double precision |           | not null |
Indexes:
    "uniq_prod_price" UNIQUE, btree (product, date)
```

Now if we want to generate a result that shows the value per product on a given day, we would join
the two tables together. The problem is that the price of the product may not change daily so the
price table may not have a price for every date in the inventory table. There are many options to
address this, you could leverage tsrange data types for example but that may increase complexity
in the model. Typically, i've used a window function to figure out a range of dates a price is
valid for similar to the following:

```sql
select i.*, p.price
from inventory i
join (
  select product, date "start",
         coalesce(lag(date) over (partition BY product order by date desc),
                 '2100-01-01')::date "end",
         price
  from price) p on p.product = i.product and p.start <= i.date and i.date < p.end
where i.date = '2021-05-20';
-- '2100-01-01' is some arbitrary time in future so we have end date for latest price
```

Notice the subquery is built with no knowledge of its relationship to the `INVENTORY` table so it
queries across entire `PRICE` table. To avoid this, we could try duplicating conditions to the
subquery but that would make the query less generic.

Luckily, the same query can be done where are subquery has knowledge of other tables and thus
does not need to blindly process an entire table by using `LATERAL` joins

```sql
select i.*, p.price
from inventory i
join lateral (
  select price
  from price p
  where p.product = i.product and p.date <= i.date
  order by date desc
  limit 1) p on true
where i.date = '2021-05-20';
```

Both queries will give the same result but the latter is more readable (once you know what a
`LATERAL` join is). It also has better performance. With an `INVENTORY` table of ~45M rows
(~100 people, each with ~25 products over ~50 years) and a `PRICE` table of ~4.5M rows (over ~50
years), querying the value of each product an individual holds on a given day generates the
following plans:

```sql
-- normal join

                                                                QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------
 Merge Join  (cost=731848.06..1908773.98 rows=6576966 width=24) (actual time=2138.093..7967.517 rows=2500 loops=1)
   Merge Cond: (i.product = price.product)
   Join Filter: ((price.date <= i.date) AND (i.date < (COALESCE(lag(price.date) OVER (?), '2100-01-01'::date))))
   Rows Removed by Join Filter: 45655000
   ->  Sort  (cost=276.90..284.94 rows=3214 width=16) (actual time=0.677..1.140 rows=2500 loops=1)
         Sort Key: i.product
         Sort Method: quicksort  Memory: 214kB
         ->  Index Scan using idx_date on inventory i  (cost=0.44..89.69 rows=3214 width=16) (actual time=0.022..0.369 rows=2500 loops=1)
               Index Cond: (date = '2021-05-20'::date)
   ->  Materialize  (cost=731422.71..879809.58 rows=4565750 width=20) (actual time=1940.290..4673.601 rows=45657501 loops=1)
         ->  WindowAgg  (cost=731422.71..822737.71 rows=4565750 width=20) (actual time=1940.282..2154.701 rows=456576 loops=1)
               ->  Sort  (cost=731422.71..742837.08 rows=4565750 width=16) (actual time=1940.249..1990.298 rows=456577 loops=1)
                     Sort Key: price.product, price.date DESC
                     Sort Method: external merge  Disk: 116224kB
                     ->  Seq Scan on price  (cost=0.00..70337.50 rows=4565750 width=16) (actual time=0.016..352.202 rows=4565750 loops=1)
 Planning Time: 0.670 ms
 JIT:
   Functions: 18
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.427 ms, Inlining 44.187 ms, Optimization 84.182 ms, Emission 60.225 ms, Total 190.021 ms
 Execution Time: 8012.483 ms

-- lateral join
                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=0.87..6662.25 rows=3214 width=24) (actual time=0.029..14.977 rows=2500 loops=1)
   ->  Index Scan using idx_date on inventory i  (cost=0.44..89.69 rows=3214 width=16) (actual time=0.010..0.397 rows=2500 loops=1)
         Index Cond: (date = '2021-05-20'::date)
   ->  Limit  (cost=0.43..2.02 rows=1 width=12) (actual time=0.005..0.006 rows=1 loops=2500)
         ->  Index Scan Backward using uniq_prod_price on price p  (cost=0.43..9695.44 rows=6088 width=12) (actual time=0.005..0.005 rows=1 loops=2500)
               Index Cond: ((product = i.product) AND (date <= i.date))
 Planning Time: 0.179 ms
 Execution Time: 15.130 ms
```

The `LATERAL` join query returns in a fraction of the time. Similarly, on a larger query that
grabs the value of each product an individual holds at the end of each year, `LATERAL` join still
provides marked performance gains:

```sql
-- regular join
explain analyze
select i.*, p.price
from inventory i
join (
  select product, date "start",
         coalesce(lag(date) over (partition BY product order by date desc),
                  '2100-01-01')::date "end",
         price
  from price) p on p.product = i.product and p.start <= i.date and i.date < p.end
where extract(month from "date") = 12 and extract(day from "date") = 31;
                                                                                                    QUERY PLAN                                             
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Merge Join  (cost=1455046.60..1868480.99 rows=2334884 width=24) (actual time=5016.843..296495.753 rows=125000 loops=1)
   Merge Cond: (i.product = price.product)
   Join Filter: ((price.date <= i.date) AND (i.date < (COALESCE(lag(price.date) OVER (?), '2100-01-01'::date))))
   Rows Removed by Join Filter: 2282750000
   ->  Sort  (cost=723569.00..723571.85 rows=1141 width=16) (actual time=3081.470..3129.406 rows=125000 loops=1)
         Sort Key: i.product
         Sort Method: external merge  Disk: 3192kB
         ->  Gather  (cost=1000.00..723511.06 rows=1141 width=16) (actual time=211.294..3040.475 rows=125000 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               ->  Parallel Seq Scan on inventory i  (cost=0.00..722396.96 rows=475 width=16) (actual time=223.979..2935.220 rows=41667 loops=3)
                     Filter: ((date_part('month'::text, (date)::timestamp without time zone) = '12'::double precision) AND (date_part('day'::text, (date)::timestamp without time zone) = '31'::double precision))
                     Rows Removed by Filter: 15177500
   ->  Materialize  (cost=731422.71..879809.58 rows=4565750 width=20) (actual time=1927.052..137719.308 rows=2282875001 loops=1)
         ->  WindowAgg  (cost=731422.71..822737.71 rows=4565750 width=20) (actual time=1927.047..2143.752 rows=456576 loops=1)
               ->  Sort  (cost=731422.71..742837.08 rows=4565750 width=16) (actual time=1927.020..1977.479 rows=456577 loops=1)
                     Sort Key: price.product, price.date DESC
                     Sort Method: external merge  Disk: 116224kB
                     ->  Seq Scan on price  (cost=0.00..70337.50 rows=4565750 width=16) (actual time=0.011..332.127 rows=4565750 loops=1)
 Planning Time: 0.231 ms
 JIT:
   Functions: 22
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 2.537 ms, Inlining 215.568 ms, Optimization 106.636 ms, Emission 86.688 ms, Total 411.429 ms
 Execution Time: 296524.393 ms


-- lateral join
explain analyze
select i.*, p.price
from inventory i
join lateral (
  select price
  from price p
  where p.product = i.product and p.date <= i.date
  order by date desc
  limit 1) p on true
where extract(month from "date") = 12 and extract(day from "date") = 31;
                                                                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=1000.43..725844.38 rows=1141 width=24) (actual time=131.019..3501.278 rows=125000 loops=1)
   ->  Gather  (cost=1000.00..723511.06 rows=1141 width=16) (actual time=130.952..2388.902 rows=125000 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Parallel Seq Scan on inventory i  (cost=0.00..722396.96 rows=475 width=16) (actual time=128.022..2826.574 rows=41667 loops=3)
               Filter: ((date_part('month'::text, (date)::timestamp without time zone) = '12'::double precision) AND (date_part('day'::text, (date)::timestamp without time zone) = '31'::double precision))
               Rows Removed by Filter: 15177500
   ->  Limit  (cost=0.43..2.02 rows=1 width=12) (actual time=0.008..0.008 rows=1 loops=125000)
         ->  Index Scan Backward using uniq_prod_price on price p  (cost=0.43..9695.44 rows=6088 width=12) (actual time=0.008..0.008 rows=1 loops=125000)
               Index Cond: ((product = i.product) AND (date <= i.date))
 Planning Time: 0.223 ms
 JIT:
   Functions: 14
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 1.765 ms, Inlining 160.967 ms, Optimization 58.548 ms, Emission 40.490 ms, Total 261.770 ms
 Execution Time: 3509.229 ms
```

Switching to `LATERAL` joins resulted in queries that took less than 1% of the original query with
the added benefit of a more readable query.
