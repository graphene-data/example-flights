---
title: "The Aircraft Age Myth: Do Older Planes Fly Worse?"
layout: notebook
---

Conventional traveler wisdom: avoid old planes. The intuition is reasonable — older machinery breaks down more, right? But in commercial aviation, aircraft face mandatory maintenance intervals, airworthiness inspections, and part replacements on schedules that have nothing to do with a plane's birthday. Age on the FAA registry might mean very little operationally.

This notebook uses FAA data from 2000–2005 to test whether aircraft age actually predicts departure delay performance — and where the real variance lies.

## How old is the U.S. commercial fleet?

```gsql fleet_age_dist
from aircraft
where flights.cancelled = 'N'
select
  (extract(year from flights.dep_time)::integer - year_built)::integer as age_years,
  count(flights.id2) as flights
having age_years >= 0 and age_years <= 50
order by age_years
```

<AreaChart
  data=fleet_age_dist
  x=age_years
  y=flights
  title="Flights flown by aircraft age (years at time of flight)"
/>

The fleet skews young but has a long tail. Most flights are on aircraft 5–20 years old, with meaningful volume out to 30+ years. That spread gives us enough coverage to see whether age is actually predictive.

## The blunt question: does age predict delay?

```gsql age_vs_delay
from aircraft
where flights.cancelled = 'N'
select
  (extract(year from flights.dep_time)::integer - year_built)::integer as age_years,
  round(avg(flights.dep_delay), 2) as avg_dep_delay,
  count(flights.id2) as n_flights
having age_years >= 0 and age_years <= 40 and n_flights > 500
order by age_years
```

