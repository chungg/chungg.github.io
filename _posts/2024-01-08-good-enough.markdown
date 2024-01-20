---
layout: post
title:  "sometimes, it's good enough"
date:   2024-01-08 00:00:00 -0500
tags: python rust performance
---

*"\<insert language here\> is slow" - everybody*

Every developer has said this at some point. Sometimes accurately, sometimes to justify their code, but ultimately, does it even matter?

## conditions

To showcase, we will compute the [Klinger Oscillator](https://www.investopedia.com/terms/k/klingeroscillator.asp) indicator.
At a high level, it's a computation that aims to capture trends in money flow but it's not
important what this means or if it's a useful indicator. In reality, we can substitute in any
computation or workload.

The input will be OHLC data from Yahoo which is a JSON dataset with the following structure:

    {
     "open": [377.8299865722656, 380.0, 378.760009765625, ...],
     "high": [377.8299865722656, 380.0, 378.760009765625, ...],
     "low": [377.8299865722656, 380.0, 378.760009765625, ...],
     "close": [377.8299865722656, 380.0, 378.760009765625, ...],
     "volume": [377.8299865722656, 380.0, 378.760009765625, ...]
    }

We'll test datasets corresponding to:
- 1 year of data (~250 datapoints)
- 10 years of data (~2500 datapoints)

The benchmarks below are run using Python 3.11.2 and Rust 1.75.0 on a Raspberry Pi 4b

## vanilla

Computing this in pure Python, the Klinger indicator computation translates to:

```python
def py_vforce(h, l, c, v):
    prevtrend = 99
    prevdm = h[0] - l[0]
    for n in range(1, len(h)):
        trend = 1 if h[n] + l[n] + c[n] > h[n-1] + l[n-1] + c[n-1] else -1
        dm = h[n] - l[n]
        cm = prevcm + dm if trend == prevtrend else prevdm + dm
        yield v[n] * 2 * ((dm / cm) - 1) * trend * 100
        prevtrend = trend
        prevdm = dm
        prevcm = cm


def py_ewma(data, alpha):
    if len(data) == 1:
        return [data[0]]
    else:
        minus1 = py_ewma(data[:-1], alpha)
        return minus1 + [(data[-1] * alpha) + (minus1[-1] * (1 - alpha))]
```

As it aligns pretty neatly with how the computation is described, it's arguably easy to understand
what the above code is doing (ignoring my lazy variable naming).

Running the code, nets the following benchmarks (using timeit in ipython for ease):

    %timeit list(py_vforce(high, low, close, vol))
    # 1 year
    1.02 ms ± 15.7 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    # 10 years
    11.4 ms ± 172 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
    
    
    %timeit py_ewma(list(py_vforce(high, low, close, vol)), 2/35)
    # 1 year
    1.87 ms ± 28.4 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
    # 10 years
    117 ms ± 3.63 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
    
    
    %timeit [a - b for a, b in zip(py_ewma(vf, 2/35), py_ewma(vf, 2/56))]
    # 1 year
    2.76 ms ± 19.7 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
    # 10 years
    215 ms ± 11.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

The runtime of the computation grows linearly and can take 200ms to compute the indicator
over 10 years.

## familiar but different

With no reference whether this is fast or slow, a common technique to improve performance when
dealing with numerical arrays is to use numpy.

Unfortunately, converting a formula into numpy is not quite as simple. Instead of iterately
computing each element and moving to the next, you need to consider how to apply,
sometimes multiple actions, across the entire array in a single pass so that cumulatively when
squashed, they yield the same results. This is particularly difficult when an element depends on
the previous element. I've heard this described as solving things arithmetically vs algebraically
but I also don't know what this means.

### if at first you don't succeed...

One attempt at computing this in numpy yields:

```python
def np_vforce_og(high, low, close, vol):
    trend = high + low + close
    trend = trend[1:] > trend[:-1]
    # making assumption starts with new trend rather than lose another datapoint
    trend_delta = np.concatenate(([False], trend[1:] == trend[:-1]))
    
    dm = high - low
    cm = np.full_like(trend_delta, np.nan, dtype=float)
    
    # if trend != trend-1
    cm[~trend_delta] = dm[:-1][~trend_delta] + dm[1:][~trend_delta]
    
    # if trend == trend-1 (dm part)
    masked = np.ma.array(dm[1:].copy(), mask=~trend_delta, fill_value=np.nan)
    masked[~trend_delta] = -np.diff(
        np.concatenate(([0], np.cumsum(np.nan_to_num(masked))[~trend_delta])))
    
    # cm-1 part
    mask = np.isnan(cm)
    idx = np.where(~mask, np.arange(mask.shape[0]), 0)
    
    # cm-1 + dm
    cm = cm[np.maximum.accumulate(idx)] + np.cumsum(masked)

    # normalise trend to +1/-1
    trend = trend * 2 - 1
    return vol[1:] * (2 * ((dm[1:] / cm) - 1)) * trend * 100
```

If you're unsure what is happening above, welcome to the club (and I wrote it). The comments were
added to help remember what each part is actually doing. With that said, it yields the same results
as the Python solution. It also is slower than the Python solution and was not worth benchmarking


### dust yourself off and try again

Revisiting the numpy solution:

```python
def np_vforce(high, low, close, vol):
    trend = high + low + close
    trend = trend[1:] > trend[:-1]
    # making assumption starts with new trend rather than lose another datapoint
    trend_delta = np.concatenate(([False], trend[1:] == trend[:-1]))
    # trend = np.r_[False, trend[1:] == trend[:-1]]
    # trend = np.insert(trend[1:] == trend[:-1], 0, False)

    dm = high - low
    cm = dm.copy()
    # trend != trend-1
    cm[1:][~trend_delta] += cm[:-1][~trend_delta]

    # trend == trend-1
    # https://stackoverflow.com/questions/18196811/cumsum-reset-at-nan
    cm_cumsum = np.zeros_like(cm)
    cm_cumsum[:-1][trend_delta] = cm[:-1][trend_delta]
    cm_cumsum[:-1][~trend_delta] -= np.diff(
        np.concatenate(([0], np.cumsum(cm_cumsum)[:-1][~trend_delta])))
    cm = np.cumsum(cm_cumsum)[:-1] + cm[1:]

    # normalise trend to +/-1
    trend = trend * 2 - 1
    return vol[1:] * (2 * ((dm[1:] / cm) - 1)) * trend * 100

def ewma(data, window):
    # https://stackoverflow.com/a/42926270
    # https://stackoverflow.com/a/50275583
    alpha = 2 / (window + 1.0)
    alpha_rev = 1 - alpha
    n = data.shape[0]

    pows = alpha_rev**(np.arange(n+1))

    scale_arr = 1 / pows[:-1]
    offset = data[0] * pows[1:]
    pw0 = alpha * alpha_rev**(n-1)

    cumsums = (data * pw0 * scale_arr).cumsum()
    out = offset + cumsums * scale_arr[::-1]
    return out
```

This code results in:

    %timeit np_vforce(high, low, close, vol)
    208 µs ± 7.99 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    586 µs ± 14.5 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    
    
    %timeit ewma(np_vforce(high, low, close, vol), 34)
    274 µs ± 652 ns per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    872 µs ± 27 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    
    
    %timeit ewma(vf, 34) - ewma(vf, 55)
    345 µs ± 1.32 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)
    1.2 ms ± 50.5 µs per loop (mean ± std. dev. of 7 runs, 1,000 loops each)

