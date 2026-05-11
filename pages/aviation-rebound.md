---
title: The 9/11 Rebound - U.S. Aviation 2000-2005
layout: dashboard
---

```gsql overall_kpis
from flights
select
  count() as total_flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay
```

```gsql sep_2001_kpis
from flights
where extract(year from dep_time)::int = 2001 and extract(month from dep_time)::int = 9
select cancellation_rate
```

```gsql best_year_kpis
from flights
where extract(year from dep_time)::int = 2003
select on_time_arrival_rate
```

```gsql year_2005_kpis
from flights
where extract(year from dep_time)::int = 2005
select avg(dep_delay) as avg_dep_delay
```

<Row>
  <BigValue data=overall_kpis value=total_flights title="Total Flights (2000-2005)" fmt=num0 />
  <BigValue data=sep_2001_kpis value=cancellation_rate title="Sep 2001 Cancellation Rate" fmt=pct1 />
  <BigValue data=best_year_kpis value=on_time_arrival_rate title="2003 On-Time Rate (Best Year)" fmt=pct1 />
  <BigValue data=year_2005_kpis value=avg_dep_delay title="2005 Avg Dep Delay (min)" fmt="0.1" />
</Row>

```gsql annual_volume
from flights
select
  extract(year from dep_time)::int as year,
  count() as total_flights
order by year
```

```gsql annual_perf
from flights
select
  extract(year from dep_time)::int as year,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay
order by year
```

<Row>
  <BarChart title="Annual Flight Volume" data=annual_volume x=year y=total_flights />
  <BarChart title="Annual On-Time Arrival Rate" data=annual_perf x=year y=on_time_arrival_rate />
</Row>

```gsql monthly_trend
from flights
select
  date_trunc('month', dep_time) as month,
  round(cancellation_rate * 100, 2) as cancel_pct,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(on_time_arrival_rate * 100, 1) as on_time_pct
order by month
```

<Row>
  <ECharts data=monthly_trend height=320px>
    title: {text: 'Monthly Cancellation Rate (%) - Sep 2001 spike visible'},
    tooltip: {trigger: 'axis'},
    xAxis: {
      type: 'time',
      axisLine: {show: false},
      axisTick: {show: false},
    },
    yAxis: {
      type: 'value',
      name: 'Cancel %',
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    series: [{
      type: 'line',
      encode: {x: 'month', y: 'cancel_pct'},
      areaStyle: {opacity: 0.2},
      smooth: false,
      lineStyle: {color: '#C87F5A', width: 2},
      itemStyle: {color: '#C87F5A'},
      markPoint: {
        data: [{type: 'max', name: 'Peak (Sep 2001)'}],
        itemStyle: {color: '#B87470'},
      },
    }],
  </ECharts>

  <AreaChart
    title="Monthly Avg Departure Delay (min, excluding cancellations)"
    data=monthly_trend
    x=month
    y=avg_dep_delay
  />
</Row>

```gsql carrier_by_year
from flights
select
  extract(year from dep_time)::int as year,
  carriers.nickname as carrier,
  count() as flights,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay,
  cancellation_rate
order by year, flights desc
```

<Table
  data=carrier_by_year
  title="Carrier Performance by Year"
  rows=100
>
  <Column id=year />
  <Column id=carrier />
  <Column id=flights fmt=num0 />
  <Column id=on_time_arrival_rate title="On-Time Arr %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.1" contentType=colorscale colorScale=negative />
  <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
</Table>
