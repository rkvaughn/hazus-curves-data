# Worked examples

All queries run directly against the hosted Parquet — no download required. Substitute a
local path if you have cloned this repository.

```sql
INSTALL httpfs; LOAD httpfs;
SET VARIABLE base = 'https://rkvaughn.github.io/hazus-curves-data/';
```

For brevity the examples below write the URL out in full.

## Flood

### A single depth-damage curve

```sql
SELECT p.x AS depth_ft, p.y AS pct_damage
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet') p
WHERE  p.curve_id = 'fl-6.1-structure-129'
ORDER  BY p.x;
```

### Every structure curve for single-family homes

```sql
SELECT c.curve_id, c.source_agency, c.description
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
WHERE  c.peril = 'fl' AND c.occupancy = 'RES1'
  AND  c.damage_type = 'structure' AND c.hazus_version = '6.1';
```

### Which curve does Hazus pick by default?

`assignment_rules` records Hazus's own default selection per building combination.

```sql
SELECT occupancy, stories, basement, flood_zone, curve_id
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/assignment_rules.parquet')
WHERE  occupancy = 'RES1' AND damage_type = 'structure'
ORDER  BY flood_zone, stories;
```

### Curves valid in a given flood zone

`curve_zone_applicability` records which zones Hazus flags a curve as usable in. Note it
covers only the curves Hazus assigns — absence means *Hazus states no zone*, not *not
applicable*.

```sql
SELECT c.curve_id, c.occupancy, c.description
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
JOIN   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_zone_applicability.parquet') z
       USING (curve_id)
WHERE  z.flood_zone = 'CoastalV' AND c.damage_type = 'structure';
```

## Hurricane

### A fragility curve for one building type

```sql
SELECT p.x AS wind_mph, p.y AS prob_exceeding_severe
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
JOIN   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet') p USING (curve_id)
WHERE  c.peril = 'hu' AND c.building_type = 'WSF1'
  AND  c.damage_type = 'damage_severe'
  AND  c.curve_id LIKE '%-t1'          -- terrain 1 = Open
LIMIT  41;
```

### Filter by construction characteristics

Characteristics live in `curve_attributes` as key/value rows.

```sql
SELECT DISTINCT c.curve_id, c.building_type
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
WHERE  c.peril = 'hu' AND c.damage_type = 'damage_total'
  AND  EXISTS (SELECT 1 FROM read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_attributes.parquet') a
               WHERE a.curve_id = c.curve_id
                 AND a.key = 'Shutters' AND a.value = 'Yes')
  AND  EXISTS (SELECT 1 FROM read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_attributes.parquet') a
               WHERE a.curve_id = c.curve_id
                 AND a.key = 'Roof Shape' AND a.value = 'Hip');
```

### Curves applicable in a territory

Hazus geographic cases are **set-valued** — `ContUS+Hawaii` applies in both CONUS and
Hawaii. Matching the raw case with `=` silently drops most applicable curves, so join
`dim_geographic_case` and match on territory instead.

```sql
SELECT count(DISTINCT c.curve_id)
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
JOIN   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_attributes.parquet') a
       ON a.curve_id = c.curve_id AND a.key = 'geographic_case'
JOIN   read_parquet('https://rkvaughn.github.io/hazus-curves-data/dim_geographic_case.parquet') g
       ON g.case_name = a.value
WHERE  c.peril = 'hu' AND g.territory = 'Hawaii';
```

### Exclude curves with a known FEMA defect

```sql
SELECT c.curve_id, c.building_type, c.defect_verified
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
WHERE  c.peril = 'hu' AND c.defect_flag IS NULL;      -- no disclosed defect
```

## Always check units

```sql
SELECT k.peril, k.damage_type, k.x_units, k.y_units, k.notes
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_kind.parquet') k;
```

## Evaluating a curve between published points

Hazus curves are piecewise linear between published points. The companion Python package
implements this, and refuses to extrapolate beyond a curve's published domain:

```python
from hazus_curves import connect, damage
con = connect()
damage(con, "fl-6.1-structure-129", 3.5)   # % structure damage at 3.5 ft
```

Rolling your own: interpolate linearly between the two bracketing `x` values, and do not
read past the first or last point — a curve that stops at 24 ft says nothing about 30 ft.

## Python

```python
import duckdb
con = duckdb.connect()
con.execute("INSTALL httpfs; LOAD httpfs;")
df = con.execute("""
    SELECT c.occupancy, p.x AS depth_ft, p.y AS pct_damage
    FROM read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
    JOIN read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet') p USING (curve_id)
    WHERE c.curve_id = 'fl-6.1-structure-129'
    ORDER BY p.x
""").df()
```

## R

```r
library(arrow)
curves <- read_parquet("https://rkvaughn.github.io/hazus-curves-data/curves.parquet")
points <- read_parquet("https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet")
merge(curves[curves$curve_id == "fl-6.1-structure-129", ], points)
```
