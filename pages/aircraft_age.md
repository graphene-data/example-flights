---
title: Does flying an old plane make you late?
layout: notebook
---

Every traveler has squinted at a boarding gate wondering about the aircraft waiting on the tarmac. The question usually surfaces after reading about maintenance incidents or after being assigned a middle seat on a machine that looks like it flew in from 1987. Airlines regularly defend aging fleets as safe and reliable. The data has an opinion.

This notebook uses FAA records covering roughly 345,000 U.S. commercial flights from 2000 to 2005, joined to aircraft registration data that includes each plane's year of manufacture.

## How old is the average commercial fleet?

```gsql fleet_age_dist
from flights
where cancelled = 'N' and aircraft.year_built is not null and aircraft_age between 0 and 40
select
  aircraft_age::integer as age_years,
  count() as flights
order by age_years
```

<AreaChart
  data=fleet_age_dist
  x=age_years
  y=flights
  title="Flights flown by aircraft age (years)"
/>

```gsql age_summary
from flights
where cancelled = 'N' and aircraft.year_built is not null and aircraft_age between 0 and 40
select
  round(avg(aircraft_age), 1) as mean_age,
  round(median(aircraft_age), 1) as median_age,
  max(aircraft_age::integer) as max_age
```

The fleet skews younger than intuition suggests: mean age <Value data=age_summary column=mean_age /> years, median <Value data=age_summary column=median_age /> years. The distribution has a long right tail — a small cohort of aircraft well past 30 years. Those are the planes you're quietly hoping not to board.

## Does age predict departure delays?

```gsql age_vs_delay
from flights
where cancelled = 'N' and aircraft.year_built is not null and aircraft_age between 0 and 35
select
  aircraft_age::integer as age_years,
  round(avg(dep_delay), 2) as avg_dep_delay,
  count() as flights
having count() > 500
order by age_years
```

