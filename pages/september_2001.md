---
title: The day air travel stopped — and how it came back
layout: notebook
---

The FAA dataset in this project covers roughly 345,000 U.S. commercial flights from 2000 through 2005. Most analyses treat that span as a convenient window onto airline operations. But it's also a complete arc of one of the most disruptive events in the history of commercial aviation. The data captures the before, the moment, and the full five-year recovery.

## The moment it happened

```gsql sep_2001_daily
from flights
where extract(year from dep_time) = 2001 and extract(month from dep_time) = 9
select
  CAST(dep_time AS DATE) as flight_date,
  count() as scheduled,
  sum(case when cancelled = 'Y' then 1 else 0 end) as cancellations,
  count() - sum(case when cancelled = 'Y' then 1 else 0 end) as operated
order by flight_date
```

<ECharts data=sep_2001_daily height=300px>
  title: {text: 'Daily flights in September 2001'},
  tooltip: {trigger: 'axis'},
  legend: {data: ['Operated', 'Cancelled'], bottom: 0},
  grid: {bottom: 36},
  xAxis: {
    type: 'time',
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: {
    type: 'value',
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  series: [
    {
      name: 'Operated',
      type: 'bar',
      stack: 'total',
      encode: {x: 'flight_date', y: 'operated'},
      itemStyle: {color: '#5B8F9E'},
    },
    {
      name: 'Cancelled',
      type: 'bar',
      stack: 'total',
      encode: {x: 'flight_date', y: 'cancellations'},
      itemStyle: {color: '#C87F5A'},
    },
  ],
</ECharts>

On September 11, 2001, scheduled flights fell from around 150 per day to 47 — those were aircraft already airborne or on the ground before the FAA grounded all civilian traffic. September 12 had **31 scheduled flights and 31 cancellations**: a 100% cancellation rate, representing the complete grounding. By September 14–15, the FAA authorized limited resumption, and by September 16 daily operations were back to roughly 120 flights.

Three days. That's how long the most heavily trafficked airspace in the world was closed.

## The long recovery

```gsql monthly_volume
from flights
select
  date_trunc('month', dep_time) as mo,
  count() as total_flights,
  count() - sum(case when cancelled = 'Y' then 1 else 0 end) as operated_flights,
  sum(case when cancelled = 'Y' then 1 else 0 end) as total_cancellations
order by mo
```

<ECharts data=monthly_volume height=320px>
  title: {text: 'Monthly flight volume 2000–2005'},
  tooltip: {trigger: 'axis'},
  legend: {data: ['Operated', 'Cancelled'], bottom: 0},
  grid: {bottom: 36},
  xAxis: {
    type: 'time',
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: {
    type: 'value',
    name: 'Flights',
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  series: [
    {
      name: 'Operated',
      type: 'bar',
      stack: 'total',
      encode: {x: 'mo', y: 'operated_flights'},
      itemStyle: {color: '#5B8F9E'},
    },
    {
      name: 'Cancelled',
      type: 'bar',
      stack: 'total',
      encode: {x: 'mo', y: 'total_cancellations'},
      itemStyle: {color: '#C87F5A'},
    },
  ],
</ECharts>

```gsql yearly_summary
from flights
select
  extract(year from dep_time)::integer as yr,
  count() as total_flights,
  cancellation_rate,
  on_time_arrival_rate,
  avg(case when cancelled = 'N' then dep_delay end) as avg_dep_delay
order by yr
```

<Row>
  <BarChart data=yearly_summary x=yr y=total_flights title="Total Scheduled Flights by Year" />
  <LineChart data=yearly_summary x=yr y=on_time_arrival_rate title="On-Time Arrival Rate by Year" />
</Row>

Two things stand out. First, the recovery was real and sustained: total scheduled flights grew from around 47,000 in 2000 to over 71,000 in 2005 — a 52% increase. By 2003, annual volumes had already surpassed pre-9/11 levels.

Second, on-time arrival performance *improved* after 9/11 and stayed higher through 2002–2003. This is counterintuitive if you focus on the security theater narrative, but makes operational sense: airlines shed tens of thousands of flights in late 2001 and early 2002, reducing congestion at hub airports. Fewer aircraft competing for the same runways and gates meant less cascading delay. The degradation in 2004–2005 coincides with the volume surge; by the time the industry fully recovered its traffic, the delay machinery had come back with it.

## Which carriers survived — and who grew

```gsql carrier_trend
from flights
where cancelled = 'N'
select
  extract(year from dep_time)::integer as yr,
  carriers.nickname as carrier,
  count() as flights
having count() > 200
order by yr, flights desc
```

```gsql carrier_change
with
pre as (
  from flights where cancelled = 'N' and extract(year from dep_time) = 2000
  select carriers.nickname as carrier, count() as flights_2000
  having count() > 200
),
post as (
  from flights where cancelled = 'N' and extract(year from dep_time) = 2005
  select carriers.nickname as carrier, count() as flights_2005
  having count() > 200
)
from pre
left join post on pre.carrier = post.carrier
select
  pre.carrier as carrier,
  coalesce(flights_2000, 0) as flights_2000,
  coalesce(flights_2005, 0) as flights_2005,
  coalesce(flights_2005, 0) - coalesce(flights_2000, 0) as volume_change
order by volume_change desc
```

<Row>
  <Table data=carrier_change title="Carrier volume: 2000 vs 2005">
    <Column id=carrier />
    <Column id=flights_2000 title="Flights (2000)" fmt=num0 />
    <Column id=flights_2005 title="Flights (2005)" fmt=num0 />
    <Column id=volume_change title="Change" fmt=num0 contentType=colorscale colorScale=positive />
  </Table>
  <ECharts data=carrier_trend height=360px>
    title: {text: 'Operated flights per carrier by year'},
    tooltip: {trigger: 'item'},
    legend: {show: false},
    grid: {right: 80, left: 120},
    xAxis: {
      type: 'value',
      min: 2000,
      max: 2005,
      interval: 1,
      axisLine: {show: false},
      axisTick: {show: false},
      splitLine: {lineStyle: {color: '#f0f0f0'}},
    },
    yAxis: {
      type: 'category',
      inverse: false,
      axisLine: {show: false},
      axisTick: {show: false},
      encode: {y: 'carrier'},
    },
    series: [{
      type: 'scatter',
      symbolSize: 10,
      encode: {x: 'yr', y: 'carrier', value: 'flights'},
      itemStyle: {color: '#3D6B7E', opacity: 0.65},
    }],
  </ECharts>
</Row>

Southwest grew substantially over this period — the low-cost point-to-point model proved resilient precisely because it didn't depend on business travel filling hub connections. Legacy carriers like United and American either shrank or barely held steady; they entered this period already pressured by the post-dot-com recession and left it diminished by 9/11, fuel prices, and the emergence of low-cost competition.

## What the data can't show

The dataset records scheduled flights, operated flights, delays, and cancellations. It doesn't record load factors, ticket prices, or revenue. What we see here is the supply side of aviation's response to 9/11 — how many flights were flown, how reliably, and by whom.

But the operational data tells its own story clearly enough. Three days of complete grounding. A two-year dip in volume, followed by a sustained recovery that by 2005 had left total scheduled flights 52% above their 2000 baseline. And embedded in the middle of all of it, a paradox: the years immediately after the worst aviation security failure in U.S. history produced the best average on-time performance in the dataset.

The industry adapted. The data says so.
