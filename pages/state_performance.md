---
title: State-by-State Flight Performance
layout: dashboard
---

```gsql nat_kpis
from flights
select
  count() as total_flights,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay
```

<Row>
  <BigValue data=nat_kpis value=total_flights title="Total Flights" fmt=num0 />
  <BigValue data=nat_kpis value=on_time_arrival_rate title="On-Time Arrival (National)" />
  <BigValue data=nat_kpis value=cancellation_rate title="Cancellation Rate (National)" />
  <BigValue data=nat_kpis value=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
</Row>

```gsql delay_leaders
from flights
where origin_airport.state is not null and origin_airport.fac_type = 'AIRPORT'
select
  origin_airport.state as state,
  round(avg(dep_delay), 1) as avg_dep_delay
having count() > 5000
order by avg_dep_delay desc
limit 15
```

```gsql cancel_leaders
from flights
where origin_airport.state is not null and origin_airport.fac_type = 'AIRPORT'
select
  origin_airport.state as state,
  round(cancellation_rate * 100, 2) as cancel_pct
having count() > 5000
order by cancel_pct desc
limit 15
```

<Row>
  <ECharts data=delay_leaders height=440px>
    title: {text: 'States: highest avg departure delay (min)', left: 'left'},
    tooltip: {trigger: 'axis', axisPointer: {type: 'shadow'}},
    grid: {left: 40, right: 70, top: 45, bottom: 20},
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
      encode: {y: 'state'},
    },
    series: [{
      type: 'bar',
      encode: {x: 'avg_dep_delay', y: 'state'},
      itemStyle: {color: '#C87F5A'},
      label: {show: true, position: 'right', formatter: '{c} min', fontSize: 11, color: '#666'},
    }]
  </ECharts>

  <ECharts data=cancel_leaders height=440px>
    title: {text: 'States: highest cancellation rate (%)', left: 'left'},
    tooltip: {trigger: 'axis', axisPointer: {type: 'shadow'}},
    grid: {left: 40, right: 65, top: 45, bottom: 20},
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
      encode: {y: 'state'},
    },
    series: [{
      type: 'bar',
      encode: {x: 'cancel_pct', y: 'state'},
      itemStyle: {color: '#B87470'},
      label: {show: true, position: 'right', formatter: '{c}%', fontSize: 11, color: '#666'},
    }]
  </ECharts>
</Row>

```gsql state_stats
from flights
where origin_airport.state is not null and origin_airport.fac_type = 'AIRPORT'
select
  origin_airport.state as state,
  count() as total_flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_arrival_rate,
  cancellation_rate,
  count(distinct origin) as airports,
  count(distinct carrier) as carriers
having total_flights > 5000
order by avg_dep_delay desc
```

<Table data=state_stats title="All States — Flight Performance (ranked by avg departure delay)" rows=60>
  <Column id=state title="State" />
  <Column id=total_flights title="Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=on_time_arrival_rate title="On-Time Arrival" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=cancellation_rate title="Cancel Rate" fmt=pct2 contentType=colorscale colorScale=negative />
  <Column id=airports title="Airports" fmt=num0 />
  <Column id=carriers title="Carriers" fmt=num0 />
</Table>