From these results, we were able to improve performance by 5x to 200x. Unfortunately, this is not
the 10000x numbers we're accustomed to (cue the people who use fibonacci numbers on the regular).

## a whole new world

If we really want to speed things up we'll need to step outside the world of Python. Consider
your past experiences and I'm sure at some point you or a sister team have switched to another
language to improve performance.

For this exercise, we'll choose Rust partly because it's the language where the answer to the
question *"what does it do?"* is *"it's written in Rust"*. :P
For the record, Rust is one of the few other languages I sort of know/use but don't use it
regularly so the below I'm sure can be improved.

In Rust:

```rust
use serde::{Deserialize, Serialize};
use std::fs;
use std::time::Instant;

#[derive(Deserialize, Serialize, Debug)]
struct SecStats {
    high: Vec<f64>,
    low: Vec<f64>,
    close: Vec<f64>,
    volume: Vec<f64>,
}

fn vforce(h: Vec<f64>, l: Vec<f64>, c: Vec<f64>, v: Vec<f64>) -> Vec<f64> {
    let mut prevtrend: i8 = 99;
    let mut prevdm: f64 = h[0] - l[0];
    let mut prevcm: f64 = 0.0;
    let mut result: Vec<f64> = Vec::new();
    for i in 1..h.len() {
        let trend: i8 = {
            if h[i] + l[i] + c[i] > h[i - 1] + l[i - 1] + c[i - 1] {
                1
            } else {
                -1
            }
        };
        let dm: f64 = h[i] - l[i];
        let cm: f64 = {
            if trend == prevtrend {
                prevcm + dm
            } else {
                prevdm + dm
            }
        };
        result.push(v[i] * 2.0 * ((dm / cm) - 1.0) * trend as f64 * 100.0);
        prevtrend = trend;
        prevdm = dm;
        prevcm = cm;
    }
    return result;
}

fn ewma(data: &[f64], alpha: f64) -> Vec<f64> {
    if data.len() == 1 {
        return vec![data[0]];
    } else {
        let mut minus1: Vec<f64> = ewma(&data[..data.len() - 1], alpha);
        minus1.push((data.last().unwrap() * alpha) + (minus1.last().unwrap() * (1.0 - alpha)));
        return minus1;
    }
}

fn ewma(data: &[f64], alpha: f64) -> Vec<f64> {
    // functional solution
    data.iter()
        .scan(None, |state, &x| {
            if *state == None {
                *state = Some(x);
                *state
            } else {
                *state = Some(x * alpha + state.unwrap() * (1.0 - alpha));
                *state
            }
        })
        .collect::<Vec<f64>>()
}

fn main() {
    let data = fs::read_to_string("./klinger.input").expect("Unable to read file");

    let stats: SecStats = serde_json::from_str(&data).expect("JSON does not have correct format.");

    let start = Instant::now();
    let vf = vforce(stats.high, stats.low, stats.close, stats.volume);
    ewma(&vf, 2.0 / 35.0)
        .iter()
        .zip(ewma(&vf, 2.0 / 56.0).iter())
        .map(|(x, y)| x - y).collect::<Vec<f64>>();
    println!("Time elapsed in fn is: {:?}", start.elapsed());
}
```

