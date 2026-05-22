---
title: Which Routes Can You Actually Trust?
layout: notebook
---

The advice you hear most often — "fly early, avoid the big hubs" — treats delay as a property of airports and time. But once you drill down to individual routes, a more granular picture emerges: some city-to-city connections are structurally reliable regardless of when you fly them, and some are reliably unreliable.

Using FAA data from 2000–2005, I looked at every route with at least 500 recorded flights and ranked them by on-time arrival rate. The spread is wider than you might expect.

## Not all routes are created equal

```gsql route_buckets
with routes as (
  from flights
  where cancelled = 'N'
  select
    origin,
    destination,
    on_time_arrival_rate,
    count() as n
  having count() > 300
)
select
  (floor(on_time_arrival_rate * 10) / 10 * 100)::integer as pct_floor,
  count() as route_count
from routes
group by pct_floor
order by pct_floor
```

<BarChart
  data=route_buckets
  x=pct_floor
  y=route_count
  title="Routes by on-time arrival rate bucket (% of flights arriving on time)"
/>

Routes cluster around 60–75% — but the tails are real. Some routes arrive on time more than 85% of the time. Others manage barely 40%. A 45-point spread across routes that all fly the same U.S. airspace tells you something structural is going on.

## The 15 most reliable routes

```gsql top_routes
from flights
where cancelled = 'N'
select
  origin_airport.city || ', ' || origin_airport.state as origin,
  destination_airport.city || ', ' || destination_airport.state as destination,
  count() as flights,
  on_time_arrival_rate,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(avg(distance), 0) as distance_mi
having count() > 500
order by on_time_arrival_rate desc
limit 15
```

<Table data=top_routes title="Most reliable routes (≥ 500 flights)" rows=15>
  <Column id=origin />
  <Column id=destination />
  <Column id=flights fmt=num0 />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
</Table>

Several patterns jump out. High-reliability routes tend to be medium-haul (300–900 miles) — long enough that the crew has room to recover a minor departure slip in the cruise phase, but not so long that the plane has absorbed a full day of cascading delays before wheels-up. The airports at both ends also tend to be secondary markets with lower traffic density.

## The 15 worst routes

```gsql worst_routes
from flights
where cancelled = 'N'
select
  origin_airport.city || ', ' || origin_airport.state as origin,
  destination_airport.city || ', ' || destination_airport.state as destination,
  count() as flights,
  on_time_arrival_rate,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(avg(distance), 0) as distance_mi
having count() > 500
order by on_time_arrival_rate asc
limit 15
```

<Table data=worst_routes title="Least reliable routes (≥ 500 flights)" rows=15>
  <Column id=origin />
  <Column id=destination />
  <Column id=flights fmt=num0 />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
</Table>

The worst routes almost universally share one trait: at least one endpoint is a major northeastern hub — EWR, PHL, JFK, or ORD. These airports sit in the densest and most weather-sensitive airspace in the country. A flight from Denver to Newark inherits Newark's slot congestion as soon as it's wheels-up; there's nothing the pilot can do to outrun arrival sequencing.

## Does flying farther protect you?

```gsql distance_vs_rate
from flights
where cancelled = 'N'
select
  origin,
  destination,
  round(avg(distance), 0) as avg_distance,
  on_time_arrival_rate,
  count() as flights
having count() > 300
order by avg_distance
```

<ECharts data=distance_vs_rate height=420px>
  title: {text: 'Route distance vs. on-time arrival rate'},
  tooltip: {trigger: 'item'},
  grid: {left: 60, right: 30, bottom: 55},
  xAxis: {
    type: 'value',
    name: 'Avg route distance (mi)',
    nameLocation: 'middle',
    nameGap: 35,
    min: 0,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: {
    type: 'value',
    name: 'On-time arrival rate',
    nameLocation: 'middle',
    nameGap: 45,
    min: 0.3,
    max: 1.0,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  series: [{
    type: 'scatter',
    encode: {x: 'avg_distance', y: 'on_time_arrival_rate'},
    symbolSize: 5,
    itemStyle: {color: '#3D6B7E', opacity: 0.55},
  }],
</ECharts>

The scatter is messy — no clean linear story here. Short hops (under 300 miles) show the widest variance: some are extremely reliable, others are among the worst in the dataset. The cluster of sub-200-mile routes in the 45–55% on-time band are almost entirely shuttle runs into congested northeastern airports. Strip those out and there's actually a mild positive relationship between distance and reliability — longer routes have more ceiling to absorb delays, and they tend to overfly rather than sequence through the worst chokepoints.

Routes above 1,500 miles tighten up noticeably. Transcontinental flights arrive on time 65–80% of the time, with few outliers. Partly that's route structure (fewer connections, more direct flight paths); partly it's that a cross-country flight has two to three hours of buffer to recover a 15-minute departure slip.

## What it means

**Destination matters more than origin.** The worst arrival airports — Newark, Philadelphia, O'Hare — stamp their delays onto inbound flights regardless of where those flights departed. Choosing your destination airport is something most travelers don't think about when they're booking, but this data argues you should.

**Mid-range distances have the best risk/reward profile.** Flights in the 400–1,200 mile range combine enough block time to recover delays with enough frequency to smooth out variance.

**Short does not mean safe.** The shortest routes have the highest variance and include the dataset's worst performers. If you're flying under 300 miles and one endpoint is a northeastern hub, budget time for the delay.
