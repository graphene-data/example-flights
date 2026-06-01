---
title: U.S. Route Network Dashboard
layout: dashboard
---

```gsql totals
from flights
select
  count() as total_flights,
  count(distinct concat(least(origin, destination), '-', greatest(origin, destination))) as unique_corridors,
  count(distinct carrier) as carriers,
  count(distinct origin) as airports
```

<Row>
  <BigValue data=totals value=total_flights title="Total Scheduled Flights" />
  <BigValue data=totals value=unique_corridors title="Unique City-Pair Corridors" />
  <BigValue data=totals value=airports title="Airports" />
  <BigValue data=totals value=carriers title="Carriers" />
</Row>

```gsql top_corridors
from flights
select
  least(origin, destination) as a,
  greatest(origin, destination) as b,
  -- city labels: pick whichever direction shows up first
  first(case when origin < destination then origin_airport.city else destination_airport.city end) as city_a,
  first(case when origin < destination then destination_airport.city else origin_airport.city end) as city_b,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_departure_rate,
  cancellation_rate
order by flights desc
limit 20
```

```gsql top_airports
from flights
select
  origin as airport,
  origin_airport.city as city,
  origin_airport.state as state,
  count() as departures,
  on_time_departure_rate,
  round(avg(dep_delay), 1) as avg_dep_delay,
  cancellation_rate
order by departures desc
limit 15
```

```gsql carrier_share
from flights
select
  carriers.nickname as carrier,
  count() as flights,
  on_time_departure_rate,
  cancellation_rate,
  round(avg(distance), 0) as avg_distance_mi
order by flights desc
```

```gsql yoy_volume
from flights
select
  extract(year from dep_time)::integer as year,
  count() as total_flights,
  count(distinct carrier) as active_carriers,
  on_time_departure_rate,
  cancellation_rate
order by year
```

<Row>
  <Table data=top_corridors title="Top 20 Busiest Corridors (Both Directions)" rows=20>
    <Column id=city_a title="From" />
    <Column id=city_b title="To" />
    <Column id=flights fmt=num0 />
    <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.1" />
    <Column id=on_time_departure_rate title="On-Time Dep %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  </Table>

  <ECharts data=top_airports height=520px>
    title: {text: 'Top 15 Airports by Departures'},
    tooltip: {trigger: 'axis', axisPointer: {type: 'shadow'}},
    grid: {left: 90, right: 40, top: 40, bottom: 20},
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
      encode: {y: 'airport'},
    },
    series: [{
      type: 'bar',
      encode: {x: 'departures', y: 'airport'},
      itemStyle: {color: '#3D6B7E'},
      label: {
        show: true,
        position: 'right',
        formatter: '{@city}, {@state}',
        color: '#555',
        fontSize: 11,
      },
    }],
  </ECharts>
</Row>

```gsql long_vs_short
from flights
where not is_cancelled
select
  case
    when distance < 500 then 'Short-haul (<500 mi)'
    when distance < 1000 then 'Medium-haul (500–1000 mi)'
    when distance < 2000 then 'Long-haul (1000–2000 mi)'
    else 'Ultra-long (2000+ mi)'
  end as segment,
  case
    when distance < 500 then 1
    when distance < 1000 then 2
    when distance < 2000 then 3
    else 4
  end as segment_order,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_departure_rate,
  round(avg(distance), 0) as avg_distance_mi
order by segment_order
```

<Row>
  <BarChart
    data=yoy_volume
    x=year
    y=total_flights
    title="Flight Volume by Year"
  />

  <Table data=carrier_share title="Carrier Market Share" rows=20>
    <Column id=carrier />
    <Column id=flights fmt=num0 />
    <Column id=avg_distance_mi title="Avg Route (mi)" fmt=num0 />
    <Column id=on_time_departure_rate title="On-Time Dep %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  </Table>
</Row>

<Row>
  <Table data=long_vs_short title="Performance by Route Distance" rows=4>
    <Column id=segment title="Segment" />
    <Column id=flights fmt=num0 />
    <Column id=avg_distance_mi title="Avg Distance (mi)" fmt=num0 />
    <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.1" />
    <Column id=on_time_departure_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
  </Table>
</Row>
