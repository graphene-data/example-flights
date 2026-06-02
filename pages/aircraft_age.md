---
title: "Age in the Air: Does Your Plane's Age Make You Late?"
layout: notebook
---

Ask a frequent flyer what they check before booking and many will mention looking up the aircraft type. Aviation enthusiasts have seat map apps and fleet trackers. Airlines advertise new deliveries in their marketing. The intuition that a newer plane means a more reliable flight is widespread — but is it actually in the FAA data?

This is an analysis of roughly 345,000 FAA-recorded commercial flights from 2000 to 2005, merged with the FAA aircraft registry, which includes the year each aircraft was built. For every flight, aircraft age is computed as the number of years between manufacture and the year of departure.

## The fleet is older than you might think

```gsql fleet_age_dist
from flights
where aircraft_age >= 0 and aircraft_age < 45
select
  floor(aircraft_age / 5)::integer * 5 as age_bucket,
  count() as flights
order by age_bucket
```

<BarChart
  data=fleet_age_dist
  x=age_bucket
  y=flights
  title="Flights operated by aircraft age group (5-year buckets)"
  height=280px
/>

The bulk of commercial operations in this dataset were flown on aircraft between 10 and 29 years old. Boeing and Airbus design their planes for 20–30-year service lives, and narrowbodies often fly longer. The long right tail is real: a meaningful slice of flights in this sample were on aircraft over 30 years old. Whatever folk wisdom says about booking new planes, most passengers don't have that option on most routes.

## Does old age mean more delays?

```gsql age_vs_delay
from flights
where not is_cancelled and aircraft_age >= 0 and aircraft_age < 40
select
  floor(aircraft_age)::integer as age_years,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(arr_delay), 1) as avg_arr_delay
order by age_years
```

<Row>
  <AreaChart
    data=age_vs_delay
    x=age_years
    y=avg_dep_delay
    title="Avg departure delay by aircraft age (years)"
  />
  <AreaChart
    data=age_vs_delay
    x=age_years
    y=avg_arr_delay
    title="Avg arrival delay by aircraft age (years)"
  />
</Row>

The charts are essentially flat. Across the entire 0–35 year age range, average departure delay hovers between 6 and 10 minutes with no systematic trend upward. Aircraft in their first year of service look slightly worse in parts of the range — new deliveries often go to fast-growing carriers already under schedule pressure. Very old aircraft (35+ years) show elevated delays, but sample sizes thin out rapidly past 30 years and the signal is noisy.

This is the first sign that aircraft age may not be the lever frequent flyers think it is.

## Cancellations tell a different story

```gsql age_vs_cancel
from flights
where aircraft_age >= 0 and aircraft_age < 40
select
  floor(aircraft_age)::integer as age_years,
  count() as total_flights,
  avg(case when cancelled = 'Y' then 100.0 else 0.0 end) as cancel_pct
order by age_years
```

<AreaChart
  data=age_vs_cancel
  x=age_years
  y=cancel_pct
  title="Cancellation rate (%) by aircraft age"
  height=280px
/>

Here the relationship is clearer. Aircraft in their first decade cancel roughly 1–1.5% of flights. By the time they reach 20–25 years old, that figure is closer to 2.5–3.5%. The mechanism is plausible: older airframes accumulate maintenance issues, AOG (aircraft on ground) events become more frequent, and spare-part availability worsens for retired model lines. A write-up that grounds a new plane for two hours can ground an old one for a day while a part gets sourced.

The effect is real but modest. Even at 30 years, most flights still operate. This is signal that would be invisible in your personal travel experience across any reasonable number of trips.

## The confound: are older fleets just worse airlines?

Before concluding that age independently causes cancellations, there's an obvious confound. Airlines that operate old fleets might simply be worse-run in general — or conversely, well-managed carriers might invest more in fleet renewal as part of the same discipline that also makes them operationally better. If that's true, the age effect is a proxy for carrier quality, not an independent causal factor.

```gsql carrier_age_profile
from flights
where not is_cancelled
select
  carriers.nickname as carrier,
  round(avg(aircraft_age), 1) as avg_fleet_age,
  round(avg(dep_delay), 1) as avg_dep_delay,
  count() as flights
order by avg_fleet_age desc
```

<ECharts data=carrier_age_profile height=460px>
  title: {text: 'Avg fleet age vs. avg departure delay by carrier'},
  tooltip: {trigger: 'item'},
  grid: {left: 55, right: 130, bottom: 55},
  xAxis: {
    type: 'value',
    name: 'Avg fleet age (years)',
    nameLocation: 'middle',
    nameGap: 35,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  yAxis: {
    type: 'value',
    name: 'Avg departure delay (min)',
    nameLocation: 'middle',
    nameGap: 40,
    axisLine: {show: false},
    axisTick: {show: false},
    splitLine: {lineStyle: {color: '#f0f0f0'}},
  },
  series: [{
    type: 'scatter',
    symbolSize: 10,
    encode: {x: 'avg_fleet_age', y: 'avg_dep_delay', itemName: 'carrier'},
    tooltip: {formatter: '{b}'},
    label: {
      show: true,
      position: 'right',
      formatter: '{b}',
      fontSize: 10,
      color: '#555',
    },
    itemStyle: {color: '#3D6B7E', opacity: 0.85},
  }]
</ECharts>

There is no discernible relationship between a carrier's average fleet age and its average departure delay. Young-fleet carriers and old-fleet carriers are scattered across the delay range without pattern. This strongly suggests that the departure delay signal in the earlier charts is driven by carrier-level effects — not by the planes themselves. The age-delay link is largely a statistical artifact of which airlines happen to fly which planes.

The cancellation finding is harder to dismiss, since it is more difficult to operationally compensate for a plane that is physically unserviceable. But even there, carrier maintenance culture and spare-part investment likely explain more variance than airframe age in isolation.

## Putting it together

```gsql summary_table
from flights
where aircraft_age >= 0 and aircraft_age < 35
select
  case
    when aircraft_age < 10 then 'Under 10 years'
    when aircraft_age < 20 then '10-19 years'
    when aircraft_age < 30 then '20-29 years'
    else '30+ years'
  end as age_group,
  case
    when aircraft_age < 10 then 0
    when aircraft_age < 20 then 1
    when aircraft_age < 30 then 2
    else 3
  end as sort_key,
  count() as total_flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  avg(case when cancelled = 'Y' then 100.0 else 0.0 end) as cancel_pct
order by sort_key
```

<Table data=summary_table title="Delays and cancellations by aircraft age band">
  <Column id=age_group title="Aircraft Age" />
  <Column id=total_flights title="Total Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=cancel_pct title="Cancellation (%)" fmt="0.00" contentType=colorscale colorScale=negative />
</Table>

The summary table makes the asymmetry visible. Average departure delay barely moves across age bands — the difference between the youngest and oldest group is under two minutes. Cancellation rate climbs roughly 1.5–2 percentage points from the youngest to the oldest bracket — a real effect, but not the dramatic reliability cliff the folklore suggests.

The practical takeaway inverts the usual advice. Aircraft age is nearly irrelevant to whether your flight will be **late**. It has a small but real relationship to whether your flight will be **cancelled**. If on-time departure is your priority, fly early in the morning — that matters far more, and was quantified in the [delay factors analysis](/pages/delay_factors). If cancellation risk is your concern, there is a modest statistical argument for preferring newer planes, though the signal is small enough that you would need a very large number of trips before you noticed it personally.
