---
title: Does Fleet Age Matter?
layout: dashboard
---

```gsql kpis
from flights
select
  count(distinct tail_num) as active_aircraft,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  on_time_arrival_rate,
  cancellation_rate
```

<Row>
  <BigValue data=kpis value=active_aircraft title="Aircraft in Service" />
  <BigValue data=kpis value=avg_fleet_age title="Avg Fleet Age (years)" />
  <BigValue data=kpis value=on_time_arrival_rate title="On-Time Arrival" />
  <BigValue data=kpis value=cancellation_rate title="Cancellation Rate" />
</Row>

```gsql age_buckets
from flights
select
  case
    when aircraft_age < 5 then '0-4 yrs'
    when aircraft_age < 10 then '5-9 yrs'
    when aircraft_age < 15 then '10-14 yrs'
    when aircraft_age < 20 then '15-19 yrs'
    when aircraft_age < 25 then '20-24 yrs'
    else '25+ yrs'
  end as age_group,
  case
    when aircraft_age < 5 then 0
    when aircraft_age < 10 then 5
    when aircraft_age < 15 then 10
    when aircraft_age < 20 then 15
    when aircraft_age < 25 then 20
    else 25
  end as age_sort,
  count() as flights,
  round(on_time_arrival_rate * 100, 1) as on_time_pct,
  round(cancellation_rate * 100, 2) as cancel_pct,
  round(avg(dep_delay), 1) as avg_dep_delay
order by age_sort
```

<ECharts data=age_buckets height=380px>
  title: {text: 'On-Time Rate & Flight Volume by Aircraft Age'},
  tooltip: {trigger: 'axis'},
  legend: {data: ['Flights', 'On-Time Rate (%)'], bottom: 0},
  grid: {top: 40, right: 80, bottom: 50},
  xAxis: {
    type: 'category',
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: [
    {
      type: 'value',
      name: 'On-Time %',
      min: 60,
      max: 90,
      axisLabel: {formatter: '{value}%'},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
      axisLine: {show: false},
    },
    {
      type: 'value',
      name: 'Flights',
      position: 'right',
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {show: false},
    },
  ],
  series: [
    {
      name: 'Flights',
      type: 'bar',
      yAxisIndex: 1,
      encode: {x: 'age_group', y: 'flights'},
      itemStyle: {color: '#d1e8f0', opacity: 0.9},
    },
    {
      name: 'On-Time Rate (%)',
      type: 'line',
      yAxisIndex: 0,
      encode: {x: 'age_group', y: 'on_time_pct'},
      lineStyle: {width: 2.5, color: '#3D6B7E'},
      itemStyle: {color: '#3D6B7E'},
      symbol: 'circle',
      symbolSize: 8,
    },
  ],
</ECharts>

```gsql carrier_fleet
from flights
select
  carriers.nickname as carrier,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  round(on_time_arrival_rate * 100, 1) as on_time_pct,
  round(cancellation_rate * 100, 2) as cancel_pct,
  on_time_arrival_rate,
  cancellation_rate,
  count() as flights
order by avg_fleet_age
```

<Row>
  <ECharts data=carrier_fleet height=420px>
    title: {text: 'Fleet Age vs On-Time Arrival Rate by Carrier'},
    tooltip: {trigger: 'item'},
    grid: {top: 40, right: 30, bottom: 50, left: 70},
    xAxis: {
      type: 'value',
      name: 'Avg Fleet Age (years)',
      nameLocation: 'middle',
      nameGap: 30,
      min: 0,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    yAxis: {
      type: 'value',
      name: 'On-Time Arrival %',
      nameLocation: 'middle',
      nameGap: 45,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
      axisLabel: {formatter: '{value}%'},
    },
    series: [{
      type: 'scatter',
      symbolSize: 16,
      encode: {x: 'avg_fleet_age', y: 'on_time_pct', itemName: 'carrier'},
      tooltip: {formatter: '{b}'},
      itemStyle: {color: '#3D6B7E', opacity: 0.8},
      label: {
        show: true,
        position: 'right',
        formatter: '{b}',
        fontSize: 11,
        color: '#555',
      },
    }],
  </ECharts>

  <Table data=carrier_fleet title="Carrier Fleet Summary" rows=20>
    <Column id=carrier />
    <Column id=avg_fleet_age title="Avg Age (yrs)" fmt="0.1" />
    <Column id=on_time_arrival_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
    <Column id=flights fmt=num0 />
  </Table>
</Row>

```gsql manufacturer_table
from flights
select
  aircraft.aircraft_models.manufacturer as manufacturer,
  count() as flights,
  round(avg(aircraft_age), 1) as avg_age,
  on_time_arrival_rate,
  cancellation_rate,
  round(avg(dep_delay), 1) as avg_dep_delay
having count() > 5000
order by flights desc
limit 10
```

<Table data=manufacturer_table title="Performance by Aircraft Manufacturer" rows=10>
  <Column id=manufacturer />
  <Column id=flights fmt=num0 />
  <Column id=avg_age title="Avg Age (yrs)" fmt="0.1" />
  <Column id=on_time_arrival_rate title="On-Time %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.1" />
</Table>
