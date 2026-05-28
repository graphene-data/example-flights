---
title: "How 9/11 Reshaped U.S. Commercial Aviation (2000–2005)"
layout: notebook
---

The September 11 attacks didn't just cause a three-day shutdown of U.S. airspace — they accelerated structural changes in the airline industry that had been building for years. This notebook uses the FAA flight record to trace what actually happened: the immediate shock, who recovered, and who didn't.

## The Shock

September 2001's numbers tell the story plainly. The FAA grounded all commercial traffic for three days after the attacks; cancellations spiked to levels unseen in the dataset.

```gsql monthly_volume
from flights
select
  date_trunc('month', dep_time) as month,
  count() as total_flights,
  sum(case when cancelled = 'Y' then 1 else 0 end) as cancellations,
  cancellation_rate
order by month
```

<AreaChart
  title="Monthly Flight Volume (2000–2005)"
  data=monthly_volume
  x=month
  y=total_flights
  height=280px
/>

<BarChart
  title="Monthly Cancellations"
  data=monthly_volume
  x=month
  y=cancellations
  height=200px
/>

The spike in September 2001 — **<Value data=monthly_volume column=cancellations row=20 />** cancellations — was nearly **5× the August 2001 rate**. December 2001 shows zero cancellations in the dataset, likely an artifact of how that month's data was collected after the industry tightened operations; the disruption is really visible in the September through November window.

## Industry-Level Recovery

```gsql yearly_totals
from flights
select
  extract(year from dep_time) as year,
  count() as total_flights,
  sum(case when cancelled = 'Y' then 1 else 0 end) as cancelled_flights,
  count() - sum(case when cancelled = 'Y' then 1 else 0 end) as completed_flights,
  cancellation_rate,
  on_time_arrival_rate,
  round(avg(case when cancelled = 'N' then distance end), 0) as avg_distance_miles
order by year
```

<Row>
  <BigValue data=yearly_totals value=completed_flights title="2000 Completed Flights" row=0 />
  <BigValue data=yearly_totals value=completed_flights title="2005 Completed Flights" row=5 />
  <BigValue data=yearly_totals value=avg_distance_miles title="2000 Avg Route (mi)" row=0 />
  <BigValue data=yearly_totals value=avg_distance_miles title="2005 Avg Route (mi)" row=5 />
</Row>

By raw volume, the industry not only recovered but surpassed pre-9/11 levels — total completed flights rose substantially from 2000 to 2005. But headline volume is misleading. The composition of that growth is where the story lives.

## Winners and Losers: Carriers Indexed to 2000

```gsql carrier_recovery
with base as (
  from flights
  where extract(year from dep_time) = 2000
  select carrier, count() as flights_2000
),
yearly as (
  from flights
  where carrier in ('WN', 'US', 'DL', 'AA', 'NW', 'UA', 'HP', 'AS', 'CO')
  select carrier, carriers.nickname as airline, extract(year from dep_time) as year, count() as flights
)
from yearly inner join base on yearly.carrier = base.carrier
select
  yearly.airline,
  yearly.year,
  yearly.flights,
  round(yearly.flights * 100.0 / base.flights_2000, 1) as index_vs_2000
order by airline, year
```

<LineChart
  title="Flight Volume Indexed to 2000 (2000 = 100)"
  data=carrier_recovery
  x=year
  y=index_vs_2000
  splitBy=airline
  height=340px
/>

The divergence is stark. Southwest ended 2005 at roughly **134%** of its 2000 volume. Delta ended at roughly **60%**. The carriers that shrank most were exactly those most exposed to business travel on long-haul routes — which collapsed after 9/11 and never fully returned in this period.

Continental's story is interesting: it also declined, but from a smaller base, and it launched **Continental Express (RU)** as a regional feeder that was effectively a new entity in the dataset from 2003 onward, masking the legacy carrier's contraction.

## The Low-Cost Carrier Takeover

```gsql market_share
from flights
select
  extract(year from dep_time) as year,
  case
    when carrier in ('WN', 'B6', 'TZ') then 'Low-Cost'
    when carrier in ('AA', 'DL', 'UA', 'US', 'NW', 'CO', 'AS', 'HP') then 'Legacy'
    else 'Regional/Other'
  end as carrier_type,
  count() as flights
order by year, carrier_type
```

<BarChart
  title="Flights by Carrier Type (2000–2005)"
  data=market_share
  x=year
  y=flights
  splitBy=carrier_type
  arrange=stack100
  height=280px
/>

The legacy carrier share of total flights dropped from roughly 50% in 2000 to 33% by 2005. Regional operators (EV, MQ, RU, OH) absorbed much of the remaining growth — themselves feeding traffic into legacy hubs. Low-cost carriers (Southwest, JetBlue) added volume through point-to-point expansion into underserved markets.

## The Route Distance Signal

One structural change barely discussed at the time: average route distance declined steadily, from 826 miles in 2000 to 705 miles by 2004–2005.

```gsql distance_trend
from flights
where cancelled = 'N'
select
  extract(year from dep_time) as year,
  round(avg(distance), 0) as avg_distance,
  round(avg(flight_time), 0) as avg_scheduled_minutes
order by year
```

<LineChart
  title="Average Route Distance and Scheduled Flight Time"
  data=distance_trend
  x=year
  y="avg_distance, avg_scheduled_minutes"
  height=240px
/>

Part of this is compositional: Southwest (avg ~600 miles) grew as a share of operations, pulling the mean down. Part is structural: legacy carriers cut long-haul domestic routes that weren't profitable at post-9/11 load factors. Whatever the cause, the U.S. domestic network was flying shorter by 2005 than it was before the attacks.

## Summary

The industry-level recovery from 9/11 looks clean from 30,000 feet — volume up, cancellations normalized, on-time rates largely stable. But beneath the aggregate:

- **Legacy carriers** (AA, DL, US, NW) collectively shrank their domestic footprints.
- **Southwest** added roughly 4,400 flights annually over five years and kept its model intact through the crisis — no federal loan, no bankruptcy.
- **Regional operators** exploded in size, flying under legacy brand names but operating as separate entities with lower cost structures.
- The **average domestic route** got 15% shorter, reflecting where demand actually held up.

The 2001 data is less a story about a three-day grounding than about a five-year competitive reset.
