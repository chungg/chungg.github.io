---
layout: post
title:  "web frameworks in the real world"
date:   2024-01-29 00:00:00 -0500
tags: python rust flask axum performance
---

*... or sort of the real world.*

While searching for a Rust web framework for my next project, i came across a
[benchmarking site](https://web-frameworks-benchmark.netlify.app) which tests the
performance of existing web frameworks across many languages. Coming from python, a Rust framework
should improve performance, or put another way, reduce resource requirements by up to 85x!

<figure>
  <img src="{{site.url}}/images/webframe/axum-flask.png"/>
  <figcaption>Axum vs Flask benchmark</figcaption>
</figure>

Since i'm porting a sample Flask app to Rust, here's how 85x translates to the reality.

## the setup

I've created a [sample Flask app](https://github.com/chungg/fullstack-flask) which leverages
htmx as its frontend. It provides most of the functionality needed to build a (numerical) web app.

For the purposes of this benchmark, the python app uses:

- python 3.11.2
- apiflask 2.1
- pyarrow 15.0.0. (why didn't i use polars? historically, i found pyarrow to be faster)
- requests 2.31
- gunicorn 21.2
- [a bunch of flask libs](https://github.com/chungg/fullstack-flask/blob/main/Pipfile)

For Rust, there is a, as of the time of writing, much thinner
[sample app](https://github.com/chungg/fullstack-axum) that provides the same views. It leverages:

- Rust 1.75
- Axum 0.7.4
- polars 0.36.2
- reqwest 0.11.23
- [other libs](https://github.com/chungg/fullstack-axum/blob/main/Cargo.toml)

### caveats

- everything is being run off a mildly overclocked, still unbearably slow, Raspberry Pi 4b with
  8GB memory. absolute numbers will be much slower than expected.
- the only external network calls are to CDN providing JS and CSS. calls to web server are
  isolated to within a network/machine.

## results

### static page

The sample app provides a very simple home page which contains some text and pulls in the
javascript and css dependencies.

![axum-home]({{ "/images/webframe/axum-chrome-index.png" | absolute_url }})
*Axum static page performance*

![flask-home]({{ "/images/webframe/flask-chrome-index.png" | absolute_url }})
*Flask static page performance*

The absolute time it takes for Axum to serve its assets to the browser **was typically ~4-20x
compared to Flask**. This can be seen by comparing the *Waiting for server response* time
values. This **difference is reduced to ~2-5x when factoring in the downloading and processing**
of the assets by the browser. It is further reduced when considering the page as a whole to
include external assets.

**As a whole, page loads were similar between the frameworks, with the range of response times
overlapping between ~700-900ms.**

### slightly more, cpu-"intensive" page

This page is not remotely cpu-bound but it does more than the static page.
It presents three datasets: one taken from a 485KB csv, another from a 600B csv and
lastly, one that is just randomly generated. Arguably, the most
intensive aspect is on the browser to render the data. The data is retrieved through AJAX calls
after page load.

![axum-data]({{ "/images/webframe/axum-chrome-analytics.png" | absolute_url }})
*Axum analytics page performance*

![flask-data]({{ "/images/webframe/flask-chrome-analytics.png" | absolute_url }})
*Flask analytics page performance*

On this page, Axum performed noticeably better, objectively and subjectively. Retrieving static page
assets performed similarly to the land page results. When considering **the content loaded via api
requests, they generally returned ~2-100x faster in Axum**.

For the full page, Axum rendered the page ~1-2x faster. In absolute terms, it would **render
up to 1.2s faster**. On some occassions, Flask could complete as quickly but normally, Axum
performed noticably better. In Axum, the "loading" dialog would rarely be visible where as in
Flask, the loading was always present.

The app also provides this page as a SPA rather than loading the entire page. In this scenario,
the results are similar where Axum typically loaded the page ~300ms quicker.

![axum-spa-data]({{ "/images/webframe/axum-spa-analytics.png" | absolute_url }})
*Axum analytics SPA performance*

![flask-spa-data]({{ "/images/webframe/flask-spa-analytics.png" | absolute_url }})
*Flask analytics SPA performance*

### io-intensive page

The last page the sample app provides is a similar charting page that allows you to input a
stock ticker and it will query Yahoo Finance for the data. For the purposes of benchmarking,
I am treating the query to Yahoo as a query to a extremely distant/slow database.

![axum-io-data]({{ "/images/webframe/axum-yahoo.png" | absolute_url }})
*Axum io-heavy page performance*

![flask-io-data]({{ "/images/webframe/flask-yahoo.png" | absolute_url }})
*Flask io-heavy page performance*

In this case, **the response of both Axum and Flask are comparable**.
In theory, Axum should be able to queue more requests but any performance gains would
be assuming the "database" has the capacity to handle load.

### api

Removing the browser and testing the web frameworks programmatically yields **varied results but
none that return in 85x improvements**.

Testing the *random* api endpoint which effectively just generates a list of random numbers,
**Axum performed ~20% better, averaging 0.8ms faster response time per request**.

<figure>
  <img src="{{site.url}}/images/webframe/api-random.png"/>
  <figcaption>Time to handle 1000 requests - random data endpoint</figcaption>
</figure>

Testing the *vehicle sales* api endpoint which reads and processes a small csv,
**Axum performed ~4x better, averaging 12ms faster response time per request**.

<figure>
  <img src="{{site.url}}/images/webframe/api-sales.png"/>
  <figcaption>Time to handle 1000 requests - sales data endpoint</figcaption>
</figure>

Testing the *financial* api endpoint which calls Yahoo api,
**Axum performed ~5x worse.** I imagine this is due to the usage of synchronous calls.

<figure>
  <img src="{{site.url}}/images/webframe/api-yahoo.png"/>
  <figcaption>Time to handle 100 requests - yahoo data endpoint</figcaption>
</figure>

### system resources

Finally, factoring in the scalability of each framework. Running Flask with 4 workers,
under load required, ~600MB of memory and all the CPUs fully utilised. 

Comparatively, Axum required ~60MB of memory to do the same task and it never fully saturated
the CPUs meaning we had ability to handle significantly more capacity.

*apologies on screenshots, i use transparent terminal background because i'm a monster*

<figure>
  <img src="{{site.url}}/images/webframe/idle-usage.png"/>
  <figcaption>system usage when idle - ~560MB of memory</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/images/webframe/axum-usage.png"/>
  <figcaption>system usage when Axum handling requests to sales api - ~625MB of memory</figcaption>
</figure>

<figure>
  <img src="{{site.url}}/images/webframe/flask-usage.png"/>
  <figcaption>system usage when Flask handling requests to sales api - ~1.1GB of memory</figcaption>
</figure>

## side notes

- the Rust ecosystem is arguably mature enough to be adopted
- there is something weird with polars but it takes forever to compile. on a pi,
  it takes 3+ mins to incrementally *cargo run* and 40+ mins to *cargo build*
- a good chunk of python's memory usage is related to pyarrow library. it is inexplicably large
- *reqwest* in Rust will make you appreciate *requests* in Python. *reqwest* is not fun
- **the timings above were run off the Pi where performance inefficiencies were greatly exaggerated.
  visiting the pages via my mobile devices made the differences imperceptible**.
- even a 1.2 second performance difference is negligble in the modern web. see below:

<figure>
  <img src="{{site.url}}/images/webframe/github.png"/>
  <figcaption>i'm assuming github is mining bitcoin when it's visited</figcaption>
</figure>
