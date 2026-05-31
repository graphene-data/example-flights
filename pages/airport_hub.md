---
title: Airport Performance Hub
layout: dashboard
---

```gsql popular_airports
from flights
select
  origin as code,
  origin_airport.city || ' – ' || origin as display_name,
  count() as total_flights
having count() > 10000
order by total_flights desc
```

<Dropdown name=airport data=popular_airports value=code label=display_name title="Airport" defaultValue="ATL" />

```gsql airport_kpis
from flights
where origin = $airport
select
  count() as total_departures,
  on_time_departure_rate,
  avg(dep_delay) as avg_dep_delay,
  cancellation_rate
```

<Row>
  <BigValue data=airport_kpis value=total_departures title="Total Departures" fmt=num0 />
  <BigValue data=airport_kpis value=on_time_departure_rate title="On-Time Departure %" />
  <BigValue data=airport_kpis value=avg_dep_delay title="Avg Departure Delay (min)" fmt="0.0" />
  <BigValue data=airport_kpis value=cancellation_rate title="Cancellation Rate" />
</Row>

```gsql monthly_volume
from flights
where origin = $airport
select date_trunc('month', dep_time) as month, count() as departures
order by month
```

```gsql top_routes
from flights
where origin = $airport and not is_cancelled
select
  destination || ' – ' || destination_airport.city as route,
  count() as flights,
  round(avg(dep_delay), 1) as avg_dep_delay
order by flights desc
limit 15
```

<Row>
  <LineChart title="Monthly Departure Volume" data=monthly_volume x=month y=departures />
  <BarChart title="Top Routes by Volume" data=top_routes x=route y=flights sort="flights desc" />
</Row>

```gsql hourly_delays
from flights
where origin = $airport and not is_cancelled and extract(hour from dep_time)::integer between 5 and 23
select
  extract(hour from dep_time)::integer as hour,
  round(avg(dep_delay), 1) as avg_dep_delay
order by hour
```

```gsql carrier_at_airport
from flights
where origin = $airport
select
  carriers.nickname as carrier,
  count() as flights,
  on_time_departure_rate,
  round(avg(dep_delay), 1) as avg_dep_delay,
  cancellation_rate
having count() > 50
order by flights desc
```

<Row>
  <BarChart title="Avg Departure Delay by Hour of Day" data=hourly_delays x=hour y=avg_dep_delay />
  <Table title="Carriers at This Airport" data=carrier_at_airport rows=20>
    <Column id=carrier />
    <Column id=flights fmt=num0 />
    <Column id=on_time_departure_rate title="On-Time Dep %" fmt=pct1 contentType=colorscale colorScale=positive />
    <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.0" />
    <Column id=cancellation_rate title="Cancel %" fmt=pct2 contentType=colorscale colorScale=negative />
  </Table>
</Row>
