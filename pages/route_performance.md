---
title: Route Performance
layout: dashboard
---

```gsql route_summary
from flights
select
  origin || ' → ' || destination as route,
  origin as origin_code,
  destination as dest_code,
  origin_airport.city as origin_city,
  destination_airport.city as dest_city,
  count() as total_flights,
  count(distinct carriers.code) as num_carriers,
  round(avg(distance), 0) as distance_mi,
  avg(case when cancelled = 'N' and arr_delay <= 0 then 1.0 when cancelled = 'N' then 0.0 end) as ontime_rate,
  avg(case when cancelled = 'N' then dep_delay end) as avg_dep_delay,
  avg(case when cancelled = 'N' then arr_delay end) as avg_arr_delay,
  avg(case when cancelled = 'Y' then 1.0 else 0.0 end) as cancel_rate
having count() >= 50
order by total_flights desc
limit 500
```

```gsql kpi_totals
from flights
select
  count(distinct origin || destination) as unique_routes,
  count() as total_flights,
  avg(case when cancelled = 'N' and arr_delay <= 0 then 1.0 when cancelled = 'N' then 0.0 end) as system_ontime,
  avg(case when cancelled = 'Y' then 1.0 else 0.0 end) as system_cancel_rate
```

<Row>
  <BigValue data=kpi_totals value=unique_routes title="Routes (≥50 flights)" />
  <BigValue data=kpi_totals value=total_flights title="Total Flights" fmt=num0 />
  <BigValue data=kpi_totals value=system_ontime title="System On-Time Rate" fmt=pct1 />
  <BigValue data=kpi_totals value=system_cancel_rate title="System Cancel Rate" fmt=pct2 />
</Row>

```gsql top_routes_by_volume
from route_summary
select route, origin_city, dest_city, total_flights, ontime_rate, avg_dep_delay, cancel_rate, distance_mi, num_carriers
order by total_flights desc
limit 20
```

<Table data=top_routes_by_volume title="20 Busiest Routes" rows=20>
  <Column id=route title="Route" />
  <Column id=origin_city title="From" />
  <Column id=dest_city title="To" />
  <Column id=total_flights title="Flights" fmt=num0 />
  <Column id=distance_mi title="Miles" fmt=num0 />
  <Column id=num_carriers title="Carriers" />
  <Column id=ontime_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=cancel_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
</Table>

```gsql distance_vs_delay
from route_summary
select
  route,
  distance_mi,
  avg_dep_delay,
  total_flights,
  num_carriers,
  case
    when distance_mi < 500 then 'Short (<500 mi)'
    when distance_mi < 1500 then 'Medium (500–1500 mi)'
    else 'Long (>1500 mi)'
  end as haul_type
order by total_flights desc
limit 200
```

```gsql volume_by_haul
from distance_vs_delay
select
  haul_type,
  min(distance_mi) as min_dist,
  count() as routes,
  sum(total_flights) as flights,
  avg(avg_dep_delay) as avg_delay
order by min_dist
```

<Row>
  <ECharts data=distance_vs_delay height=420px>
    title: {text: 'Route Distance vs Avg Departure Delay'},
    tooltip: {trigger: 'item', formatter: '{b}'},
    grid: {top: 40, bottom: 60},
    xAxis: {
      type: 'value',
      name: 'Distance (miles)',
      nameLocation: 'middle',
      nameGap: 28,
      splitLine: {lineStyle: {color: '#f0f0f0'}},
      axisLine: {show: false},
      axisTick: {show: false},
    },
    yAxis: {
      type: 'value',
      name: 'Avg Dep Delay (min)',
      nameLocation: 'middle',
      nameGap: 32,
      splitLine: {lineStyle: {color: '#f0f0f0'}},
      axisLine: {show: false},
      axisTick: {show: false},
    },
    visualMap: [
      {
        dimension: 'total_flights',
        type: 'continuous',
        min: 0, max: 600,
        inRange: {symbolSize: [4, 22]},
        show: false,
      },
      {
        dimension: 'distance_mi',
        type: 'continuous',
        min: 0, max: 2800,
        inRange: {color: ['#C87F5A', '#3D6B7E', '#6B9E5E']},
        show: true,
        orient: 'horizontal',
        bottom: 4,
        left: 'center',
        text: ['Long haul', 'Short haul'],
      }
    ],
    series: [
      {
        type: 'scatter',
        encode: {x: 'distance_mi', y: 'avg_dep_delay', itemName: 'route'},
        itemStyle: {opacity: 0.75},
      },
    ],
  </ECharts>

  <BarChart
    data=volume_by_haul
    x=haul_type
    y=flights
    title="Flight Volume by Haul Type"
    height=420px
  />
