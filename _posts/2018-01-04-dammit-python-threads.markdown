---
layout: post
title:  "python: when to use threads"
date:   2018-01-04 17:00:00 -0500
categories: python concurrency fml
---
let's have a fun talk about that GIL...

ok, let's not because i don't know enough about it and far smarter people have
written vastly about it already. if you do want to want to learn about the GIL
and would like a headache, i've found Larry Hastings' talk on
[removing the GIL](https://www.youtube.com/embed/P3AyI_u66Bw) interesting.
ultimately, all posts regarding threading state that it is only useful if
something is I/O-bound. the [ambiguity](https://www.youtube.com/watch?v=kTHNpusq654)
of the term "I/O-bound" will confound you for the rest of your life.

for my purposes, i've been benchmarking the most recent code of
[Gnocchi](http://gnocchi.xyz) to verify we didn't introduce any performance
regressions since the last release. as i lost my Ceph environment recently,
i've been benchmarking using a redis+file Gnocchi deployment rather than a pure
Ceph deployment as i've done [previously](https://www.slideshare.net/GordonChung/gnocchi-v4-preview)

during my benchmarking, i discovered an issue when adjusting the
`parallel_operations` option in Gnocchi which executes certain I/O related
parts of the code in threads to improve performance. suffice it to say, it did
not improve performance but rather decreased performance by 15%-30%. this was
contrary to the results i had when benchmarking Gnocchi previously, which
gave 25% better performance when threading was enabled.

so i decided to do what i normally don't do: not give up. i wrote really
trivial code to test what was happening. the following is my test code
which crudely simulates a part of Gnocchi's code. it will essentially try
to read and write eight unique time-series to it's own file/key/object:

{% highlight python %}
from concurrent import futures
import itertools
import os
import tempfile
import uuid

import numpy
import redis

rand = uuid.uuid4()
files = ['%s_%s' % (rand, i) for i in range(8)]

client = redis.StrictRedis()
client_remote = redis.StrictRedis(host='192.168.11.1', port='6379')

def fake_work():
    """fake compute work"""
    values = numpy.random.rand(100)
    counts = numpy.array([20,20,20,20,20])
    sums = numpy.bincount(numpy.repeat(numpy.arange(counts.size), counts),
                          weights=values)
    arr = numpy.zeros(len(values), dtype=[('timestamps', '<datetime64[ns]'),
                                          ('values', '<d')])
    arr['values'] = values
    return arr

def executor(fn, list_of_args, threads):
    """uses either threading or itertools to execute list of tasks"""
    if threads == 1:
        return list(itertools.starmap(fn, list_of_args))
    with futures.ThreadPoolExecutor(max_workers=threads) as executor:
        return list(executor.map(lambda args: fn(*args), list_of_args))

def read_work_write_redis(client, key):
    """see method name. does this for redis"""
    hm, item = key.split('_')
    data = client.hget(hm, item)
    result = fake_work()
    client.hset(hm, item, result.tobytes())

def read_work_write_file(path, key):
    """see method name. does this for file io"""
    hm, item = key.split('_')
    dest = os.path.join(path, hm, item)
    try:
        with open(dest, 'rb') as f:
            data = f.read()
    except IOError:
        try:
            os.mkdir(os.path.join(path, hm), 0o750)
        except OSError:
            pass
    result = fake_work()
    tmpfile = tempfile.NamedTemporaryFile(prefix='gnocchi', dir=path,
                                          delete=False)
    tmpfile.write(result.tobytes())
    tmpfile.close()
    os.rename(tmpfile.name, dest)
{% endhighlight %}

running the above code, i got what i saw while running my test script against
Gnocchi: threading sucks and makes things worse.

when reading and writing to the local disk, the performance dips up to 8x
occassionally:
{% highlight python %}
# single-threaded
timeit executor(read_work_write_file, [('/tmp', f) for f in files], 1)
1000 loops, best of 3: 525 µs per loop

# multi-threaded
timeit executor(read_work_write_file, [('/tmp', f) for f in files], 6)
100 loops, best of 3: 4.31 ms per loop
{% endhighlight %}

a similar result happens when interacting with a local redis. I/O in this case
should be less significant as it's now interacting with an in-memory service:
{% highlight python %}
# single-threaded
timeit executor(read_work_write_redis, [(client, f) for f in files], 1)
1000 loops, best of 3: 1.15 ms per loop

# multi-threaded
timeit executor(read_work_write_redis, [(client, f) for f in files], 6)
100 loops, best of 3: 4.87 ms per loop
{% endhighlight %}

where this becomes arguably interesting is when i change the targets to
interact with remote targets. when writing to a machine with a ping of ~400ms:
{% highlight python %}
# single-threaded
timeit executor(read_work_write_file, [('/mnt/tmp', f) for f in files], 1)
10 loops, best of 3: 34.7 ms per loop

# multi-threaded
timeit executor(read_work_write_file, [('/mnt/tmp', f) for f in files], 6)
10 loops, best of 3: 19.5 ms per loop
{% endhighlight %}

in the above case, writing to a remote drive makes the threaded scenario perform
almost 2x better. similarly, when pushing to a redis service on the same remote
machine, threading performs better:
{% highlight python %}
# single-threaded
timeit executor(read_work_write_redis, [(client_remote, f) for f in files], 1)
100 loops, best of 3: 10.8 ms per loop

# multi-threaded
timeit executor(read_work_write_redis, [(client_remote, f) for f in files], 6)
100 loops, best of 3: 6.28 ms per loop
{% endhighlight %}

when pushing to a remote machine that is ~200ms away, it becomes less obvious
whether to use a single thread or multiple threads. pushing to redis,
single thread execution netted better performance in my environment but when
pushing to a remote disk, the inverse held true:
{% highlight python %}
# single-threaded
timeit executor(read_work_write_file, [('/mnt/tmp', f) for f in files], 1)
10 loops, best of 3: 19.5 ms per loop
timeit executor(read_work_write_redis, [(client_remote, f) for f in files], 1)
100 loops, best of 3: 3.71 ms per loop

# multi-threaded
timeit executor(read_work_write_file, [('/mnt/tmp', f) for f in files], 6)
100 loops, best of 3: 12.4 ms per loop
timeit executor(read_work_write_redis, [(client_remote, f) for f in files], 6)
100 loops, best of 3: 4.72 ms per loop
{% endhighlight %}

so when should you use threads? i have no idea. for reference, the fake
workload i am creating in this scenario is minimal so it's definitely
I/O-bound regardless if storage is local or remote:
{% highlight python %}
timeit fake_work()
100000 loops, best of 3: 6.75 µs per loop
{% endhighlight %}

all i can conclude is that concurrency is not parallelism, threading in python
is concurrency, and concurrency in python is a b!tch. that said, you probably
don't need threading if you're doing anything locally... although it could be a
big file... but how big a file... ARGGGHHH! 
