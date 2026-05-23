---
title: Top Routes — Performance Dashboard
layout: dashboard
---

```gsql kpis
from flights
select
  count(distinct origin || '-' || destination) as route_count,
  count() as total_flights,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay
```

<Row>
  <BigValue data=kpis value=route_count title="Distinct Routes" fmt=num0 />
  <BigValue data=kpis value=total_flights title="Total Flights" fmt=num0 />
  <BigValue data=kpis value=on_time_arrival_rate title="On-Time Arrival" />
  <BigValue data=kpis value=cancellation_rate title="Cancellation Rate" />
</Row>

```gsql top_routes
from flights
select
  origin || ' → ' || destination as route,
  origin_airport.city || ', ' || origin_airport.state as origin_city,
  destination_airport.city || ', ' || destination_airport.state as dest_city,
  carriers.nickname as top_carrier,
  count() as total_flights,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(distance), 0) as distance_mi
having count() > 500
order by total_flights desc
limit 25
```

<BarChart
  data=top_routes
  x=route
  y=total_flights
  title="25 Busiest Routes by Flight Volume"
  sort="total_flights desc"
/>

```gsql distance_vs_delay
from flights
where not is_cancelled
select
  origin || '-' || destination as route,
  round(avg(distance), 0) as distance_mi,
  round(avg(arr_delay), 1) as avg_arr_delay,
  count() as flights
having count() > 300
```

<ECharts data=distance_vs_delay height=440px>
  title: {text: 'Distance vs Arrival Delay — Does Flying Farther Mean Arriving Later?'},
  tooltip: {trigger: 'item'},
  grid: {top: 40, right: 20, bottom: 60, left: 65},
  visualMap: {
    dimension: 'flights',
    type: 'continuous',
    min: 300, max: 30000,
    inRange: {symbolSize: [4, 28], color: ['#9ecae1', '#3D6B7E']},
    show: false,
  },
  xAxis: {
    type: 'value',
    name: 'Route Distance (miles)',
    nameLocation: 'middle',
    nameGap: 35,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  yAxis: {
    type: 'value',
    name: 'Avg Arrival Delay (min)',
    nameLocation: 'middle',
    nameGap: 45,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  series: [{
    type: 'scatter',
    encode: {x: 'distance_mi', y: 'avg_arr_delay', itemName: 'route'},
    itemStyle: {opacity: 0.65},
    tooltip: {formatter: '{b}'},
  }],
</ECharts>

```gsql routes_table
from flights
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' (' || origin || ')' as origin,
  destination_airport.city || ' (' || destination || ')' as destination,
  count() as total_flights,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay_min,
  round(avg(arr_delay), 1) as avg_arr_delay_min,
  round(avg(distance), 0) as distance_mi
having count() > 500
order by total_flights desc
limit 50
```

<Table data=routes_table title="Top 50 Routes — Full Breakdown" rows=50>
  <Column id=route />
  <Column id=origin />
  <Column id=destination />
  <Column id=total_flights title="Flights" fmt=num0 />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  <Column id=avg_dep_delay_min title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=avg_arr_delay_min title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
</Table>