Similar to the pure Python solution, it reads quite similarly to how it's described in the
Investopedia page but instead of Python syntax, you have Rust.

The runtimes are as follows (i don't know how to do the timeit equivalent in Rust):

    // vforce
    // 1 year
    ~77-120us. 78.758µs as median of 11 runs
    // 1 year (optimised)
    ~8.87-26.759us. 9.741us as median of 11 runs
    // 10 years
    ~796us-1.439ms. 1.04ms as median of 11 runs
    // 10 years (optimised)
    ~95-296us. 133.961us as median of 11 runs
    
    // factoring in ewma
    // 1 year
    ~213-223us. 215.238µs as median of 11 runs
    // 1 year (functional)
    ~114-119us. 116.813µs as median of 11 runs
    // 1 year (functional+optimised)
    ~12.223-19.407us. 13.592us as median of 11 runs
    // 10 years
    ~1.89ms-2.82ms. 2.345ms as median of 11 runs
    // 10 years (functional)
    ~0.997-1.106ms. 1.040651ms as median of 11 runs
    // 10 years (functional+optimised)
    ~127.59-321.181us. 216.386us as median of 11 runs
    
    // klinger 34/55
    // 1 year
    ~261-337us. 265.736µs as median of 11 runs
    // 1 year (functional)
    ~152-156us. 154.405us as median of 11 runs
    // 1 year (functional+optimised)
    ~15.093-38.833us. 18.111us as median of 11 runs
    // 10 years
    ~2.57-4.76ms. 3.719ms as median of 11 runs
    // 10 years (functional)
    ~1.347-1.411ms. 1.378ms as median of 11 runs
    // 10 years (functional+optimised)
    ~147.647-258.349us. 195.442us as median of 11 runs

When dealing with small datasets, we've improved performance by 19-23x versus the numpy solution.
With larger datasets, Rust performs 4-6x better than numpy.

## what have we done

By converting the pure Python solution to other solutions we've reduced the runtime by magnitudes.
We've also increased the cognitive load (apologies for the psychology pseudo-science terminology).
In some cases, it's quite minor. The Rust solution took marginally longer than Python for me
while the numpy solution took significantly longer (even though i have more familiarity with numpy).

The question is whether 1200x performance is worth it. Alternatively, is 200ms worth it?

If this indicator is triggered by a user, they will arguably never notice a 2x or even a 10000x
improvement if computing over 1 year or 10 years. Relative to any other task they will do in a day, 
200ms is negligible. Loading a basic webpage takes multiple times longer.

If this indicator is being run on millions of securities many times a day and used to trigger
alerts, then maybe it's worth the optimisation.

## revisions
- 2024-01-16: add functional ewma solution for rust
- 2024-01-20: benchmark properly on optimised code
