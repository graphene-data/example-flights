---
title: Route Performance
layout: dashboard
---

```gsql route_kpis
from flights
select
  count(distinct origin || '-' || destination) as total_routes,
  round(avg(distance), 0) as avg_distance_mi,
  count() as total_flights,
  round(avg(case when cancelled = 'N' then dep_delay end), 1) as avg_dep_delay
```

<Row>
  <BigValue data=route_kpis value=total_routes title="Unique Routes" fmt=num0 />
  <BigValue data=route_kpis value=avg_distance_mi title="Avg Route Distance (mi)" fmt=num0 />
  <BigValue data=route_kpis value=total_flights title="Total Flights" fmt=num0 />
  <BigValue data=route_kpis value=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
</Row>

```gsql top_routes
from flights where cancelled = 'N'
select
  origin || ' → ' || destination as route,
  origin_airport.city as origin_city,
  destination_airport.city as dest_city,
  count() as flights,
  round(avg(distance), 0) as distance_mi,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(avg(case when dep_delay <= 0 then 1.0 else 0.0 end) * 100, 1) as pct_on_time
having count() > 500
order by flights desc
limit 20
```

<Table data=top_routes title="Top 20 Busiest Routes (Non-Cancelled)" rows=20>
  <Column id=route title="Route" />
  <Column id=origin_city title="Origin City" />
  <Column id=dest_city title="Destination City" />
  <Column id=flights title="Flights" fmt=num0 />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=pct_on_time title="On-Time Dep %" fmt="0.0" contentType=colorscale colorScale=positive />
</Table>

```gsql delay_vs_distance
from flights where cancelled = 'N'
select
  origin || '-' || destination as route,
  round(avg(distance), 0) as distance_mi,
  round(avg(dep_delay), 1) as avg_dep_delay,
  count() as flights
having count() > 200
```

```gsql worst_routes
from flights where cancelled = 'N'
select
  origin_airport.city || ' → ' || destination_airport.city as route,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(case when dep_delay <= 0 then 1.0 else 0.0 end) * 100, 1) as pct_on_time
having count() > 300
order by avg_dep_delay desc
limit 15
```

```gsql best_routes
from flights where cancelled = 'N'
select
  origin_airport.city || ' → ' || destination_airport.city as route,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(case when dep_delay <= 0 then 1.0 else 0.0 end) * 100, 1) as pct_on_time
having count() > 300
order by avg_dep_delay asc
limit 15
```

<Row>
  <BarChart
    data=worst_routes
    x=avg_dep_delay
    y=route
    title="15 Worst Routes by Avg Departure Delay (min)"
    height=420px
  />
  <BarChart
    data=best_routes
    x=avg_dep_delay
    y=route
    title="15 Best Routes by Avg Departure Delay (min)"
    height=420px
  />
</Row>

```gsql delay_by_distance_bucket
from flights where cancelled = 'N'
select
  case
    when distance < 300 then 1
    when distance < 600 then 2
    when distance < 1000 then 3
    else 4
  end as sort_key,
  case
    when distance < 300 then '< 300 mi (Short)'
    when distance < 600 then '300–600 mi (Medium)'
    when distance < 1000 then '600–1000 mi (Long)'
    else '> 1000 mi (Cross-Country)'
  end as distance_band,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(avg(case when dep_delay <= 0 then 1.0 else 0.0 end) * 100, 1) as pct_on_time_dep,
  count() as flights
order by sort_key
```

<Row>
  <BarChart
    data=delay_by_distance_bucket
    x=distance_band
    y=avg_dep_delay
    title="Avg Departure Delay by Distance Band (min)"
    height=300px
  />
  <BarChart
    data=delay_by_distance_bucket
    x=distance_band
    y=pct_on_time_dep
    title="On-Time Departure Rate by Distance Band (%)"
    height=300px
  />
</Row>

```gsql cancellation_by_route_type
from flights
select
  case
    when distance < 300 then 1
    when distance < 600 then 2
    when distance < 1000 then 3
    else 4
  end as sort_key,
  case
    when distance < 300 then 'Short (< 300 mi)'
    when distance < 600 then 'Medium (300–600 mi)'
    when distance < 1000 then 'Long (600–1000 mi)'
    else 'Cross-Country (> 1000 mi)'
  end as distance_band,
  round(avg(case when cancelled = 'Y' then 1.0 else 0.0 end) * 100, 2) as cancellation_rate_pct,
  count() as flights
order by sort_key
```

<BarChart
  data=cancellation_by_route_type
  x=distance_band
  y=cancellation_rate_pct
  title="Cancellation Rate by Route Distance Band (%)"
  height=280px
/>

```gsql top_state_pairs
from flights where cancelled = 'N' and origin_airport.state <> destination_airport.state
select
  origin_airport.state || ' → ' || destination_airport.state as state_pair,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay
having count() > 1000
order by flights desc
limit 15
```

<Table data=top_state_pairs title="Busiest Interstate Corridors" rows=15>
  <Column id=state_pair title="State Corridor" />
  <Column id=flights title="Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
</Table>
