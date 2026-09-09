---
title: Kinds of Airports
---

# What kinds of airports are in the data?

The `airports` table is really an FAA registry of **all aviation facilities** — not just runways for planes. Every facility is classified by its `fac_type`. There are **7 kinds**, spanning nearly **19,800 facilities**.

```sql fac_types
select
  fac_type,
  count(*) as num_facilities,
  round(100.0 * count(*) / sum(count(*)) over (), 1) as pct
from airports
group by fac_type
order by num_facilities desc
```

<BarChart data="fac_types" y="fac_type" x="num_facilities" label={true} sort="num_facilities asc" height="360px" />

The registry is dominated by two categories: conventional **airports** (<Value data=fac_types column=pct row=0 />% of all facilities) and **heliports** (<Value data=fac_types column=pct row=1 />%). Everything else is a long tail of specialized landing sites.

## What each kind is

- **Airport** — <Value data=fac_types column=num_facilities row=0 /> facilities. Standard fixed-wing runway facilities, from small strips to major hubs.
- **Heliport** — <Value data=fac_types column=num_facilities row=1 /> facilities. Landing pads for helicopters (hospitals, corporate rooftops, etc.).
- **Seaplane base** — <Value data=fac_types column=num_facilities row=2 /> facilities. Water landing areas for float-equipped aircraft.
- **Ultralight** — <Value data=fac_types column=num_facilities row=3 /> facilities. Strips dedicated to lightweight, low-speed ultralight aircraft.
- **STOLport** — <Value data=fac_types column=num_facilities row=4 /> facilities. Short Take-Off and Landing fields for aircraft that need very little runway.
- **Gliderport** — <Value data=fac_types column=num_facilities row=5 /> facilities. Sites for launching and landing engineless gliders/sailplanes.
- **Balloonport** — <Value data=fac_types column=num_facilities row=6 /> facilities. The rarest type: launch/landing sites for hot-air balloons.
