# Power Query Notes

Decisions and checks made while transforming the CSO HPA06 data in Power Query.

## Data source

- Source: CSO table HPA06, Residential Property Price Index (data.cso.ie/table/HPA06)
- File: `Data/HPA06.20260928154352.xlsx` (Excel download)
- Annual data, 2005–2025
- Values are an index, not euro prices. Base: January 2005 = 100.
- Because the base is a single month and the data is annual, 2005 values are above 100
  (e.g. National 2005 = 105.5, meaning the 2005 average was 5.5% above January 2005).
- Raw file: 8 columns, 840 rows (20 area/property type pairs × 21 years × 2 statistics).

## Query structure

```
Excel file
     ↓
HPA06_Base (shared steps, load disabled)
     ↓                         ↓
FactPropertyIndex          CheckYoY
(index, HPA06C01)          (CSO % change, HPA06C02)
```

- **HPA06_Base** holds the shared steps, so a fix there flows to both child queries.
  Load is disabled because it still mixes both statistics.
- **FactPropertyIndex** is the main fact table. Year-on-year change will be calculated
  in DAX from this index.
- **CheckYoY** keeps CSO's own published % change, to check my DAX YoY measure against.
- Used Reference (not Duplicate) so both child queries always follow the base.

## Steps in HPA06_Base

1. Kept the automatic type detection after checking it: VALUE became decimal,
   blanks became null (0% errors).
2. Split "Type of Residential Property" at the **right-most** " - " delimiter.
   - The property type is always the last part of the label.
   - Using " - " with spaces keeps area names with hyphens intact
     (e.g. Dun Laoghaire-Rathdown).
3. Renamed the split columns to **Area** and **PropertyType**.

## Steps in each child query

1. Filtered STATISTIC (HPA06C01 for FactPropertyIndex, HPA06C02 for CheckYoY).
2. Kept only Year, Area, PropertyType and the value column.
3. Renamed VALUE to **PriceIndex** (FactPropertyIndex) and **CSO_YoYPct** (CheckYoY).

## Columns removed and why

| Column | Reason |
|---|---|
| STATISTIC | One value after filtering, so no information |
| Statistic Label | Readable version of STATISTIC, same problem |
| TLIST(A1) | Duplicates Year in this annual table |
| C02803V03373 | CSO code for area + property type, already covered by Area and PropertyType; missing for one row |
| UNIT | One value after filtering ("Base Jan 2005 = 100"); kept in documentation instead |

## Data facts found

- **14 areas, 3 property types, 20 area/property type pairs.**
- Only **National, Dublin and National excluding Dublin** have all three property types
  (houses, apartments, all residential properties).
- The other 11 areas (4 Dublin sub-areas and 7 regions) are **houses only**.
  Checked by filtering to apartments: only 3 distinct areas.
- CSO region names differ from simple region names and must be matched exactly:
  - Border excluding Louth
  - Mid-East including Louth
  - Mid-West including South Tipperary
  - South-East excluding South Tipperary
  - Dun Laoghaire-Rathdown (no accent)

## Checks

| Query | Columns | Rows |
|---|---|---|
| HPA06_Base | 9 | 840 |
| FactPropertyIndex | 4 | 420 |
| CheckYoY | 4 | 420 |

**Nulls are expected and explained:**
- PriceIndex: 13% null. 11 areas have no data for 2005–2009 (11 × 5 = 55 rows; 55 / 420 = 13%).
- CSO_YoYPct: 18% null. The same 55 rows, plus 2005 for the other 9 pairs (no 2004),
  plus 2010 for the 11 late-starting areas (no 2009). 75 / 420 = 18%.

## DimArea (area lookup table)

Built by hand with Home → Enter Data. One row per area (14 rows).

| Column | Type | Purpose |
|---|---|---|
| Area | Text | Links to FactPropertyIndex[Area]. Spelling copied exactly from the data. |
| Level | Text | Groups areas so totals are not compared with their own parts. |
| SortOrder | Whole Number | Controls the order areas appear in charts and slicers (default is alphabetical). |

**Level groups:**
- **National total:** National, National excluding Dublin
- **Dublin total:** Dublin
- **Dublin sub-area:** Dublin City, Dun Laoghaire-Rathdown, Fingal, South Dublin
- **Region:** Border excluding Louth, Midland, West, Mid-East including Louth,
  Mid-West including South Tipperary, South-East excluding South Tipperary, South-West

**Why Level matters:**
- The Dublin Deep-Dive page can filter to Level = "Dublin sub-area".
- Rankings do not mix totals (National) with parts (Fingal).
- Note: Dublin is also one of Ireland's 8 official regions. The Regional Comparison
  page may need to show Dublin together with the 7 regions.

**Check:** 3 columns, 14 rows, 0% errors.

## DimDate (year table)

Built from a blank query using a list of years, then converted to a table.

| Column | Type | Purpose |
|---|---|---|
| Year | Whole Number | Links to FactPropertyIndex[Year]. Same type as the fact table, so the relationship works. |

- 21 rows: 2005 to 2025.
- **Limitation:** the year range is hard-coded (2005–2025). If CSO adds a new year
  and the data is refreshed, DimDate must be extended by hand.

**Check:** 1 column, 21 rows, 0% errors.

## Final query summary

| Query | Loads into model? | Rows |
|---|---|---|
| HPA06_Base | No (shared steps only) | 840 |
| FactPropertyIndex | Yes | 420 |
| CheckYoY | Yes | 420 |
| DimArea | Yes | 14 |
| DimDate | Yes | 21 |