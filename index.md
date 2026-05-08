---
title: Home
layout: dashboard
---

This project is for operations and analysis of the FAA's commercial flight data from **2000–2005**.

## The data at a glance

```gsql totals
from flights select
  count() as flights,
  count(distinct carrier) as carriers,
  count(distinct origin) as airports,
  count(distinct tail_num) as aircraft
```

<Row>
  <BigValue data=totals value=flights title="Flights" fmt=num0 />
  <BigValue data=totals value=carriers title="Carriers" />
  <BigValue data=totals value=airports title="Airports" />
  <BigValue data=totals value=aircraft title="Aircraft" fmt=num0 />
</Row>

```gsql monthly_volume
from flights where not is_cancelled
select date_trunc('month', dep_time) as month, count() as flights
order by month
```

<AreaChart
  data=monthly_volume
  x=month
  y=flights
  title="Monthly flight volume"
  height=240px
/>

## Pages in this project

- **[Operations Overview](/pages/operations_overview)** — top-line KPIs, monthly volume, a delay heatmap by hour and day-of-week, and a ranked table of carriers. The default landing dashboard if you just want the numbers.
- **[Carrier Detail](/pages/carrier_detail)** — pick any airline from the dropdown to see its rank, fleet, delay distribution, and 2005 monthly trend against the rest of the industry.
- **[Delay Factors](/pages/delay_factors)** — a deeper analysis of what actually predicts a late departure: hour of day, airline, origin airport, day of week. Spoiler: one factor dominates the others.

## The data model

Five tables (in `/tables`), joined into a star around `flights`:

| Table | Grain | Joins to | Notable fields |
| --- | --- | --- | --- |
| `flights` | one scheduled flight | `carriers`, `airports` (×2), `aircraft` | `dep_delay`, `arr_delay`, `cancelled`, `on_time_arrival_rate`, `cancellation_rate`, `aircraft_age` |
| `carriers` | one airline | `flights` | `code`, `name`, `nickname` |
| `airports` | one FAA facility | `flights` (as origin and destination) | `code`, `city`, `state`, `latitude`, `longitude`, `major` |
| `aircraft` | one registered tail number | `flights`, `aircraft_models` | `tail_num`, `year_built`, owner info |
| `aircraft_models` | one aircraft model | `aircraft` | `manufacturer`, `model`, `seats`, `engines`, `speed` |
