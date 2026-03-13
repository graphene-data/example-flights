This is a Graphene project to analyze FAA flight data from 2000-2005.

Assume all DuckDB functions are available when writing GSQL.
This project uses npm, so run all Graphene CLIs using `npm exec`.

Available tables:
- **tables/flights.gsql** — core fact table; one row per flight. Columns: `id2`, `carrier`, `origin`, `destination`, `flight_num`, `flight_time`, `tail_num`, `dep_time`, `arr_time`, `dep_delay`, `arr_delay`, `taxi_out`, `taxi_in`, `distance`, `cancelled`, `diverted`. Dimensions: `is_long_haul`, `is_cancelled_or_diverted`, `miles_flown`. Measures: `on_time_departure_rate`, `on_time_arrival_rate`, `cancellation_rate`, `diversion_rate`. Joins: `carriers`, `airports` (as `origin_airport` and `destination_airport`), `aircraft`.
- **tables/carriers.gsql** — airline lookup. Columns: `code`, `name`, `nickname`. Joins: `flights`.
- **tables/airports.gsql** — airport lookup. Columns: `id`, `code`, `fac_type`, `fac_use`, `faa_region`, `city`, `county`, `state`, `full_name`, `longitude`, `latitude`, `elevation`, `cbd_dist`, `cbd_dir`, `joint_use`, `mil_rts`, `cntl_twr`, `major`, and more. Joins: `flights` (as `departures` and `arrivals`).
- **tables/aircraft.gsql** — individual aircraft (by tail number). Columns: `id`, `tail_num`, `aircraft_model_code`, `year_built`, `name` (owner), `city`, `state`, `status_code`, and more. Dimensions: `age` (years old as of 2005). Measures: `avg_age`. Joins: `aircraft_models`, `flights`.
- **tables/aircraft_models.gsql** — aircraft model specs. Columns: `aircraft_model_code`, `manufacturer`, `model`, `engines`, `seats`, `weight`, `speed`, and more. Joins: `aircraft`.
