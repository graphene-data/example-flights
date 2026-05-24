---
title: "The Day the Skies Went Silent: U.S. Aviation Before and After 9/11"
layout: notebook
---

On the morning of September 11, 2001, four commercial airliners were hijacked and used as weapons. By 9:45 AM, the FAA issued a nationwide ground stop — the first total shutdown of U.S. civil airspace in history. What follows is a read of that event through the raw flight data: how abrupt the collapse was, which carriers absorbed the biggest hit, and what the recovery arc looked like over the next four years.

---

## The Shutdown, Day by Day

The FAA ordered all airborne aircraft to land immediately after the second tower was struck. Commercial aviation data captures this moment with unsettling precision.

```gsql daily_shutdown
from flights
where dep_time >= '2001-09-09' and dep_time < '2001-09-21'
select
  dep_time::date as day,
  count() as flights,
  sum(case when is_cancelled then 1 else 0 end) as cancelled,
  round(cancellation_rate * 100, 1) as cancel_pct
order by day
```

<Table data=daily_shutdown title="Daily Flights, September 9–20, 2001" sortable>
  <Column id=day title="Date" />
  <Column id=flights title="Scheduled" fmt=num0 />
  <Column id=cancelled title="Cancelled" fmt=num0 />
  <Column id=cancel_pct title="Cancel Rate (%)" />
</Table>

September 9 and 10 were normal flying days — roughly 150–160 flights in the sample, almost none cancelled. Then on September 11, only <Value data=daily_shutdown column=flights row=2 /> flights departed before the ground stop came down. September 12 recorded <Value data=daily_shutdown column=cancelled row=3 /> scheduled departures with <Value data=daily_shutdown column=cancelled row=3 /> cancellations: a complete shutdown. Service resumed on September 13 under emergency security protocols, but volumes were still less than half of normal through the end of that week.

---

## Monthly Context: The September Collapse

Zooming out to the monthly view, the dip is unmistakable against the otherwise steady 2000–2002 baseline.

```gsql monthly_volume_context
from flights
where dep_time >= '2000-01-01' and dep_time < '2003-01-01'
select
  date_trunc('month', dep_time) as month,
  count() as flights,
  round(cancellation_rate * 100, 2) as cancel_pct
order by month
```

<AreaChart
  data=monthly_volume_context
  x=month
  y=flights
  title="Monthly Flight Volume, 2000–2002"
  height=300px
/>

September 2001 was not just a missed summer peak — it broke the trend. October through December 2001 also fell well below the same months in 2000, as airlines slashed capacity and passengers stayed grounded. The cancellation rate in September 2001 was roughly **6×** the August baseline.

---

## Who Was Hit Hardest?

Two carriers had their planes directly weaponized — American Airlines (Flights 11 and 77) and United Airlines (Flights 175 and 93). But the data tells a more nuanced story about who absorbed the largest proportional hit.

```gsql carrier_impact
from flights
where dep_time >= '2001-08-01' and dep_time < '2001-10-01'
select
  carriers.nickname as carrier,
  sum(case when extract(month from dep_time) = 8 then 1 else 0 end) as aug_flights,
  sum(case when extract(month from dep_time) = 9 then 1 else 0 end) as sep_flights,
  round(
    100.0 *
    (sum(case when extract(month from dep_time) = 9 then 1 else 0 end) -
     sum(case when extract(month from dep_time) = 8 then 1 else 0 end)) /
    sum(case when extract(month from dep_time) = 8 then 1 else 0 end), 1
  ) as pct_change
order by pct_change asc
```

<Table data=carrier_impact title="August vs. September 2001 — Flights by Carrier" sortable>
  <Column id=carrier title="Carrier" />
  <Column id=aug_flights title="Aug Flights" fmt=num0 />
  <Column id=sep_flights title="Sep Flights" fmt=num0 />
  <Column id=pct_change title="MoM Change (%)" redNegatives />
</Table>

Alaska Airlines led the decline at -49.6%, likely because its long-haul Pacific Coast routes were among the first to see demand collapse — passengers on transcontinental legs had the longest replacement windows and the least urgency. American, despite being directly implicated in the attacks, came in second at -37.7%. Southwest, with its short-haul, high-frequency domestic network, was the most resilient at -11.8% — passengers couldn't easily substitute driving for a 45-minute hop.

A note on the numbers: September has fewer calendar days than August (30 vs. 31), so even the carriers that held up relatively well were flying fewer total flights. The per-operating-day decline was somewhat less severe than the raw month-over-month comparison suggests.

---

## The Recovery Arc

August 2001 became the industry's benchmark for what "normal" looked like. How long before monthly volumes surpassed it?

```gsql recovery_arc
from flights
where dep_time >= '2001-01-01' and dep_time < '2006-01-01'
select
  date_trunc('month', dep_time) as month,
  count() as flights
order by month
```

<LineChart
  data=recovery_arc
  x=month
  y=flights
  title="Monthly Flight Volume, 2001–2005"
  height=320px
/>

The industry did not simply bounce back. Monthly volumes stayed depressed through most of 2002. The real inflection came in 2003, driven largely by Southwest and low-cost regional carriers aggressively filling the void left by legacy airlines that had cut capacity. By 2004–2005, total flight counts had exceeded their pre-9/11 trajectory, as though the event were a temporary setback rather than a structural break.

---

## The Unexpected Upside: Delays Improved

A quieter sky is a more punctual one. With fewer aircraft competing for gates, runways, and ATC attention, did on-time performance improve?

```gsql delays_by_year
from flights
where not is_cancelled
select
  extract(year from dep_time)::integer as year,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(on_time_departure_rate * 100, 1) as pct_on_time_dep
order by year
```

<BarChart
  data=delays_by_year
  x=year
  y=avg_dep_delay
  title="Average Departure Delay by Year (minutes, non-cancelled flights)"
  height=300px
/>

Yes — dramatically so. Average departure delay fell from 11 minutes in 2000 to 5.5 minutes in 2002 and a nadir of 5.1 minutes in 2003. Then, as the industry roared back to life and volumes surged past pre-9/11 levels, delays followed: 7.8 minutes by 2004 and 9.1 minutes by 2005, almost exactly where they'd been before the attacks. The congestion that had plagued the system in 2000 was back with a vengeance by 2005.

The implication is uncomfortable: the brief window of excellent on-time performance wasn't the result of better processes, smarter scheduling, or improved infrastructure. It was the result of an emptier sky. As soon as demand returned, the old constraints reasserted themselves.

---

*Data: FAA domestic flight records, 2000–2005 (sample dataset). All flight counts reflect sampled records, not total U.S. industry volumes.*
