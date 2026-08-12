# Schema reference

Generated from the schema definition and measured against the Parquet files, so it cannot drift from the data.

## Design in one minute

Three layers:

1. **`curves`** — one row per curve, carrying identity and provenance.
2. **`curve_points`** — the curve itself, in long format: one row per `(curve_id, x)`. No measurements are encoded in column names.
3. **`curve_attributes`** — a key/value table holding the dimensions that differ by peril (terrain, shutters, roof deck attachment, flood zone, …).

The third layer is what lets one schema hold perils that are keyed completely differently. Flood curves are keyed by occupancy, zone, stories and basement; hurricane curves by building type, terrain and around a dozen construction characteristics. Rather than widening `curves` with columns that are null for most rows, peril-specific keys live as rows in `curve_attributes`. Adding earthquake later adds rows, not columns.

## Units

**`x` and `y` mean different things depending on the curve.** Join `curve_kind` on `(peril, damage_type)` rather than assuming.

| peril | damage_type | x | x units | y | y units |
|---|---|---|---|---|---|
| fl | contents | depth | `ft_above_first_floor` | damage | `percent` |
| fl | inventory | depth | `ft_above_first_floor` | damage | `percent` |
| fl | structure | depth | `ft_above_first_floor` | damage | `percent` |
| hu | building_loss | wind_speed | `mph_3s_gust` | loss | `loss_ratio_0_1` |
| hu | content_loss | wind_speed | `mph_3s_gust` | loss | `loss_ratio_0_1` |
| hu | damage_moderate | wind_speed | `mph_3s_gust` | exceedance_probability | `probability_0_1` |
| hu | damage_severe | wind_speed | `mph_3s_gust` | exceedance_probability | `probability_0_1` |
| hu | damage_slight | wind_speed | `mph_3s_gust` | exceedance_probability | `probability_0_1` |
| hu | damage_total | wind_speed | `mph_3s_gust` | exceedance_probability | `probability_0_1` |
| hu | debris_brick_wood | wind_speed | `mph_3s_gust` | debris | `lbs_per_sqft` |
| hu | debris_concrete_steel | wind_speed | `mph_3s_gust` | debris | `lbs_per_sqft` |
| hu | loss_of_use | wind_speed | `mph_3s_gust` | loss_of_use | `days` |

Note in particular that hurricane `loss_of_use` is measured in **days** and the two debris classes in **lbs/sq ft**. They are not ratios and must not be mixed with the loss curves.

## curve_id format

```
flood:      fl-<hazus_version>-<damage_type>-<source_row_id>
            e.g. fl-6.1-structure-129
hurricane:  hu-<hazus_version>-<damage_type>-<wbID>-t<terrain_id>
            e.g. hu-6.1-damage_severe-1-t1
```

The trailing identifier is the native Hazus primary key, preserved verbatim so any curve can be traced back to its source row.

## Tables

### `curves`

One row per damage/vulnerability curve.