</Row>

```gsql competitive_routes
from route_summary
where num_carriers >= 3
select route, origin_city, dest_city, total_flights, num_carriers, ontime_rate, avg_dep_delay, distance_mi
order by num_carriers desc, total_flights desc
limit 20
```

```gsql monopoly_routes
from route_summary
where num_carriers = 1
select route, origin_city, dest_city, total_flights, ontime_rate, avg_dep_delay, distance_mi
order by total_flights desc
limit 20
```

```gsql competition_vs_performance
from route_summary
select
  num_carriers,
  count() as routes,
  round(avg(total_flights), 0) as avg_flights_per_route,
  avg(ontime_rate) as avg_ontime_rate,
  avg(avg_dep_delay) as avg_delay
order by num_carriers
```

<Row>
  <ECharts data=competition_vs_performance height=320px>
    title: {text: 'On-Time Rate by Number of Competing Carriers'},
    tooltip: {trigger: 'axis'},
    grid: {left: 50, right: 30},
    xAxis: {
      type: 'category',
      name: 'Number of carriers on route',
      nameLocation: 'middle',
      nameGap: 28,
      encode: {x: 'num_carriers'},
    },
    yAxis: [
      {
        type: 'value',
        name: 'On-Time Rate',
        min: 0.5, max: 1.0,
        splitLine: {lineStyle: {color: '#f0f0f0'}},
        axisLine: {show: false}, axisTick: {show: false},
      },
      {
        type: 'value',
        name: 'Avg Delay (min)',
        axisLine: {show: false}, axisTick: {show: false},
        splitLine: {show: false},
      }
    ],
    series: [
      {
        type: 'bar',
        name: 'On-Time Rate',
        yAxisIndex: 0,
        encode: {x: 'num_carriers', y: 'avg_ontime_rate'},
        itemStyle: {color: '#3D6B7E'},
        barMaxWidth: 60,
      },
      {
        type: 'line',
        name: 'Avg Dep Delay',
        yAxisIndex: 1,
        encode: {x: 'num_carriers', y: 'avg_delay'},
        itemStyle: {color: '#C87F5A'},
        lineStyle: {width: 2.5},
        symbol: 'circle', symbolSize: 7,
      }
    ],
  </ECharts>

  <Table data=competitive_routes title="Most Competitive Routes (3+ Carriers)" rows=20>
    <Column id=route title="Route" />
    <Column id=num_carriers title="Carriers" />
    <Column id=total_flights title="Flights" fmt=num0 />
    <Column id=ontime_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Delay" fmt="0.0" />
  </Table>
</Row>

```gsql worst_ontime_routes
from route_summary
where total_flights >= 200
select route, origin_city, dest_city, total_flights, ontime_rate, avg_dep_delay, cancel_rate, distance_mi
order by ontime_rate asc
limit 15
```

```gsql best_ontime_routes
from route_summary
where total_flights >= 200
select route, origin_city, dest_city, total_flights, ontime_rate, avg_dep_delay, cancel_rate, distance_mi
order by ontime_rate desc
limit 15
```

<Row>
  <Table data=worst_ontime_routes title="Worst On-Time Routes (≥200 flights)" rows=15>
    <Column id=route title="Route" />
    <Column id=origin_city title="From" />
    <Column id=dest_city title="To" />
    <Column id=total_flights title="Flights" fmt=num0 />
    <Column id=ontime_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.0" />
    <Column id=cancel_rate title="Cancel %" fmt=pct2 />
  </Table>

  <Table data=best_ontime_routes title="Best On-Time Routes (≥200 flights)" rows=15>
    <Column id=route title="Route" />
    <Column id=origin_city title="From" />
    <Column id=dest_city title="To" />
    <Column id=total_flights title="Flights" fmt=num0 />
    <Column id=ontime_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.0" />
    <Column id=cancel_rate title="Cancel %" fmt=pct2 />
  </Table>
</Row>
