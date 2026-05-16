---
title: The Short-Haul Paradox
layout: notebook
---

Ask a frequent flyer whether they prefer a quick hop or a longer direct flight, and most will immediately say: "whatever gets me there faster." But there's a nagging suspicion among road warriors that short-haul flights — the 45-minute connector, the shuttle between nearby cities — are somehow less reliable than their longer cousins. Cancellations feel more frequent. Delays feel harder to shake.

Using the FAA's dataset of over 345,000 commercial flights from 2000–2005, I looked at whether that suspicion has merit. The result is more nuanced than either the pessimists or optimists expect — and the answer depends entirely on which failure mode you're worried about.

## The departure delay surprise

```gsql distance_buckets
from flights
where cancelled = 'N'
select
  case
    when distance < 250 then '< 250 mi'
    when distance < 500 then '250–499 mi'
    when distance < 1000 then '500–999 mi'
    when distance < 1500 then '1,000–1,499 mi'
    else '1,500+ mi'
  end as distance_band,
  case
    when distance < 250 then 1
    when distance < 500 then 2
    when distance < 1000 then 3
    when distance < 1500 then 4
    else 5
  end as sort_order,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(on_time_arrival_rate * 100, 1) as on_time_pct,
  round(avg(taxi_out), 1) as avg_taxi_out
order by sort_order
```

<BarChart
  data=distance_buckets
  x=distance_band
  y=avg_dep_delay
  sort="sort_order asc"
  title="Avg departure delay by distance band (minutes)"
  height=280px
/>

Counterintuitively, the *shortest* flights have the lowest average departure delay in this dataset — about 6.6 minutes, compared to 8 minutes or more for medium-haul flights. If your mental model of short-haul misery centers on sitting at the gate while the minutes tick by, the data doesn't support it.

Why? Short-haul routes disproportionately serve smaller secondary airports — think Burbank instead of LAX, Midway instead of O'Hare — where runway queues are shorter, gate conflicts rarer, and ATC workload lighter. The regional carriers that dominate these routes also tend to operate simpler point-to-point networks that accumulate less cascading delay than hub-and-spoke systems.

## What happens after you push back

The departure picture is only half the story. The more revealing column is what happens during the flight itself.

```gsql buffer_vs_distance
from flights
where cancelled = 'N' and dep_delay is not null and arr_delay is not null
select
  case
    when distance < 250 then '< 250 mi'
    when distance < 500 then '250–499 mi'
    when distance < 1000 then '500–999 mi'
    when distance < 1500 then '1,000–1,499 mi'
    else '1,500+ mi'
  end as distance_band,
  case
    when distance < 250 then 1
    when distance < 500 then 2
    when distance < 1000 then 3
    when distance < 1500 then 4
    else 5
  end as sort_order,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(avg(arr_delay), 1) as avg_arr_delay,
  round(avg(dep_delay) - avg(arr_delay), 1) as delay_recovered,
  round(avg(taxi_out), 1) as avg_taxi_out,
  round(avg(flight_time), 0) as avg_flight_time_min
order by sort_order
```

<Table data=buffer_vs_distance sort="sort_order asc">
  <Column id=distance_band title="Distance Band" />
  <Column id=avg_flight_time_min title="Avg Flight Time (min)" fmt="0" />
  <Column id=avg_taxi_out title="Avg Taxi-Out (min)" fmt="0.0" />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <Column id=avg_arr_delay title="Avg Arr Delay (min)" fmt="0.0" />
  <Column id=delay_recovered title="Delay Recovered In-Air (min)" fmt="0.0" />
</Table>

The **Delay Recovered In-Air** column is where the short-haul disadvantage appears. Flights over 1,500 miles recover an average of 4.1 minutes of departure delay during the flight — pilots can throttle up, controllers can grant a more direct routing, and a four-hour cruise gives ample time to claw back a few minutes. A 39-minute hop recovers less than one minute. Whatever delay exists on departure, nearly all of it lands with you.

Notice that taxi-out time is essentially constant across all distance bands — about 13–18 minutes regardless of whether you're flying 150 miles or 2,000. For a sub-250-mile flight with a 39-minute average block time, taxi time is roughly a third of the total trip. There is simply no padding.

## The cancellation gap

Departure delays are one thing. But the sharper reliability story is cancellations.

