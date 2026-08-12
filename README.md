# hazus-curves-data

Prebuilt data tables for **[hazus-curves](https://github.com/rkvaughn/hazus-curves)** — an open database of FEMA Hazus
damage and vulnerability curves.

This repository holds the data. It is designed to be usable on its own: the schema is
documented in [SCHEMA.md](SCHEMA.md), worked queries are in [EXAMPLES.md](EXAMPLES.md),
and where the numbers come from is in [PROVENANCE.md](PROVENANCE.md).

> Not affiliated with or endorsed by FEMA. "Hazus" is a trademark of the Federal
> Emergency Management Agency. See [Licence and reuse](#licence-and-reuse).

**Browse without downloading:** https://rkvaughn.github.io/hazus-curves/

---

## What is in here

| Peril | Hazus version | Curves |
|---|---|---|
| flood | 4.0 | 1,220 |
| flood | 6.1 | 1,826 |
| hurricane wind | 6.1 | 275,220 |

**278,266 curves, 11,372,354 data points** in total.

Flood curves are depth-damage functions: depth in feet relative to the first finished
floor (−4 to +24 ft, 29 points per curve), damage as a percentage of replacement value,
split into structure, contents and business inventory.

Hurricane curves run over 3-second peak gust wind speed (50–250 mph in 5 mph steps), for
6,116 wind building types × 5 terrain roughnesses × 9 loss classes.

## ⚠️ Read this before using the hurricane data

**The nine hurricane loss classes do not share units.** Four are damage-state exceedance
probabilities, two are loss ratios, one is measured in **days**, and two are in
**lbs/sq ft**. Averaging or plotting them together produces a meaningless number.

Join the `curve_kind` table to get the units for any curve rather than assuming them.
[SCHEMA.md](SCHEMA.md#units) explains this in full.

## Files

| File | Rows | Size | Note |
|---|---|---|---|
| `assignment_rules.parquet` | 2,040 | 0.0 MB |  |
| `curve_attributes.parquet` | 3,641,565 | 8.0 MB |  |
| `curve_attributes_hu.parquet` | 3,641,220 | 7.7 MB | hurricane-only slice of `curve_attributes` |
| `curve_kind.parquet` | 12 | 0.0 MB |  |
| `curve_points.parquet` | 11,372,354 | 44.7 MB |  |
| `curve_points_hu.parquet` | 11,284,020 | 51.1 MB | hurricane-only slice of `curve_points` |
| `curve_zone_applicability.parquet` | 494 | 0.0 MB |  |
| `curves.parquet` | 278,266 | 0.7 MB |  |
| `curves_hu.parquet` | 275,220 | 0.3 MB | hurricane-only slice of `curves` |
| `dim_building_type.parquet` | 39 | 0.0 MB |  |
| `dim_geographic_case.parquet` | 6 | 0.0 MB |  |
| `dim_occupancy.parquet` | 33 | 0.0 MB |  |
| `provenance.parquet` | 19 | 0.0 MB |  |

The three `_hu` files are hurricane-only slices of the combined tables, kept because the
web browser reads them directly. If you are working with this data yourself, prefer the
combined tables (`curves`, `curve_points`, `curve_attributes`) and filter on
`peril = 'hu'` — they contain the same rows plus flood.

## Quickstart

These files are Parquet, readable by most data tools without a download step.

```sql
-- DuckDB, straight over HTTP
INSTALL httpfs; LOAD httpfs;
SELECT c.curve_id, c.occupancy, p.x AS depth_ft, p.y AS pct_damage
FROM   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curves.parquet') c
JOIN   read_parquet('https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet') p USING (curve_id)
WHERE  c.occupancy = 'RES1' AND c.damage_type = 'structure'
  AND  c.hazus_version = '6.1'
ORDER  BY c.curve_id, p.x;
```

```python
import pandas as pd
curves = pd.read_parquet("https://rkvaughn.github.io/hazus-curves-data/curves.parquet")
points = pd.read_parquet("https://rkvaughn.github.io/hazus-curves-data/curve_points.parquet")
```

Prefer a local database, or another SQL engine? The companion Python package builds
SQLite, DuckDB, PostgreSQL or Snowflake from these same files in one command:

```bash
pip install "git+https://github.com/rkvaughn/hazus-curves@v0.1.0"
hazus-curves install --perils fl,hu
```

More worked examples, including the hurricane joins, are in [EXAMPLES.md](EXAMPLES.md).

## Data quality flags you should not ignore

Two columns on `curves` carry warnings that travel with the data rather than living in a
footnote:

- **`defect_flag`** — non-null when FEMA has publicly disclosed a defect affecting this
  curve. Null means *no known defect*, not *verified correct*.
- **`defect_verified`** — our own measurement, kept separate from FEMA's disclosure.
  `identical_to_1_story` means the curve was measured to be a byte-for-byte duplicate of
  its 1-story counterpart.

34,560 curves carry a defect flag and
15,200 are confirmed duplicates. They are published rather than
withheld: a flagged curve is more useful than a missing one, and the corrected data is
not publicly extractable. See [PROVENANCE.md](PROVENANCE.md#known-defects).

## How this data was made

No damage value in these files was computed, interpolated, smoothed, clamped or inferred.
The pipeline renames columns and reshapes wide source tables into long ones; the numbers
pass through untouched, and a missing source value is preserved as `NULL` rather than
estimated.

Every row carries `source_file`, `source_table` and `source_row_id` identifying the
originating cell, and every input file is pinned by SHA-256. See
[PROVENANCE.md](PROVENANCE.md).

A 23-page report verifying this data against FEMA's own published manuals — with source
pages reproduced as screenshots — is in the main repository at
[`docs/Hazus_Data_Verification_Report.docx`](https://github.com/rkvaughn/hazus-curves/blob/main/docs/Hazus_Data_Verification_Report.docx).

## Regenerating

Nothing here is authored by hand. To rebuild from FEMA sources:

```bash
git clone https://github.com/rkvaughn/hazus-curves.git
cd hazus-curves
python -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/python scripts/build_all.py --perils fl,hu
cp dist/*.parquet /path/to/hazus-curves-data/
.venv/bin/python scripts/build_data_repo_docs.py --out /path/to/hazus-curves-data
```

## Licence and reuse

The curves are, to the best of our understanding, works of the United States Government
and therefore not subject to domestic copyright under 17 U.S.C. § 105. We assert no
additional copyright over them and place no restrictions on their reuse.

Please cite FEMA and the originating agencies, and state which Hazus version you used.
The open questions on this — and there are some — are set out in full in
[LICENSE-DATA.md](https://github.com/rkvaughn/hazus-curves/blob/main/LICENSE-DATA.md).

**Do not use these curves for life-safety decisions without independent verification
against the current official Hazus release.**
