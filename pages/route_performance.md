---
title: Route Performance
layout: dashboard
---

```gsql route_kpis
from flights
select count() as total_flights, on_time_arrival_rate, cancellation_rate, round(avg(dep_delay), 1) as avg_dep_delay
```

<Row>
  <BigValue data=route_kpis value=total_flights fmt=num0 title="Total Flights" />
  <BigValue data=route_kpis value=avg_dep_delay fmt="0.0" title="Avg Dep Delay (min)" />
  <BigValue data=route_kpis value=on_time_arrival_rate title="On-Time Arrival %" />
  <BigValue data=route_kpis value=cancellation_rate title="Cancellation Rate" />
</Row>

```gsql routes_ranked
from flights
where cancelled = 'N'
select
  origin || ' → ' || destination as route,
  origin_airport.city || ' → ' || destination_airport.city as city_route,
  count() as total_flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_arrival_rate,
  round(avg(distance), 0) as distance_mi,
  round(avg(arr_delay), 1) as avg_arr_delay
having count() > 200
order by total_flights desc
limit 40
```

<Table data=routes_ranked title="Busiest Routes (>200 flights)" rows=40>
  <Column id=route />
  <Column id=city_route title="Cities" />
  <Column id=total_flights title="Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=distance_mi title="Distance (mi)" fmt=num0 />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
</Table>

```gsql delay_vs_distance
from flights
where cancelled = 'N'
select
  origin || ' → ' || destination as route,
  round(avg(distance), 0) as distance_mi,
  round(avg(dep_delay), 1) as avg_dep_delay,
  count() as total_flights
having count() > 100
order by total_flights desc
```

<Row>
  <ECharts data=delay_vs_distance height=440px>
    title: {text: 'Departure Delay vs Route Distance'},
    tooltip: {trigger: 'item'},
    grid: {top: 40, left: 60, right: 30, bottom: 60},
    visualMap: {
      dimension: 'total_flights',
      type: 'continuous',
      min: 100, max: 2000,
      inRange: {symbolSize: [5, 28]},
      show: false,
    },
    xAxis: {
      type: 'value',
      name: 'Route Distance (miles)',
      nameLocation: 'middle',
      nameGap: 36,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    yAxis: {
      type: 'value',
      name: 'Avg Departure Delay (min)',
      nameLocation: 'middle',
      nameGap: 40,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    series: [{
      type: 'scatter',
      encode: {x: 'distance_mi', y: 'avg_dep_delay', itemName: 'route'},
      tooltip: {formatter: '{b}'},
      itemStyle: {color: '#3D6B7E', opacity: 0.7},
    }],
  </ECharts>

  <ECharts data=routes_ranked height=440px>
    title: {text: 'Top Routes — Avg Departure Delay (min)'},
    tooltip: {trigger: 'item'},
    grid: {left: 130, right: 90, top: 40, bottom: 20},
    xAxis: {
      type: 'value',
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    yAxis: {
      type: 'category',
      inverse: true,
      axisLine: {show: false},
      axisTick: {show: false},
      encode: {y: 'route'},
    },
    series: [{
      type: 'bar',
      encode: {x: 'avg_dep_delay', y: 'route'},
      itemStyle: {color: '#3D6B7E'},
      label: {show: true, position: 'right', formatter: '{@avg_dep_delay} min', fontSize: 10, color: '#555'},
    }],
  </ECharts>
</Row>

```gsql ontime_by_distance_bucket
from flights
where cancelled = 'N'
select
  case
    when distance < 250 then '< 250 mi'
    when distance < 500 then '250–500 mi'
    when distance < 750 then '500–750 mi'
    when distance < 1000 then '750–1000 mi'
    when distance < 1500 then '1000–1500 mi'
    else '1500+ mi'
  end as distance_bucket,
  case
    when distance < 250 then 1
    when distance < 500 then 2
    when distance < 750 then 3
    when distance < 1000 then 4
    when distance < 1500 then 5
    else 6
  end as sort_key,
  count() as flights,
  on_time_arrival_rate,
  round(avg(dep_delay), 1) as avg_dep_delay
order by sort_key
```

<Row>
  <BarChart
    data=ontime_by_distance_bucket
    x=distance_bucket
    y=on_time_arrival_rate
    title="On-Time Arrival Rate by Route Distance"
    sort="sort_key asc"
  />
  <BarChart
    data=ontime_by_distance_bucket
    x=distance_bucket
    y=avg_dep_delay
    title="Avg Departure Delay by Route Distance"
    sort="sort_key asc"
  />
</Row>
