---
title: Route Performance
layout: dashboard
---

```gsql route_stats
from flights where not is_cancelled
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' (' || origin || ')' as origin_name,
  destination_airport.city || ' (' || destination || ')' as dest_name,
  count() as flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay,
  avg(arr_delay) as avg_arr_delay,
  avg(distance)::integer as distance_mi
having count() > 500
order by flights desc
```

```gsql kpis
from (
  from flights where not is_cancelled
  select
    origin || ' → ' || destination as route,
    count() as route_flights
  having count() > 500
)
select
  count() as total_routes,
  sum(route_flights) as total_flights_on_routes,
  max(route_flights) as peak_route_flights
```

<Row>
  <BigValue data=kpis value=total_routes title="Routes Analyzed (500+ flights)" />
  <BigValue data=kpis value=total_flights_on_routes title="Flights on These Routes" fmt=num0 />
  <BigValue data=kpis value=peak_route_flights title="Busiest Single Route" fmt=num0 />
</Row>

```gsql top_25_routes
from flights where not is_cancelled
select
  origin || ' → ' || destination as route,
  count() as flights,
  on_time_arrival_rate
having count() > 500
order by flights desc
limit 25
```

<Row>
  <BarChart
    data=top_25_routes
    x=route
    y=flights
    title="Top 25 Busiest Routes (non-cancelled flights)"
    sort="flights desc"
    height=520px
  />

  <ECharts data=route_stats height=520px>
    title: {text: 'On-Time Arrival Rate vs Distance by Route'},
    tooltip: {trigger: 'item'},
    grid: {top: 40, bottom: 60, left: 65, right: 20},
    visualMap: {
      dimension: 'flights',
      type: 'continuous',
      min: 500,
      max: 2100,
      inRange: {symbolSize: [8, 32]},
      show: false,
    },
    xAxis: {
      type: 'value',
      name: 'Average distance (mi)',
      nameLocation: 'middle',
      nameGap: 35,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    yAxis: {
      type: 'value',
      name: 'On-time arrival rate',
      nameLocation: 'middle',
      nameGap: 45,
      min: 0.3,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    series: [{
      type: 'scatter',
      encode: {x: 'distance_mi', y: 'on_time_arrival_rate', itemName: 'route', tooltip: 'flights'},
      itemStyle: {color: '#3D6B7E', opacity: 0.7},
      tooltip: {formatter: '{b}'},
    }],
  </ECharts>
</Row>

```gsql worst_routes
from flights where not is_cancelled
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' → ' || destination_airport.city as route_name,
  count() as flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay,
  avg(arr_delay) as avg_arr_delay,
  avg(distance)::integer as distance_mi
having count() > 500
order by on_time_arrival_rate asc
limit 15
```

```gsql best_routes
from flights where not is_cancelled
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' → ' || destination_airport.city as route_name,
  count() as flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay,
  avg(arr_delay) as avg_arr_delay,
  avg(distance)::integer as distance_mi
having count() > 500
order by on_time_arrival_rate desc
limit 15
```

<Row>
  <Table data=worst_routes title="Worst On-Time Routes (500+ flights)" rows=15>
    <Column id=route />
    <Column id=route_name title="City Pair" />
    <Column id=flights fmt=num0 />
    <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
    <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
    <Column id=distance_mi title="Distance (mi)" fmt=num0 />
  </Table>

  <Table data=best_routes title="Best On-Time Routes (500+ flights)" rows=15>
    <Column id=route />
    <Column id=route_name title="City Pair" />
    <Column id=flights fmt=num0 />
    <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
    <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
    <Column id=distance_mi title="Distance (mi)" fmt=num0 />
  </Table>
</Row>

```gsql all_routes
from flights where not is_cancelled
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' → ' || destination_airport.city as route_name,
  count() as flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay,
  avg(arr_delay) as avg_arr_delay,
  avg(distance)::integer as distance_mi
having count() > 500
order by flights desc
```

<Table data=all_routes title="All Routes (500+ flights)" rows=100>
  <Column id=route />
  <Column id=route_name title="City Pair" />
  <Column id=flights fmt=num0 />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
</Table>