<ECharts data=age_vs_delay height=380px>
  title: {text: 'Avg departure delay by aircraft age'},
  tooltip: {trigger: 'axis'},
  grid: {left: 60, right: 40, bottom: 50},
  xAxis: {
    type: 'value',
    name: 'Aircraft age (years)',
    nameLocation: 'middle',
    nameGap: 30,
    min: 0,
    max: 40,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 42,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  visualMap: {
    dimension: 'n_flights',
    type: 'continuous',
    min: 500,
    max: 15000,
    inRange: {symbolSize: [4, 20]},
    show: false,
  },
  series: [
    {
      type: 'scatter',
      encode: {x: 'age_years', y: 'avg_dep_delay', itemName: 'age_years'},
      itemStyle: {color: '#3D6B7E', opacity: 0.75},
    }
  ],
</ECharts>

No clean trend. If older aircraft reliably produced more delays, we'd see a rising slope from left to right. Instead, the scatter is essentially flat with noise — younger planes have some of the worst single-age averages, and some of the oldest aircraft groups fly with below-average delays. The dot size encodes flight volume; the high-volume ages cluster in the middle and show very similar performance.

## Binning by age group: the pattern stays flat

```gsql age_buckets
from aircraft
where flights.cancelled = 'N'
select
  case
    when (extract(year from flights.dep_time)::integer - year_built) < 5 then '0–4 yrs'
    when (extract(year from flights.dep_time)::integer - year_built) < 10 then '5–9 yrs'
    when (extract(year from flights.dep_time)::integer - year_built) < 15 then '10–14 yrs'
    when (extract(year from flights.dep_time)::integer - year_built) < 20 then '15–19 yrs'
    when (extract(year from flights.dep_time)::integer - year_built) < 25 then '20–24 yrs'
    when (extract(year from flights.dep_time)::integer - year_built) < 30 then '25–29 yrs'
    else '30+ yrs'
  end as age_group,
  case
    when (extract(year from flights.dep_time)::integer - year_built) < 5 then 0
    when (extract(year from flights.dep_time)::integer - year_built) < 10 then 1
    when (extract(year from flights.dep_time)::integer - year_built) < 15 then 2
    when (extract(year from flights.dep_time)::integer - year_built) < 20 then 3
    when (extract(year from flights.dep_time)::integer - year_built) < 25 then 4
    when (extract(year from flights.dep_time)::integer - year_built) < 30 then 5
    else 6
  end as sort_key,
  round(avg(flights.dep_delay), 1) as avg_dep_delay,
  round(avg(case when flights.dep_delay > 15 then 1 else 0 end) * 100, 1) as pct_delayed,
  count(flights.id2) as n_flights
having (extract(year from flights.dep_time)::integer - year_built) >= 0
order by sort_key
```

<Row>
  <BarChart data=age_buckets x=age_group y=avg_dep_delay sort="sort_key asc"
    title="Avg departure delay by age group (min)" />
  <BarChart data=age_buckets x=age_group y=pct_delayed sort="sort_key asc"
    title="% flights delayed >15 min by age group" />
</Row>

Both metrics tell the same story: there's no meaningful penalty for age group. The 20–24 year cohort actually posts the lowest average delay in the dataset. The 30+ cohort, which you might expect to be the worst, is comfortably mid-pack.

## Manufacturer matters more than age

```gsql manufacturer_perf
from aircraft_models
where aircraft.flights.cancelled = 'N'
select
  manufacturer,
  round(avg(aircraft.flights.dep_delay), 1) as avg_dep_delay,
  round(avg(case when aircraft.flights.dep_delay > 15 then 1 else 0 end) * 100, 1) as pct_delayed,
  count(aircraft.flights.id2) as n_flights
having n_flights > 5000
order by avg_dep_delay asc
```

<Table data=manufacturer_perf title="Delay performance by aircraft manufacturer (min 5,000 flights)">
  <Column id=manufacturer />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=pct_delayed title="% Delayed >15 min" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=n_flights title="Flights" fmt=num0 />
</Table>

There's more spread here than in the age buckets, though some of it reflects route mix and carrier assignment rather than pure mechanical reliability. Boeing and Airbus aircraft, which fly the bulk of commercial volume, are clustered close together — differences of less than a minute.

## The top-volume models

```gsql model_perf
from aircraft_models
where aircraft.flights.cancelled = 'N'
select
  manufacturer || ' ' || model as aircraft_model,
  round(avg(aircraft.flights.dep_delay), 1) as avg_dep_delay,
  round(avg(case when aircraft.flights.dep_delay > 15 then 1 else 0 end) * 100, 1) as pct_delayed,
  round(avg(extract(year from aircraft.flights.dep_time)::integer - aircraft.year_built), 1) as avg_age,
  count(aircraft.flights.id2) as n_flights
having n_flights > 20000
order by n_flights desc
```

<Table data=model_perf title="High-volume aircraft models (min 20,000 flights)">
  <Column id=aircraft_model title="Model" />
  <Column id=n_flights title="Flights" fmt=num0 />
  <Column id=avg_age title="Avg Age (yrs)" fmt="0.0" />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=pct_delayed title="% Delayed >15 min" fmt="0.0" contentType=colorscale colorScale=negative />
</Table>

These are the workhorses: 737s, 757s, MD-80s, regional jets. Their average ages range from around 10 to 25 years across this dataset, yet their delay rates sit within a narrow 2–3 minute band. The Boeing 737 variants — which span enormous age ranges depending on the specific subtype — show no meaningful differentiation.

## Does age matter within the same airline?

If the age effect were real, we'd expect it to show up when controlling for carrier — an airline flying its old planes should see worse performance on those aircraft. Here's the average aircraft age and delay performance per carrier:

```gsql carrier_age_perf
from flights
where cancelled = 'N'
select
  carriers.nickname as carrier,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(case when dep_delay > 15 then 1 else 0 end) * 100, 1) as pct_delayed,
  count() as n_flights
order by avg_fleet_age asc
```

<ECharts data=carrier_age_perf height=360px>
  title: {text: 'Carrier average fleet age vs departure delay'},
  tooltip: {trigger: 'item'},
  grid: {left: 60, right: 30, bottom: 60},
  xAxis: {
    type: 'value',
    name: 'Avg fleet age at time of flight (years)',
    nameLocation: 'middle',
    nameGap: 36,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 42,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  visualMap: {
    dimension: 'n_flights',
    type: 'continuous',
    min: 1000,
    max: 60000,
    inRange: {symbolSize: [6, 24]},
    show: false,
  },
  series: [
    {
      type: 'scatter',
      encode: {x: 'avg_fleet_age', y: 'avg_dep_delay', itemName: 'carrier'},
      itemStyle: {color: '#3D6B7E', opacity: 0.8},
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

Still no slope. Carriers with the oldest fleets aren't clustering at the top of the y-axis. Dot size encodes flight volume. The largest carriers (biggest dots) span a wide range of fleet ages and delay rates with no discernible pattern.

## The verdict

Aircraft age is not a useful predictor of departure delay performance in this dataset. The variance across age groups is essentially noise — within the same 2–3 minute band that separates the best from worst carrier, split eight different ways.

The mechanics behind this non-finding are worth understanding. FAA airworthiness requirements mandate component replacement schedules that disconnect "age of airframe" from "probability of malfunction on a given day." A 25-year-old 737 is flying with engines, avionics, and control surfaces that may be only a few years old. The regulatory model treats aircraft more like ships than cars.

What this means practically: the age label on a plane is, at best, a proxy for the carrier's fleet strategy — older fleets tend to appear on regional routes with lower margins, and those routes face different congestion patterns. But when you look at the data, age itself contributes essentially nothing to predicting whether your departure will be delayed.

Pick your flight time. The rest is noise.
