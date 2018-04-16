---
layout: post
title:  "orm: verbosity rears its ugly head"
date:   2018-04-16 22:00:00 -0500
tags: python postgresql arrays time-series ORM
---

### preface

This post was originally named `quick and dirty time-series with postgresql
arrays` and aimed to show how leveraging PostgreSQL ARRAYs may improve
the performance of your time-series queries. As i began writing the post, it
soon became apparent the solution did not offer the anticipated gains. Rather,
the benchmarking highlighted the overhead associated with ORM and the potential
pitfalls of blindly building objects. As i believe the gist of the contents
remain valid (and i'm lazy), i've decided to keep the post as is. Please see
benchmarks if that's all you care about.

### start

As a contributor to [Gnocchi](http://gnocchi.xyz), an open-source times-series
database, one of it's strengths i've come to appreciate is it's ability to
scale across different workloads through horizontal scaling or deployment
strategies. With that said, if i were to provide criticism, like many
open-source projects, it is not the easiest to pick up and install.

With Gnocchi, there is no simple `dnf install gnocchi` and `systemctl start
gnocchi`. Instead, you need to be aware of the architectural design of Gnocchi,
the role of APIs, metricd workers, and the underlying storage services. Granted,
this is some pretty petty criticism where in reality, some blame is on the user
for being too lazy to read and learn about the service they're adopting but i
needed an introduction and the preceeding is the result.

To help out those who: don't need the scale Gnocchi provides; don't want to
learn new technology; or want to keep it real simple, another alternative to
storing time-series can be done by leveraging PostgreSQL.

### arrays

To begin, let's first recognise that time-series are arrays; they are
two contiguous arrays of timestamps and values. In Gnocchi, they are stored
as a structured numpy array:

{% highlight python %}
arr = numpy.zeros(len(timestamps), dtype=[('timestamps', '<datetime64[ns]'),
                                          ('values', '<d')])
arr['timestamps'] = timestamps
arr['values'] = values
{% endhighlight %}

They can also be captured as a single array of timestamp-value tuples which
numpy provides if you iterate through the series created above.

Time-series **are not** dictionary or hashmaps. While it is possible to store a
series as `{<time>: <value>, <time>: <value>, ...}` or
`[{'time': <time>, 'value': <value>}, {'time': <time>, 'value': <value>},
...]`, doing so adds unnecessary overhead and voids the sequential
characteristic of time-series by scattering the data all over and requiring
lookups for the simplest of tasks.

### postgresql arrays

[Postgresql](https://www.postgresql.org/) needs no introduction so to get
right into it, one of the neat features it provides beyond the standard SQL
standard is that it can store complex datastructures such as JSON or for this
post, [arrays](https://www.postgresql.org/docs/9.1/static/arrays.html)

How PostgreSQL handles and stores array is beyond my expertise but information
on the topic can be found in the [documentation](
https://www.postgresql.org/docs/current/static/storage-page-layout.html) or in
[forums](
https://dba.stackexchange.com/questions/163659/how-does-postgres-store-array-values)

### time-series in postgresql

As this post aims to provide a quick and easy solution to time-series. The
simplest way to store the data would be as you would most data in SQL: one
datapoint per row.

{% highlight sql %}
CREATE TABLE timeseries (
    id INT,
    time TIMESTAMP,
    value DOUBLE PRECISION);
{% endhighlight %}

At this point, we are not leveraging ARRAYs which is not ideal but this is the
simplest approach and also has the benefit of using standard SQL meaning it
can be ported to other SQL vendors. Additionally, it has the benefit of having
independent datapoints which avoids having to lock, read, update when writing
data.

Alternatively, data can be stored as ARRAYs of times and values:

{% highlight sql %}
CREATE TABLE timeseries (
  id INT,
  times TIMESTAMP[]
  values DOUBLE PRECISION[]);
{% endhighlight %}

or a 2D array of DOUBLE PRECISION values:

{% highlight sql %}
CREATE TABLE timeseries (
  id INT,
  series DOUBLE PRECISION[][]);
{% endhighlight %}

Doing so will optimise your queries and enable better storage efficiency. It
will also add extra complexity as you need to consider your series length. In
cases, where you expect to store data beyond the max column size of PostgreSQL,
you'll need to consider how to hash data appropriately whether by year or by
a more novel hash key. There are [interesting posts](
https://grisha.org/blog/2015/09/23/storing-time-series-in-postgresql-efficiently/)
on this topic but in this case, it actually might be best to use a time-series
database that handles this complexity for you.

Keeping it simple, we'll use PostgreSQL to return ARRAYs rather than storing
them as ARRAYs. This will most likely result in slower queries as it will
build datastructures on demand. To do so, we leverage ARRAY_AGG which is an
aggregate function in PostgreSQL which computes a single result from a set.

{% highlight sql %}
SELECT
    ts.id,
    ARRAY_AGG(ts.time order by ts.time) as times,
    ARRAY_AGG(ts.value order by ts.time) as values
FROM timeseries AS ts
WHERE ts.id IN ('{value}', '{value}', '{value}')
   AND ts.time BETWEEN \'{start_date}\' and \'{end_date}\'
GROUP BY ts.id;
{% endhighlight %}

So that's it. With a simple query, you can return data as a row per series,
rather than many hundreds/thousands of rows of single values.

###  benchmarks

The question that remains is whether this is worth it as it's not difficult to
make python loop through the results and build the series itself. To verify this,
i've timed three scenarios which involve a basic timeseries table that is
joined to another table describing the series using Django:

1. Using standard SQL to retrieve individual rows and build series with Python

{% highlight python %}
import time

from django.db. import connection


start = time.time()
with connection.cursor() as cursor:
    cursor.execute(f"""select
        t.key, t.name, ts.date, ts.close
        from timeseries as ts
        inner join traits as t on ts.id = t.id
        where t.key in {keys}
        and ts.date between \'{start_date}\' and \'{end_date}\';""")
    series = defaultdict(list)
    for i in cursor.fetchall():
        series[(i[0], i[1])].append((i[2], i[3]))
print(f'runtime: {time.time() - start}')
{% endhighlight %}

2. Using ARRAY_AGG to make PostgreSQL build series.

{% highlight python %}
import time

from django.db. import connection


start = time.time()
with connection.cursor() as cursor:
    cursor.execute(f"""select
        t.key, t.name,
        ARRAY_AGG(ts.date order by ts.date) as dates,
        ARRAY_AGG(ts.close order by ts.date) as values
        from timeseries as ts
        inner join traits as t on ts.id = t.id
        where t.key in {keys}
        and ts.date between \'{start_date}\' and \'{end_date}\'
        group by inst.name, inst.key;""");
    instruments = dict()
    for i in cursor.fetchall():
        instruments[(i[0], i[1])] = list(zip(i[3], i[4]))
print(f'runtime: {time.time() - start}')
{% endhighlight %}

3. Using ORM to retreive individual rows and build series with Python

{% highlight python %}
import time


start = time.time()
results = Timeseries.objects.filter(
    traits__key__in=keys,
    date__range=[start_date, end_date]).select_related('traits').only(
        'traits__key', 'traits__name', 'date', 'close')
series = defaultdict(list)
for result in results:
    series[(result.traits.key, result.traits.name)].append(
        (result.date, result.close))
print(f'runtime: {time.time() - start}')
{% endhighlight %}

and the results... when testing against ~200 datapoints/rows:

{% highlight python %}
# response time for standard raw SQL
runtime: 0.0055370330810546875
runtime: 0.005191802978515625
runtime: 0.007096767425537109

# response time for ARRAY_AGG
runtime: 0.0097198486328125
runtime: 0.01343292808532715
runtime: 0.011900901794433594

# response time with ORM.
runtime: 0.02117013931274414
runtime: 0.018471956253051758
{% endhighlight %}

Using arrays in PostgreSQL was not faster than letting python generate series.
ORM was slowest but only by tens of milliseconds so this is insignificant.

When testing against ~40K rows:

{% highlight python %}
# response time for standard raw SQL
runtime: 0.18927311897277832
runtime: 0.1837482452392578
runtime: 0.17506194114685059

# response time for ARRAY_AGG
runtime: 0.272500991821289
runtime: 0.2411351203918457
runtime: 0.22919511795043945

# response time with ORM
runtime: 4.437584161758423
runtime: 3.284132242202759
runtime: 3.108381986618042
{% endhighlight %}

Again, using ARRAY_AGG or not, raw SQL performed relatively similar regardless.
ORM was significantly slower than the other options.

Lastly, when testing against ~800K rows:

{% highlight python %}
# response time for standard raw SQL
runtime: 3.748445987701416
runtime: 4.132349967956543

# response time for ARRAY_AGG
runtime: 3.347858190536499
runtime: 3.4461381435394287

# response time with ORM
runtime: 40.093228816986084
runtime: 40.30661606788635
{% endhighlight %}

In a 'bigger' dataset, ARRAY_AGG performed slightly better but not significantly.
ORM sucked.

### conclusion

This wasn't the conclusion i hoped for, but there is no quick and easy way to
use ARRAYs in PostgreSQL. Using ARRAY_AGG does not really offer much unless
your queries touch millions of row. In this case, it might just be better to
use a real TSDB which does all the formating/computing pre-storage, or to store
the data as an ARRAY in PostgreSQL.

What can be concluded is that my distain for ORM is somewhat justified and if
the model you build is an intermeditary step to your end result, the overhead
of building objects that will be thrown away can have significant impact on
the performance of your system.

I'll need to do some proper profiling of ORM and its effects but that's for
another post.
