---
layout: notebook
title: Does seasonality drive delays?
---

# Do flight delays have seasonality?

Winter storms and summer thunderstorms are the two most common excuses for a late flight. This notebook checks whether that folk wisdom holds up in the FAA data: roughly 345,000 U.S. commercial flights from 2000 to 2005, broken out by month.

## Delays follow a clear seasonal curve

```gsql monthly_delay
from flights
where cancelled = 'N'
select
  extract(month from dep_time) as month,
  avg(dep_delay) as avg_delay,
  avg(case when dep_delay > 15 then 1.0 else 0 end) as delay_rate
order by month
```

<AreaChart
  data=monthly_delay
  x=month
  y=avg_delay
  title="Avg departure delay by month (all years pooled)"
/>

There are two distinct peaks — **June through August**, and **December through March** — separated by two calm troughs in April and September. The pattern is not subtle: average delay in July (10.5 min) is nearly double that in September (4.8 min), and the share of flights delayed more than 15 minutes swings from 12% to 19% across the year.

The seasons map onto known operational stress points. Summer is thunderstorm season across the central and eastern U.S., which backs up en-route traffic and hub airports all afternoon. Winter is snow and ice season in the Northeast and Midwest, which slows de-icing, taxiing, and runway throughput. Spring and early fall sit in between — the "shoulder seasons" with the mildest weather nationwide.

## The pattern repeats every year, not just once

A single hump could be a one-year fluke — an especially bad summer, say. Splitting the same query out by year shows whether the shape is a stable feature of the calendar or a coincidence.

```gsql monthly_delay_by_year
from flights
where cancelled = 'N'
select
  extract(year from dep_time)::varchar as year,
  extract(month from dep_time) as month,
  avg(dep_delay) as avg_delay
order by year, month
```

<LineChart
  data=monthly_delay_by_year
  x=month
  y=avg_delay
  splitBy=year
  title="Avg departure delay by month, one line per year (2000–2005)"
/>

Every single year traces the same twin-peak silhouette — summer bump, autumn dip, winter bump, spring dip — just shifted up or down depending on how bad that particular year was overall. That consistency is the signature of a real seasonal effect rather than noise: whatever drives the monthly swing is baked into the calendar, not into any one year's events.

## How big is the seasonal effect, really?

To put this in context, here's month's η² (share of variance in departure delay explained) next to the other factors examined in [What makes your flight late?](/delay_factors):

```gsql factor_importance
with
info as (
  from flights where cancelled = 'N'
  select avg(dep_delay) as grand_mean, var_pop(dep_delay) * count() as ss_total
),
month_groups as (
  from flights where cancelled = 'N'
  select extract(month from dep_time) as grp, count() as n, avg(dep_delay) as grp_mean
),
hour_groups as (
  from flights where cancelled = 'N' and extract(hour from dep_time)::integer between 5 and 23
  select extract(hour from dep_time) as grp, count() as n, avg(dep_delay) as grp_mean
),
carrier_groups as (
  from flights where cancelled = 'N'
  select carriers.nickname as grp, count() as n, avg(dep_delay) as grp_mean
),
dow_groups as (
  from flights where cancelled = 'N'
  select extract(dow from dep_time)::varchar as grp, count() as n, avg(dep_delay) as grp_mean
)
select 'Hour of day' as factor, sum(n * power(grp_mean - grand_mean, 2)) / max(ss_total) as eta_squared
  from hour_groups inner join info on true
union all
select 'Month', sum(n * power(grp_mean - grand_mean, 2)) / max(ss_total)
  from month_groups inner join info on true
union all
select 'Airline', sum(n * power(grp_mean - grand_mean, 2)) / max(ss_total)
  from carrier_groups inner join info on true
union all
select 'Day of week', sum(n * power(grp_mean - grand_mean, 2)) / max(ss_total)
  from dow_groups inner join info on true
order by eta_squared desc
```

<BarChart
  data=factor_importance
  x=eta_squared
  y=factor
  title="Variance in departure delay explained by each factor (η²)"
  height=220px
/>

Month explains about **0.4%** of the variance in any individual flight's delay — real, but modest, and well behind hour of day. Seasonality shifts the *average* delay across the whole system by several minutes, which matters for planning capacity and staffing months in advance. It says much less about whether any single flight will be late, because individual flights vary enormously for reasons (a late inbound aircraft, one bad afternoon at one airport) that swamp the broad seasonal drift.

## Cancellations spike in winter — and in one anomalous month

```gsql monthly_cancel
from flights
select
  extract(month from dep_time) as month,
  avg(case when cancelled = 'Y' then 1.0 else 0 end) as cancel_rate
order by month
```

<BarChart
  data=monthly_cancel
  x=month
  y=cancel_rate
  title="Cancellation rate by month (all years pooled)"
/>

Cancellations follow a related but distinct pattern: low in the shoulder months, elevated in summer (storms) and rising toward December (snow). September stands out as the single highest month in this chart, which looks at first like it contradicts the delay-rate story above (September had the *lowest* average delay).

```gsql september_by_year
from flights
where extract(month from dep_time) = 9
select
  extract(year from dep_time) as year,
  count() as flights,
  avg(case when cancelled = 'Y' then 1.0 else 0 end) as cancel_rate,
  round(avg(case when cancelled = 'Y' then 1.0 else 0 end) * 100, 1)::varchar || '%' as cancel_rate_label
order by year
```

<BarChart
  data=september_by_year
  x=year
  y=cancel_rate
  title="September cancellation rate by year"
/>

That's not a seasonal signal at all — it's <Value data=september_by_year column=cancel_rate_label row=1 />, the cancellation rate for September **2001**, an order of magnitude above every other September in the dataset. Every other year's September cancellation rate sits comfortably below 2%, in line with the mild shoulder-season pattern seen in delays. This is a good reminder to check whether a seasonal aggregate is being driven by the calendar or by a single, non-recurring event before drawing conclusions from it.

## Takeaways

- Departure delays have genuine seasonality: worse in summer (thunderstorms) and winter (snow/ice), better in spring and fall — and the shape repeats reliably across all six years in the data.
- The effect is real but secondary. Month explains roughly a tenth as much delay variance as hour of day does; seasonality is a planning-horizon signal, not a strong predictor for any one flight.
- Cancellation seasonality looks similar at first glance, but the September figure is an artifact of the September 11, 2001 groundings, not a recurring pattern — a reminder to check aggregates for single-event contamination before trusting them.