**278,266 rows** · primary key `(curve_id)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `curve_id` | VARCHAR | 278,266 | no nulls | `fl-4.0-structure-110`, `fl-4.0-structure-111` | Stable synthetic key: peril-version-type-id. |
| `peril` | VARCHAR | 2 | no nulls | `fl`, `hu` | fl = flood, hu = hurricane wind. |
| `hazus_version` | VARCHAR | 2 | no nulls | `6.1`, `4.0` | Hazus release the content represents. |
| `damage_type` | VARCHAR | 12 | no nulls | `inventory`, `damage_slight` | structure \| contents \| inventory \| downtime. |
| `occupancy` | VARCHAR | 33 | 275,220 null (99%) | `COM4`, `COM9` | Hazus occupancy class (RES1, COM1, ...). Null where not applicable. |
| `building_type` | VARCHAR | 39 | 3,046 null (1%) | `WMUH2`, `WMUH3` | Hazus specific building type. Hurricane; null for flood. |
| `source_agency` | VARCHAR | 23 | no nulls | `USACE Generic`, `FIA` | Originating agency, verbatim from Hazus (e.g. 'USACE - Galveston'). |
| `description` | VARCHAR | 3,987 | no nulls | `two floors, w/ basement, Structure, A-`, `three or more floors, no basement, Str` | Hazus description, verbatim. |
| `defect_flag` | VARCHAR | 1 | 243,706 null (88%) | `FEMA Hazus 7.1 release notes (RSA-2118` | Non-null when FEMA has disclosed a defect in this curve. See docs/provenance.md. Null means no known defect, NOT 'verified correct'. |
| `defect_verified` | VARCHAR | 2 | 243,706 null (88%) | `differs_from_1_story`, `identical_to_1_story` | Our own measurement, kept separate from FEMA's disclosure above. 'identical_to_1_story' means this curve was measured to be a byte duplicate of its 1-story counterpart; 'differs_from_1_story' means it was not. See docs/hurricane_defect.md. 'differs' does NOT mean correct. |
| `source_file` | VARCHAR | 5 | no nulls | `HazusFloodDamageFunctions_Hazus61.xlsx`, `HazusWindDamFunctions_Hazus61.xlsx` | File in raw/ this row came from. |
| `source_table` | VARCHAR | 6 | no nulls | `flBldgStructDmgFn`, `flBldgContDmgFunc` | Sheet or table within that file. |
| `source_row_id` | VARCHAR | 276,219 | no nulls | `135`, `190` | Native Hazus primary key, verbatim. |

### `curve_points`

Tidy long-format curve points. One row per (curve, x).

**11,372,354 rows** · primary key `(curve_id, x)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `curve_id` | VARCHAR | 278,266 | no nulls | `hu-6.1-damage_severe-2513-t3`, `hu-6.1-damage_severe-2514-t3` |  |
| `x` | DOUBLE | 70 | no nulls | `160.0`, `75.0` | flood: depth in feet relative to first finished floor (-4..24). hurricane: 3-second peak gust in mph. Units in curve_kind. |
| `y` | DOUBLE | 2,690,988 | 32 null (0%) | `0.99792`, `0.82239` | Damage as percent of replacement value. NULL means the source was empty -- gaps are preserved, never interpolated or filled. |

### `curve_attributes`

Peril-specific keying dimensions. This is what makes the unified schema lossless and peril-pluggable.

**3,641,565 rows** · primary key `(curve_id, key)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `curve_id` | VARCHAR | 275,548 | no nulls | `hu-6.1-debris_concrete_steel-5294-t1`, `hu-6.1-debris_concrete_steel-5295-t3` |  |
| `key` | VARCHAR | 33 | no nulls | `Roof-Wall Connection`, `Secondary Water Resistance` | Attribute name, e.g. flood_zone, basement, terrain. |
| `value` | VARCHAR | 8,662 | no nulls | `ContUS+Hawaii`, `BUR` | Attribute value as text, verbatim from source. |

### `curve_kind`

What x and y actually mean, per peril. Consumers should read this rather than assume units.

**12 rows** · primary key `(peril, damage_type)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `peril` | VARCHAR | 2 | no nulls | `fl`, `hu` |  |
| `damage_type` | VARCHAR | 12 | no nulls | `damage_moderate`, `inventory` |  |
| `x_name` | VARCHAR | 2 | no nulls | `wind_speed`, `depth` |  |
| `x_units` | VARCHAR | 2 | no nulls | `ft_above_first_floor`, `mph_3s_gust` |  |
| `y_name` | VARCHAR | 5 | no nulls | `loss_of_use`, `exceedance_probability` |  |
| `y_units` | VARCHAR | 5 | no nulls | `probability_0_1`, `lbs_per_sqft` |  |
| `interpolation` | VARCHAR | 1 | no nulls | `piecewise_linear` | How to read between points. |
| `notes` | VARCHAR | 7 | no nulls | `Loss as a fraction of building replace`, `Probability of reaching or exceeding S` |  |

### `assignment_rules`

Hazus's own default curve selection. Note that Hazus 7.0 changed the coastal selection rule (depth-limited at 3 ft and 6 ft) without changing any curve; that rule is recorded here, not as curve data.

**2,040 rows** · primary key `(rule_id)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `rule_id` | VARCHAR | 2,040 | no nulls | `fl-6.1-structure-R13B -Riverine`, `fl-6.1-structure-R13N -CoastalV` |  |
| `peril` | VARCHAR | 1 | no nulls | `fl` |  |
| `hazus_version` | VARCHAR | 2 | no nulls | `4.0`, `6.1` |  |
| `damage_type` | VARCHAR | 3 | no nulls | `inventory`, `structure` |  |
| `occupancy` | VARCHAR | 33 | no nulls | `COM4`, `COM9` |  |
| `flood_zone` | VARCHAR | 3 | no nulls | `CoastalV`, `CoastalA` | Riverine \| CoastalA \| CoastalV. |
| `stories` | VARCHAR | 10 | no nulls | `3 Story`, `Split Level` |  |
| `basement` | VARCHAR | 3 | 351 null (17%) | `1`, `0` |  |
| `curve_id` | VARCHAR | 202 | no nulls | `fl-6.1-structure-109`, `fl-6.1-structure-110` | Curve this combination selects by default. |
| `source_file` | VARCHAR | 8 | no nulls | `Building_DDF_CoastalV_LUT_Hazus4p0.csv`, `Content_DDF_CoastalA_LUT_Hazus4p0.csv` |  |
| `source_table` | VARCHAR | 10 | no nulls | `flBldgContDmgFinal`, `Content_DDF_CoastalA_LUT_Hazus4p0` |  |
| `notes` | VARCHAR | 1 | 892 null (44%) | `Superseded in Hazus 7.0 by the depth-l` |  |

### `curve_zone_applicability`

Which flood zones Hazus flags a curve as applicable to. IMPORTANT: Hazus only publishes zone flags for the curves it assigns by default -- 47 of 597 structure curves in Hazus 4.0. The remainder are alternates a Hazus user selects manually and carry no zone assignment in any published table. Absence from this table therefore means 'Hazus states no zone', NOT 'not applicable'.

**494 rows** · primary key `(curve_id, flood_zone)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `curve_id` | VARCHAR | 202 | no nulls | `fl-4.0-structure-107`, `fl-4.0-structure-110` |  |
| `flood_zone` | VARCHAR | 3 | no nulls | `CoastalV`, `CoastalA` | Riverine \| CoastalA \| CoastalV. |
| `source_file` | VARCHAR | 8 | no nulls | `Building_DDF_CoastalV_LUT_Hazus4p0.csv`, `Content_DDF_CoastalV_LUT_Hazus4p0.csv` |  |
| `source_table` | VARCHAR | 10 | no nulls | `Content_DDF_CoastalA_LUT_Hazus4p0`, `flBldgContDmgFinal` |  |

### `dim_geographic_case`

Decomposes Hazus's geographic applicability cases into the territories each one covers. Hazus cases are SET-VALUED -- 'ContUS+Hawaii' applies in both CONUS and Hawaii -- so filtering by case equality silently excludes most applicable curves. Join through this table and match on territory instead.

**6 rows** · primary key `(case_name, territory)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `case_name` | VARCHAR | 4 | no nulls | `Hawaii`, `ContUS+Hawaii` | Hazus CaseName, e.g. ContUS+Hawaii. |
| `territory` | VARCHAR | 3 | no nulls | `CONUS`, `Hawaii` | CONUS \| Hawaii \| Caribbean. |
| `case_description` | VARCHAR | 4 | no nulls | `Used in Hawaii`, `Used in Continental and Hawaii` | Hazus CaseDescription, verbatim. |

### `dim_occupancy`

Hazus occupancy classes. Source values are space-padded; stripped on load.

**33 rows** · primary key `(occupancy)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `occupancy` | VARCHAR | 33 | no nulls | `COM4`, `COM9` |  |
| `category` | VARCHAR | 8 | no nulls | `Residential`, `Religion` |  |
| `description` | VARCHAR | 33 | no nulls | `Emergency Response`, `Multi-dwellings (20 to 49 units)` |  |

### `dim_building_type`

Hazus specific building types (hurricane).

**39 rows** · primary key `(building_type)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `building_type` | VARCHAR | 39 | no nulls | `SPMBM`, `WMUH2` |  |
| `description` | VARCHAR | 39 | no nulls | `Masonry Engineered Residential Buildin`, `Manufactured Home, 1976-1994` |  |

### `provenance`

Integrity record for every upstream file, copied from raw/MANIFEST.json so the database is self-describing once detached from the repo.

**19 rows** · primary key `(source_file)`

| column | type | distinct | nulls | example values | meaning |
|---|---|---|---|---|---|
| `source_file` | VARCHAR | 19 | no nulls | `Building_DDF_CoastalV_LUT_Hazus4p0.csv`, `fema_Hazus-6.1-Release-Notes.pdf` |  |
| `url` | VARCHAR | 19 | no nulls | `https://raw.githubusercontent.com/nhra`, `https://raw.githubusercontent.com/nhra` |  |
| `sha256` | VARCHAR | 19 | no nulls | `941d41388451290530a05a295ab5c6c7c480e8`, `75ae8d7460389da49102c5fedc774483f64fe0` |  |
| `bytes` | BIGINT | 19 | no nulls | `781943`, `286703` |  |
| `retrieved_at` | VARCHAR | 11 | no nulls | `2026-07-26T18:17:07+00:00`, `2026-07-26T15:28:54+00:00` |  |
| `hazus_version` | VARCHAR | 4 | no nulls | `7.1`, `6.1` |  |
| `note` | VARCHAR | 19 | no nulls | `Sheets: flBldgStrucDmgFn (2), flBldgSt`, `RSA-21186: the huDamLossFun multi-fami` |  |

## A note on the `_hu` files

`curves_hu.parquet`, `curve_points_hu.parquet` and `curve_attributes_hu.parquet` are hurricane-only slices of the corresponding combined tables. They exist because the web browser queries them directly, and they duplicate rows already present in the combined files.

For your own work prefer the combined tables and filter `peril = 'hu'`.
