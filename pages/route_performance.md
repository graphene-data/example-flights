---
title: Route Performance
layout: dashboard
---

```gsql major_origins
from flights
select origin as code, origin_airport.city as city, origin_airport.state as state
having count() > 5000
order by city
```

<Dropdown name=origin data=major_origins value=code label=city title="Origin Airport" defaultValue="ORD" />

```gsql origin_kpis
from flights where origin = $origin
select
  count() as total_flights,
  on_time_departure_rate,
  cancellation_rate,
  avg(dep_delay) as avg_dep_delay,
  count(distinct destination) as destinations
```

<Row>
  <BigValue data=origin_kpis value=total_flights title="Total Flights" fmt=num0 />
  <BigValue data=origin_kpis value=on_time_departure_rate title="On-Time Departure" />
  <BigValue data=origin_kpis value=cancellation_rate title="Cancellation Rate" />
  <BigValue data=origin_kpis value=avg_dep_delay title="Avg Dep Delay (min)" fmt="0.0" />
  <BigValue data=origin_kpis value=destinations title="Destinations Served" />
</Row>

```gsql monthly_trend
from flights where origin = $origin and cancelled = 'N'
select date_trunc('month', dep_time) as month, avg(dep_delay) as avg_delay
order by month
```

```gsql top_dest
from flights where origin = $origin and cancelled = 'N'
select destination_airport.city as city, count() as flights
order by flights desc
limit 15
```

<Row>
  <AreaChart data=monthly_trend x=month y=avg_delay title="Avg Departure Delay by Month (min)" />
  <BarChart data=top_dest x=city y=flights title="Top 15 Destinations by Volume" />
</Row>

```gsql routes
from flights where origin = $origin and cancelled = 'N'
select
  destination as code,
  destination_airport.city as city,
  destination_airport.state as state,
  count() as flights,
  on_time_departure_rate,
  avg(dep_delay) as avg_dep_delay,
  round(avg(distance), 0) as avg_distance_mi
order by flights desc
```

<Table data=routes title="All Routes from This Airport" rows=100>
  <Column id=city title="Destination City" />
  <Column id=state title="State" />
  <Column id=code title="IATA" />
  <Column id=flights title="Flights" fmt=num0 />
  <Column id=on_time_departure_rate title="On-Time Dep %" fmt=pct1 contentType=colorscale colorScale=positive />
  <Column id=avg_dep_delay title="Avg Delay (min)" fmt="0.0" contentType=colorscale colorScale=negative />
  <Column id=avg_distance_mi title="Avg Distance (mi)" fmt=num0 />
</Table>