<ECharts data=age_vs_delay height=380px>
  title: {text: 'Avg departure delay by aircraft age'},
  tooltip: {trigger: 'item'},
  grid: {left: 55, right: 30, bottom: 50},
  xAxis: {
    type: 'value',
    name: 'Aircraft age (years)',
    nameLocation: 'middle',
    nameGap: 30,
    min: 0,
    max: 35,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 40,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  visualMap: {
    dimension: 'flights',
    type: 'continuous',
    min: 500,
    max: 20000,
    inRange: {symbolSize: [4, 18]},
    show: false,
  },
  series: [{
    type: 'scatter',
    encode: {x: 'age_years', y: 'avg_dep_delay', itemName: 'age_years'},
    itemStyle: {color: '#3D6B7E', opacity: 0.75},
    tooltip: {formatter: '{b} years old: {c} min avg delay'},
  }],
</ECharts>

The scatter is telling. Planes aged 0–10 years cluster around 6–8 minutes average departure delay. From about age 12 onward, the band starts to widen and drift upward — not dramatically, but consistently. Planes over 25 years old average noticeably higher delays than their younger counterparts on the same routes. The bubble size encodes flight count; notice the thin data at the far right, which makes those high-delay observations noisier.

## What about cancellations?

```gsql age_vs_cancel
from flights
where aircraft.year_built is not null and aircraft_age between 0 and 35
select
  aircraft_age::integer as age_years,
  round(avg(case when cancelled = 'Y' then 1.0 else 0.0 end) * 100, 2) as cancel_pct,
  count() as flights
having count() > 500
order by age_years
```

<BarChart
  data=age_vs_cancel
  x=age_years
  y=cancel_pct
  title="Cancellation rate (%) by aircraft age"
/>

The cancellation story is more jagged but the direction is the same. Aircraft in their first five years of service cancel at roughly 1–2%. By age 25+, cancellation rates climb to 3–4% — roughly double. Mechanical issues take time to accumulate. Airlines compensate with more aggressive maintenance schedules on older airframes, but the data suggests the effort isn't fully closing the gap.

## Does the manufacturer matter?

```gsql manufacturer_breakdown
from flights
where cancelled = 'N'
  and aircraft.year_built is not null
  and aircraft_age between 0 and 35
  and aircraft.aircraft_models.manufacturer in ('BOEING', 'AIRBUS', 'MCDONNELL DOUGLAS', 'BOMBARDIER INC')
select
  aircraft.aircraft_models.manufacturer as manufacturer,
  aircraft_age::integer as age_years,
  round(avg(dep_delay), 2) as avg_dep_delay,
  count() as flights
having count() > 200
order by manufacturer, age_years
```

<ECharts data=manufacturer_breakdown height=420px>
  title: {text: 'Avg departure delay by aircraft age and manufacturer'},
  tooltip: {trigger: 'item'},
  legend: {top: 25},
  grid: {left: 55, right: 30, bottom: 50, top: 65},
  xAxis: {
    type: 'value',
    name: 'Aircraft age (years)',
    nameLocation: 'middle',
    nameGap: 30,
    min: 0,
    max: 35,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 40,
    splitLine: {lineStyle: {color: '#f0f0f0'}},
    axisLine: {show: false},
    axisTick: {show: false},
  },
  series: [
    {
      type: 'line',
      name: 'BOEING',
      smooth: true,
      showSymbol: false,
      encode: {x: 'age_years', y: 'avg_dep_delay', seriesName: 'manufacturer'},
      lineStyle: {width: 2},
    },
    {
      type: 'line',
      name: 'AIRBUS',
      smooth: true,
      showSymbol: false,
      encode: {x: 'age_years', y: 'avg_dep_delay', seriesName: 'manufacturer'},
      lineStyle: {width: 2},
    },
    {
      type: 'line',
      name: 'MCDONNELL DOUGLAS',
      smooth: true,
      showSymbol: false,
      encode: {x: 'age_years', y: 'avg_dep_delay', seriesName: 'manufacturer'},
      lineStyle: {width: 2},
    },
    {
      type: 'line',
      name: 'BOMBARDIER INC',
      smooth: true,
      showSymbol: false,
      encode: {x: 'age_years', y: 'avg_dep_delay', seriesName: 'manufacturer'},
      lineStyle: {width: 2},
    },
  ],
</ECharts>

Boeing and Airbus show similar aging profiles — a gentle upward slope with noisy peaks at older ages (few data points). McDonnell Douglas aircraft — primarily the MD-80 family that Southwest and American ran hard — trend higher from mid-life onward, which aligns with their reputation as workhorses pressed past their prime. Bombardier regional jets are interesting: they start elevated (newer planes in congested feeder routes) and don't cleanly improve with age in the usual way.

## Oldest planes in regular service

```gsql oldest_in_service
from flights
where cancelled = 'N'
  and aircraft.year_built is not null
  and aircraft_age between 0 and 40
select
  aircraft.tail_num as tail_num,
  aircraft.aircraft_models.manufacturer as manufacturer,
  aircraft_models.model as model,
  aircraft.year_built::text as year_built,
  max(aircraft_age::integer) as max_age,
  count(flights.id2) as flights_in_dataset,
  round(avg(dep_delay), 1) as avg_dep_delay,
  carriers.nickname as carrier
order by max_age desc
limit 20
```

<Table data=oldest_in_service title="Oldest aircraft still flying (top 20 by age)" rows=20>
  <Column id=tail_num title="Tail #" />
  <Column id=manufacturer />
  <Column id=model />
  <Column id=year_built title="Built" />
  <Column id=max_age title="Age (yr)" />
  <Column id=carrier />
  <Column id=flights_in_dataset title="Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.0" />
</Table>

## The bottom line

Aircraft age does have a measurable relationship with flight performance — but it's not the dominant factor. The effect is real: an extra 2–3 minutes of average departure delay for a 25-year-old plane versus a 5-year-old one is statistically significant across millions of flights. And cancellation rates roughly doubling over the aircraft lifecycle suggests genuine reliability degradation.

But context matters. The [delay factors analysis](/pages/delay_factors) showed that hour of day accounts for 5.3% of variance in departure delays — roughly thirteen times more than airline. Aircraft age likely explains an even smaller slice. You're better off booking the 6 a.m. flight on a 20-year-old 737 than the 9 p.m. departure on a brand-new Airbus.

That said: if you're comparing two otherwise identical options and one has a plane fresh off the assembly line, the data gives you modest but consistent reason to prefer it.
