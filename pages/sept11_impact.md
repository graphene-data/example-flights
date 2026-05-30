---
title: "How 9/11 Reshaped U.S. Air Travel"
layout: notebook
---

The FAA dataset spans 2000–2005, which means September 11, 2001 is in the data. This notebook uses that window to measure the immediate shock, the multi-year contraction, and the uneven carrier-level recovery — all from the raw schedule records.

---

## The Ground Stop in Numbers

The FAA ordered a nationwide ground stop at 9:42 AM on September 11. What that looks like in the data:

```sql daily_around_911
select
  date_trunc('day', dep_time) as day,
  count() as scheduled,
  sum(case when cancelled = 'Y' then 1 else 0 end) as cancelled_count,
  cancellation_rate
from flights
where dep_time between '2001-09-08' and '2001-09-20'
order by day
```

<Table
  data="daily_around_911"
  sortable
/>

On September 12, every single scheduled flight in the dataset was cancelled (`cancellation_rate = 1.0`). The system began to cautiously resume on the 14th — but at roughly half the pre-attack volume. Normal-ish operations didn't resume until around September 16–17.

The cancellation spike on September 11 itself (62%) is slightly understated: many flights that departed before the attacks are coded as completed.

---

## Monthly Volume: The Trough and the Climb Back

Looking at monthly totals across the full dataset makes the post-9/11 hollowing-out visible. Note that 2000 is the pre-attack baseline and 2002–2005 trace the recovery.

```sql monthly_volume
select
  date_trunc('month', dep_time) as month,
  extract(year from dep_time) as year,
  count() as flights
from flights
where dep_time between '2000-01-01' and '2005-12-31'
order by month
```

<AreaChart
  data="monthly_volume"
  x="month"
  y="flights"
  title="Monthly Scheduled Flights, 2000–2005"
/>

A few things worth calling out:

- **September 2001** drops sharply relative to August 2001 (4,607 → 3,616, a **22% month-over-month decline**).
- The industry didn't fully recover to its 2001 pre-attack peak until **mid-2003**, dragged further by the post-9/11 recession and airline bankruptcies.
- By 2004–2005, monthly totals were running **40–50% above** their 2001 levels — driven largely by low-cost carrier expansion.

---

## Carrier Resilience: Not Everyone Contracted the Same

The shock hit legacy carriers far harder than Southwest, which was already operating a point-to-point model with lower fixed costs.

```sql carrier_annual
select
  carrier,
  carriers.name as carrier_name,
  extract(year from dep_time) as year,
  count() as flights
from flights
where dep_time between '2000-01-01' and '2005-12-31'
  and carrier in ('WN', 'DL', 'AA', 'UA', 'US', 'NW', 'CO', 'AS', 'HP')
order by carrier_name, year
```

<LineChart
  data="carrier_annual"
  x="year"
  y="flights"
  splitBy="carrier_name"
  title="Annual Flights by Carrier, 2000–2005"
/>

The contrast is stark. Southwest (WN) never dipped — it grew through 2001 and 2002, even as the rest of the industry contracted. Delta and US Airways bore the worst of it: Delta shed nearly a third of its 2000 volume by 2002, and US Airways was in and out of bankruptcy by mid-decade.

---

## Did Punctuality Change?

One hypothesis: with fewer flights in the air, the remaining flights should run more on time — less congestion, more slack in the network. Let's check whether on-time departure rates shifted after 9/11.

```sql annual_ontime
select
  extract(year from dep_time) as year,
  on_time_departure_rate,
  on_time_arrival_rate,
  avg(dep_delay) as avg_dep_delay_min,
  avg(arr_delay) as avg_arr_delay_min
from flights
where dep_time between '2000-01-01' and '2005-12-31'
  and cancelled = 'N'
order by year
```

<Table
  data="annual_ontime"
  sortable
/>

<BarChart
  data="annual_ontime"
  x="year"
  y="on_time_departure_rate"
  title="On-Time Departure Rate by Year"
/>

2002 and 2003 do show a modest on-time improvement over 2000–2001 — consistent with the reduced load theory. But by 2004–2005, as volume rebounded, delays crept back up. The post-9/11 lull was a brief operational reprieve, not a structural improvement.

---

## Summary

| | What the data shows |
|---|---|
| **Immediate shock** | 100% cancellation rate on 9/12; system back at ~80% by 9/16 |
| **Monthly trough** | Sept 2001 was down 22% vs Aug 2001; recovery to pre-9/11 peak took ~18 months |
| **Who survived best** | Southwest, the only major carrier that grew through 2001–2002 |
| **Operational effect** | A 1–2 year window of marginally better on-time performance, reversed as volume returned |

The data tells the story you'd expect — but seeing the 9/12 cancellation rate at exactly `1.0` still lands with some weight.
