---
title: Airport Performance
layout: dashboard
---

```gsql totals
from flights
select
  count(distinct origin) as airports_tracked,
  count() as total_departures,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay_min
```

<Row>
  <BigValue data=totals value=airports_tracked title="Airports Tracked" />
  <BigValue data=totals value=total_departures title="Total Departures" fmt=num0 />
  <BigValue data=totals value=on_time_arrival_rate title="On-Time Arrival" fmt=pct1 />
  <BigValue data=totals value=cancellation_rate title="Cancellation Rate" fmt=pct2 />
  <BigValue data=totals value=avg_dep_delay_min title="Avg Dep Delay (min)" />
</Row>

```gsql top_airports
from flights
select
  origin as code,
  origin_airport.city as city,
  origin_airport.state as state,
  count() as departures,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(arr_delay), 1) as avg_arr_delay
having departures > 2000
order by departures desc
```

```gsql state_delays
from flights
select
  origin_airport.state as state,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_arrival_rate,
  cancellation_rate
having flights > 2000
order by avg_dep_delay desc
limit 20
```

<Row>
  <BarChart
    title="Avg Departure Delay by State (Top 20 by Delay)"
    data=state_delays
    x=avg_dep_delay
    y=state
    sort="avg_dep_delay desc"
    height=420px
    width=50%
  />

  <ECharts data=top_airports height=420px width=50%>
    title: {text: 'Departure Delay vs Cancellation Rate by Airport'},
    tooltip: {trigger: 'item'},
    grid: {top: 40, right: 20, bottom: 60, left: 60},
    visualMap: {
      dimension: 'departures',
      type: 'continuous',
      min: 0, max: 18000,
      inRange: {symbolSize: [5, 36]},
      show: false,
    },
    xAxis: {
      type: 'value',
      name: 'Avg Departure Delay (min)',
      nameLocation: 'middle',
      nameGap: 30,
      axisLine: {show: false},
      axisTick: {show: false},
    },
    yAxis: {
      type: 'value',
      name: 'Cancellation Rate',
      nameLocation: 'middle',
      nameGap: 45,
      axisLine: {show: false},
      axisTick: {show: false},
    },
    series: [{
      type: 'scatter',
      encode: {x: 'avg_dep_delay', y: 'cancellation_rate', itemName: 'code'},
      itemStyle: {opacity: 0.75},
      label: {show: false},
    }],
  </ECharts>
</Row>

```gsql yearly_trend
from flights
select
  extract(year from dep_time) as year,
  round(avg(dep_delay), 2) as avg_dep_delay,
  on_time_arrival_rate,
  cancellation_rate
order by year
```

```gsql airport_ranking
from flights
select
  origin as code,
  origin_airport.city as city,
  origin_airport.state as state,
  count() as departures,
  round(avg(dep_delay), 1) as avg_dep_delay,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(arr_delay), 1) as avg_arr_delay,
  count(distinct carriers.code) as carriers_served
having departures > 1000
order by departures desc
```

<Row>
  <LineChart
    title="Network-Wide Performance Trend (2000–2005)"
    data=yearly_trend
    x=year
    y="avg_dep_delay, on_time_arrival_rate"
    height=280px
    width=50%
  />

  <AreaChart
    title="Cancellation Rate by Year"
    data=yearly_trend
    x=year
    y=cancellation_rate
    height=280px
    width=50%
  />
</Row>

<Table
  data=airport_ranking
  title="Airport Performance (>1,000 departures)"
  sortable=true
  sort="departures desc"
  rows=50
>
  <Column id=code title="Code" />
  <Column id=city title="City" />
  <Column id=state title="State" />
  <Column id=departures title="Departures" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=carriers_served title="Carriers" />
</Table>
