---
layout: post
title:  "time"
date:   2024-03-26 00:00:00 -0500
tags: rust datetime performance
---

*this isn't an introspective post nor about how great the Inception song is...*

Here is a quick test to see how well the, as of writing, main datetime
libraries in Rust handle unix timestamps.

I had always thought that datetimes were stored as integers relative to epoch,
a few additional attributes to capture timezone and resolution, and a bunch of
methods to extract info as needed. It turns out, both libraries in Rust store
date and time separately with date being captured as year + day of year +
other stuff and time captured as secs and fractional seconds
(apparently leap seconds exists ¯\\_(ツ)_/¯). I have no idea why ordinal dates
are used but i imagine there's a reason (personally, never had a use case for it
as a user).

The following just converts a given timestamp to a calendar date and back.

# chrono

```rust
fn chrono_bench(ts: u64) {
    DateTime::from_timestamp(ts as i64, 0).unwrap()
        .date_naive().and_hms_opt(0,0,0).unwrap().timestamp();
}

// chrono                  time:   [44.748 ns 44.774 ns 44.811 ns]
//                         change: [-3.1965% -2.1425% -1.3246%] (p = 0.00 < 0.05)
//                         Performance has improved.
// Found 6 outliers among 100 measurements (6.00%)
//   1 (1.00%) low mild
//   2 (2.00%) high mild
//   3 (3.00%) high severe
```

<figure>
  <img src="{{site.url}}/images/time/dt-chrono.png"/>
  <figcaption>statistics for chrono lib</figcaption>
</figure>

# time-rs

```rust
fn time_bench(ts: u64) {
    OffsetDateTime::from_unix_timestamp(ts as i64).unwrap()
        .date().midnight().assume_utc().unix_timestamp();
}

// time-rs                 time:   [51.653 ns 51.672 ns 51.693 ns]
//                         change: [-14.726% -10.086% -6.0346%] (p = 0.00 < 0.05)
//                         Performance has improved.
// Found 9 outliers among 100 measurements (9.00%)
//   5 (5.00%) low mild
//   1 (1.00%) high mild
//   3 (3.00%) high severe
```

<figure>
  <img src="{{site.url}}/images/time/dt-time-rs.png"/>
  <figcaption>statistics for time-rs lib</figcaption>
</figure>

# eras (howard's version)

I found [this algorithm](https://howardhinnant.github.io/date_algorithms.html)
years back and have decided to implement it. One thing i found neat was that it
treats March 1st as the start of the year to simplify leap years.

```rust
pub fn iso_to_days(mut y: i32, m: i32, d: i32) -> i32 {
    // days since epoch 1970-01-01
    y -= (m <= 2) as i32; // logic treats March 1 as start of year to make leap years easier
    let era = if y >= 0 { y } else { y - 399 } / 400;
    let yoe = (y - era * 400).abs();
    let doy = (153 * (if m > 2 { m - 3 } else { m + 9 }) + 2) / 5 + d - 1;
    let doe = yoe * 365 + yoe / 4 - yoe / 100 + doy;
    era * 146097 + doe - 719468
}

pub fn days_to_iso(mut days: i32) -> (i32, i32, i32) {
    days += 719468;
    let era = if days >= 0 { days } else { days - 146096 } / 146097;
    let doe = (days - era * 146097).abs();
    let yoe = (doe - doe / 1460 + doe / 36524 - doe / 146096) / 365;
    let doy = doe - (365 * yoe + yoe / 4 + yoe / 100);
    let mp = (5 * doy + 2) / 153;
    let d = doy - (153 * mp + 2) / 5 + 1;
    let m = if mp < 10 { mp + 3 } else { mp - 9 };
    let y = yoe.abs() + era * 400 + (m <= 2) as i32;
    (y, m, d)
}

fn eras_bench(ts: u64) {
    let (y, m, d) = days_to_iso((ts / 86400) as i32);
    iso_to_days(y, m, d) * 86400;
}

// eras                    time:   [43.761 ns 43.790 ns 43.821 ns]
//                         change: [-5.4260% -4.5470% -3.7963%] (p = 0.00 < 0.05)
//                         Performance has improved.
// Found 5 outliers among 100 measurements (5.00%)
//   1 (1.00%) low mild
//   1 (1.00%) high mild
//   3 (3.00%) high severe
```

<figure>
  <img src="{{site.url}}/images/time/dt-eras.png"/>
  <figcaption>statistics for eras date algorithms</figcaption>
</figure>

# python

Just because...

```python
datetime.datetime.fromordinal(
    datetime.datetime.fromtimestamp(ts).date().toordinal()
).timestamp()
# 2.58 µs ± 186 ns per loop (mean ± std. dev. of 7 runs, 100,000 loops each)
```

# conclusion

use chrono
