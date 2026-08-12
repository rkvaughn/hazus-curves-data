# Provenance

Where every number in these files comes from. The full account, including the parts that
could not be verified, is in the main repository at
[docs/provenance.md](https://github.com/rkvaughn/hazus-curves/blob/main/docs/provenance.md).

## Sources

Hazus is a Windows-only ArcGIS Pro application backed by SQL Server. There is no official
standalone download of its damage function tables and no API. These files were assembled
from two public sources that had already escaped that enclosure:

- **Hazus 4.0 flood** — FEMA's own Flood Assessment Structure Tool repository
  (`github.com/nhrap-hazus/FAST`), which publishes direct CSV dumps of the Hazus SQL
  tables.
- **Hazus 6.1 flood and hurricane** — Excel extracts from a public bucket operated by the
  OS-Climate physrisk project, which names that exact bucket and file in its published
  source code.

**That bucket was disabled on 12 August 2026** and now returns HTTP 403 for every object.
The original files can no longer be downloaded or byte-compared by anyone. SHA-256
verified copies are re-hosted by the main repository.

## Integrity record

Every input file, with the hash recorded when it was retrieved:

| source file | Hazus version | bytes | sha256 |
|---|---|---|---|
| `Building_DDF_CoastalA_LUT_Hazus4p0.csv` | 4.0 | 11,582 | `e949a4a6a9b11c19…` |
| `Building_DDF_CoastalV_LUT_Hazus4p0.csv` | 4.0 | 11,571 | `b5b56c1dcd2eb9f0…` |
| `Building_DDF_Riverine_LUT_Hazus4p0.csv` | 4.0 | 33,699 | `941d413884512905…` |
| `Content_DDF_CoastalA_LUT_Hazus4p0.csv` | 4.0 | 12,120 | `45182435896dd7a2…` |
| `Content_DDF_CoastalV_LUT_Hazus4p0.csv` | 4.0 | 12,116 | `367421fc8c6b567f…` |
| `Content_DDF_Riverine_LUT_Hazus4p0.csv` | 4.0 | 37,054 | `e9879f6de93ba8b3…` |
| `HazusFloodDamageFunctions_Hazus61.xlsx` | 6.1 | 403,618 | `e32e0a7f55d6efd8…` |
| `HazusWindDamFunctions_Hazus61.xlsx` | 6.1 | 102,182,624 | `0fe93bbdc5d4ae3a…` |
| `Inventory_DDF_LUT_Hazus4p0.csv` | 4.0 | 9,849 | `e5b502dc0d483063…` |
| `OccupancyTypes.csv` | 4.0 | 4,959 | `e5cd56817dfd2790…` |
| `fema_Hazus-6.1-Release-Notes.pdf` | 6.1 | 781,943 | `75ae8d7460389da4…` |
| `fema_hazus-7-1-release-notes.pdf` | 7.1 | 286,703 | `ee156040d4c973ba…` |
| `fema_hazus_7_release_notes.pdf` | 7.0 | 1,686,388 | `bcf0d092089f89a8…` |
| `fema_rsl_hazus-7-fltm_06272025_0.pdf` | 7.0 | 1,234,449 | `4fd5a0402dab23c9…` |
| `fema_rsl_hazus-7-hutm_06272025_0.pdf` | 7.0 | 32,374,860 | `7a1e3189db4b38c7…` |
| `flBldgContDmgFn.csv` | 4.0 | 86,626 | `4aac53864935e688…` |
| `flBldgInvDmgFn.csv` | 4.0 | 18,141 | `918d65589f8f1d93…` |
| `flBldgStructDmgFn.csv` | 4.0 | 98,616 | `b7fc0e12c618165c…` |
| `flUtilFltyDmgFn.csv` | 6.1 | 7,480 | `30b2573bcbaa8d16…` |

The same record ships inside the data as `provenance.parquet`, so a detached copy of
these files remains self-describing.

## What the pipeline does and does not do

It renames columns and reshapes wide source tables into long format. It performs **no
arithmetic on damage values** — nothing is computed, interpolated, smoothed, clamped or
inferred, and a missing source value is preserved as `NULL` rather than estimated.

Every row in `curves` carries `source_file`, `source_table` and `source_row_id`
identifying the exact originating cell.

## Verification against FEMA documents

Nine quantities FEMA states in its published manuals were tested against this data. Eight
match; one is a partial match reported as an unresolved discrepancy. The full report,
with FEMA source pages reproduced as screenshots alongside URLs and page numbers, is at
[docs/Hazus_Data_Verification_Report.docx](https://github.com/rkvaughn/hazus-curves/blob/main/docs/Hazus_Data_Verification_Report.docx).

Highlights:

- FEMA states 62 wind building characteristics; the data contains exactly 62.
- FEMA states 39 specific building types; all 39 names match Table C-1 verbatim.
- FEMA states "over 275,000" hurricane damage functions; the data contains 275,220.
- FEMA disclosed a specific defect in the Hazus 6.1 wind data in its 7.1 release notes;
  **that defect is present and reproducible here at row level.** A fabricated or
  re-derived dataset would not carry another organisation's bug.

Independently, the flood curves agree with a separate FEMA publication across 17,313
individual values with zero discrepancies.

## Known defects

FEMA's Hazus 7.1 release notes disclose that in Hazus 6.1 and 7.0 the 2- and 3-story
multi-family wind damage functions were inadvertently overwritten with copies of the
1-story functions. Comparing every affected curve against its 1-story counterpart with
identical characteristics, terrain and loss class confirms it.

Affected curves are flagged in the data (`defect_flag`, `defect_verified`) rather than
removed. The corrected functions exist only in Hazus 7.1+, which is not publicly
extractable.

Other upstream quirks preserved as-is — including probabilities marginally exceeding 1.0
and nine building characteristic codes missing from Hazus's own decode table — are
documented at [docs/data_quality.md](https://github.com/rkvaughn/hazus-curves/blob/main/docs/data_quality.md).

## Vintage

Flood curve values are current: FEMA's documentation shows the flood library unchanged
from Hazus 6.1 through 7.1. Hurricane data is Hazus 6.1 and carries the defect above.
Hazus 7.2's release notes ship inside a CAPTCHA-gated download, so what changed there is
unknown.