```gsql cancel_by_distance
from flights
select
  case
    when distance < 250 then '< 250 mi'
    when distance < 500 then '250–499 mi'
    when distance < 1000 then '500–999 mi'
    when distance < 1500 then '1,000–1,499 mi'
    else '1,500+ mi'
  end as distance_band,
  case
    when distance < 250 then 1
    when distance < 500 then 2
    when distance < 1000 then 3
    when distance < 1500 then 4
    else 5
  end as sort_order,
  count() as total_flights,
  round(cancellation_rate * 100, 2) as cancel_pct
order by sort_order
```

<BarChart
  data=cancel_by_distance
  x=distance_band
  y=cancel_pct
  sort="sort_order asc"
  title="Cancellation rate by distance band (%)"
  height=260px
/>

The disparity here is stark. Sub-250-mile flights cancel at **1.47%** — nearly three times the rate of any other distance band, which all cluster around 0.25–0.54%. This is where the road warrior's intuition finally finds solid ground.

Several structural factors converge on the short end of the spectrum. Many of these routes are operated by regional affiliates under capacity-purchase agreements — United Express, Delta Connection, American Eagle — flying smaller turboprops and regional jets. These aircraft face lower weather minimums for operations: a crosswind or ceiling that a Boeing 737 handles routinely grounds a Bombardier Dash 8. Maintenance resources at the smaller airports they serve are thinner, so a mechanical squawk more often means a cancellation rather than a quick fix.

The economics also differ. Airlines weigh the cost of an overnight hotel bill against the cost of operating the flight. A cancelled transcontinental trip strands passengers far from home; the airline faces significant reaccommodation costs and reputational risk. A cancelled shuttle between two cities 200 miles apart offers the option of driving — and the airline knows it.

## The worst short-haul routes

Not all short-haul routes are equal. These are the highest-delay routes under 500 miles with at least 200 flights in the dataset:

```gsql worst_routes
from flights
where cancelled = 'N' and distance < 500
select
  origin || ' → ' || destination as route,
  origin,
  destination,
  origin_airport.city as from_city,
  destination_airport.city as to_city,
  distance,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay,
  round(on_time_arrival_rate * 100, 1) as on_time_pct,
having count() >= 200
order by avg_dep_delay desc
limit 15
```

<Table data=worst_routes title="Most Delayed Short-Haul Routes (< 500 miles, ≥ 200 flights)">
  <Column id=route title="Route" />
  <Column id=distance title="Distance (mi)" fmt=num0 />
  <Column id=flights title="Flights" fmt=num0 />
  <Column id=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=on_time_pct title="On-Time Arr %" fmt="0.0" contentType=colorscale colorScale=positive />
</Table>

Two patterns emerge. One set is regional spokes out of Atlanta — routes to Dothan (DHN), Montgomery (MGM), Panama City (PFN), Gainesville (GNV) — all feeding into Hartsfield, one of the world's busiest airports. These routes inherit delays directly from the ATL queue: a plane sitting at the gate in Dothan is waiting for an inbound connection that's been circling in ATL's hold stack. The other cluster is the northeast corridor — PHL to DCA, DTW to EWR, MDW routes — flights that are short by distance but touch some of the most congested airspace in the country. The takeaway: it's not just flight distance that matters, it's how tightly both endpoints are wired into congested hubs.

## The real paradox

The full picture resolves the intuition without vindicating it entirely.

Short-haul flights don't actually suffer from worse departure performance — in raw minutes, they leave closer to schedule than medium or long-haul routes. The frequent flyer who remembers a 45-minute hop as a departure nightmare is probably misremembering, or conflating two different problems.

Where short-haul genuinely falls short: **recovery and cancellation**. When a short flight does leave late, it arrives late — there is no airborne buffer to absorb the slip. And when conditions deteriorate, short-haul flights cancel at a dramatically higher rate, driven by smaller aircraft, thinner maintenance coverage, and an airline calculus that treats a 200-mile trip as substitutable.

The practical advice follows directly. If you must take a short-haul flight and reliability is the priority: fly in the morning when departure queues are short and delay hasn't accumulated, avoid routes that touch the northeast mega-airports, and if the weather looks marginal, have a contingency plan. Short flights that depart on time almost always arrive close to on time. The problem is when they don't depart at all.
