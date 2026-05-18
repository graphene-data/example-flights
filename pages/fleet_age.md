---
title: Does aircraft age actually predict delays?
layout: notebook
---

Every nervous flyer has scanned the departure board and quietly wondered whether the 22-year-old MD-83 assigned to their gate is going to hold together. The assumption feels intuitive — older machines break down more, older planes must mean more delays. This analysis tests that assumption against five years of FAA data.

## The fleet in numbers

```gsql fleet_summary
from flights
where cancelled = 'N' and aircraft_age is not null and aircraft_age between 0 and 35
select
  count() as flights_with_age_data,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  min(aircraft_age) as min_age,
  max(aircraft_age) as max_age
```

<Row>
  <BigValue data=fleet_summary value=flights_with_age_data title="Flights with age data" fmt=num0 />
  <BigValue data=fleet_summary value=avg_fleet_age title="Avg fleet age (years)" />
  <BigValue data=fleet_summary value=min_age title="Youngest aircraft (years)" />
  <BigValue data=fleet_summary value=max_age title="Oldest aircraft (years)" />
</Row>

About 340,000 flights in the dataset can be matched to an aircraft with a known build year. That's a solid base for the analysis — enough that sampling noise won't obscure a real signal if one exists.

## Age vs. departure delay: the relationship you expected isn't there

```gsql age_vs_delay
from flights
where aircraft_age is not null and aircraft_age between 0 and 30
select
  floor(aircraft_age / 5) * 5 as age_bucket,
  count() as total_flights,
  avg(case when cancelled = 'N' then dep_delay end) as avg_dep_delay,
  avg(case when dep_delay > 15 then 1 else 0 end) as pct_late,
  avg(case when cancelled = 'Y' then 1 else 0 end) as cancellation_rate
order by age_bucket
```

<BarChart
  data=age_vs_delay
  x=age_bucket
  y=avg_dep_delay
  title="Avg departure delay by aircraft age (5-year buckets)"
/>

The pattern refuses to be linear. Planes in the 0–5 year bracket average **7.7 minutes** late. Planes aged 10–15 are actually *better* at **5.8 minutes**. There's a jump at 25+ years, but that bucket has only about 1,400 flights — probably a handful of carriers still running legacy regional aircraft, not a statistically meaningful signal.

If aircraft age were a primary driver of delays, you'd see a steady climb left to right. You don't.

## Cancellation rates tell a similar story

<BarChart
  data=age_vs_delay
  x=age_bucket
  y=cancellation_rate
  title="Cancellation rate by aircraft age"
/>

The 10–20 year band has slightly elevated cancellation rates (~1.1–1.6%), while brand-new planes cancel at only 0.3%. There's a hint of a signal here, but it's modest and non-monotonic. The 25–30 year cohort cancels at roughly the same rate as 5–10 year old aircraft. Mechanical reliability isn't the dominant force behind cancellations in this dataset either.

## Who's flying what

```gsql manufacturer_perf
from flights
where cancelled = 'N' and aircraft.aircraft_models.manufacturer is not null
  and aircraft_age is not null and aircraft_age between 0 and 35
select
  case
    when aircraft.aircraft_models.manufacturer like '%BOEING%' then 'Boeing'
    when aircraft.aircraft_models.manufacturer like '%AIRBUS%' then 'Airbus'
    when aircraft.aircraft_models.manufacturer like '%MCDONNELL%' then 'McDonnell Douglas'
    when aircraft.aircraft_models.manufacturer like '%EMBRAER%' then 'Embraer'
    when aircraft.aircraft_models.manufacturer like '%BOMBARDIER%' or aircraft.aircraft_models.manufacturer like '%CANADAIR%' then 'Bombardier'
    when aircraft.aircraft_models.manufacturer like '%AEROSPATIALE%' then 'ATR/Aerospatiale'
    else null
  end as manufacturer,
  count() as flights,
  round(avg(aircraft_age), 1) as avg_age,
  round(avg(dep_delay), 1) as avg_dep_delay
having manufacturer is not null
order by flights desc
```

<Table data=manufacturer_perf title="Performance by manufacturer family">
  <Column id=manufacturer title="Manufacturer" />
  <Column id=flights fmt=num0 />
  <Column id=avg_age title="Avg Age (yrs)" />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" />
</Table>

Boeing dominates with 182K flights across the period — not surprising given Southwest's massive 737 operation. McDonnell Douglas aircraft average 13.6 years and post 8.2 minutes of delay. Airbus aircraft average just 4.6 years (these are mostly newer A319/A320 deliveries in the mid-2000s) and average 6.3 minutes. That 2-minute gap between the youngest manufacturer family and the oldest might tempt a "see, age matters" conclusion — but the confounds are substantial. Carriers that fly Airbus (US Airways, JetBlue, Frontier) have different route structures and hub footprints than carriers that inherited MD-80 fleets (American, Delta). You can't cleanly separate manufacturer from operator.

## The carrier view: young fleets, bad delays

```gsql carrier_fleet_age
from flights
where cancelled = 'N' and aircraft_age is not null and aircraft_age between 0 and 35
select
  carriers.nickname as carrier,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  round(avg(dep_delay), 1) as avg_dep_delay,
  count() as flights
having count() > 5000
order by avg_fleet_age desc
```

<ECharts data=carrier_fleet_age height=340px>
  title: {text: 'Fleet age vs. departure delay by carrier'},
  tooltip: {trigger: 'item', formatter: '{b}<br/>Avg fleet age: {@avg_fleet_age} yrs<br/>Avg delay: {@avg_dep_delay} min'},
  grid: {left: 60, right: 30, bottom: 60, top: 50},
  xAxis: {
    type: 'value',
    name: 'Avg fleet age (years)',
    nameLocation: 'middle',
    nameGap: 35,
    min: 0,
    max: 20,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 45,
    min: 0,
    max: 15,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  series: [
    {
      type: 'scatter',
      symbolSize: 12,
      encode: {x: 'avg_fleet_age', y: 'avg_dep_delay', itemName: 'carrier'},
      itemStyle: {color: '#3D6B7E'},
      label: {
        show: true,
        position: 'right',
        formatter: '{b}',
        fontSize: 10,
        color: '#555',
      },
    }
  ],
</ECharts>

This scatter plot is the clearest refutation of the age hypothesis. **Alaska Airlines** flies one of the younger fleets in the dataset (avg 8.4 years) yet posts the worst average departure delays at 12.3 minutes. **Continental** flies the oldest fleet (avg 16.4 years) and sits comfortably in the middle of the pack at 6.0 minutes. If age were driving delays, the points would run bottom-left to top-right. Instead, they're scattered.

Southwest's position is also instructive: a young, homogeneous fleet of 737s (avg 5.3 years) still averages 9.2 minutes late — worse than American or Delta with much older fleets. Southwest's hub-less, high-frequency model creates its own delay pressures independent of mechanical reliability.

## What's actually going on

Aircraft age isn't irrelevant. A 25-year-old plane likely has more unscheduled maintenance events than a 3-year-old one. But in commercial aviation, airlines schedule maintenance proactively — planes don't fly when they're not airworthy. What shows up in the delay data isn't mechanical failure so much as the operational factors analyzed in [What makes your flight late?](/pages/delay_factors): time of day, network structure, and hub congestion.

An airline with an old fleet but a point-to-point route structure (Continental Express) will out-run an airline with a young fleet but complex hub dependencies on the on-time scorecard. The 737 you're boarding this evening isn't going to be late because it was built in 1998. It's going to be late because the inbound plane is already 40 minutes behind in Denver.
