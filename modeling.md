# Modeling: Total Passengers — Measures, Validation Queries, and Power Query Fix

This file collects interview-ready DAX measures, DAX Studio validation queries, a recommended Power Query (M) fix, and short talking points you can use when applying for BI Engineer / Semantic Model roles. I converted the guidance and examples we discussed into a single reference file.

---

## Measures

1) Fast (preferred when `Passengers On` is numeric)
```DAX
Total Passengers =
SUM( 'Actual Trips'[Passengers On] )
```

2) Robust (handles blanks, trimmed text, and non-numeric values)
```DAX
Total Passengers (Robust) =
SUMX(
  'Actual Trips',
  VAR v = TRIM( COALESCE( 'Actual Trips'[Passengers On], "" ) )
  RETURN
    IF(
      v = "",
      0,
      IFERROR( VALUE( v ), 0 )
    )
)
```

3) Ignore filters / grand total (always returns overall total)
```DAX
Total Passengers (All Trips) =
CALCULATE( [Total Passengers (Robust)], ALL( 'Actual Trips' ) )
```

4) Distinct passengers (if you track individual passenger IDs and want unique counts)
```DAX
Distinct Passengers =
DISTINCTCOUNT( 'Actual Trips'[PassengerID] )
```

---

## DAX Studio Validation Queries

1) Scalar check — returns the measure value
```DAX
EVALUATE
ROW( "Total Passengers (Robust)", [Total Passengers (Robust)] )
```

2) Compare measure to raw SUM (useful when column is numeric)
```DAX
EVALUATE
ROW(
  "SUM_PassengersOn", SUM( 'Actual Trips'[Passengers On] ),
  "Measure_TotalPassengers", [Total Passengers]
)
```

3) Compare robust measure to numeric conversion of raw column
```DAX
EVALUATE
ROW(
  "SUM_PassengersOn_AsNumber",
    SUMX(
      'Actual Trips',
      IF(
        TRIM( COALESCE( 'Actual Trips'[Passengers On], "" ) ) = "",
        0,
        IFERROR( VALUE( TRIM( 'Actual Trips'[Passengers On] ) ), 0 )
      )
    ),
  "Measure_TotalPassengers_Robust", [Total Passengers (Robust)]
)
```

4) Top routes by numeric column vs measure (spot per-route mismatches)
```DAX
EVALUATE
ADDCOLUMNS(
  TOPN(
    50,
    VALUES( 'Actual Trips'[Route] ),
    CALCULATE( SUMX( 'Actual Trips', IF( TRIM( COALESCE( 'Actual Trips'[Passengers On], "" ) ) = "", 0, IFERROR( VALUE( TRIM( 'Actual Trips'[Passengers On] ) ), 0 ) ) ) ),
    DESC
  ),
  "PassengersOn_Column_Num", CALCULATE( SUMX( 'Actual Trips', IF( TRIM( COALESCE( 'Actual Trips'[Passengers On], "" ) ) = "", 0, IFERROR( VALUE( TRIM( 'Actual Trips'[Passengers On] ) ), 0 ) ) ) ),
  "TotalPassengers_Robust", [Total Passengers (Robust)]
)
```

5) Sample rows where `Passengers On` is blank or non-numeric (change ordering column as needed)
```DAX
EVALUATE
FILTER(
  TOPN(200, 'Actual Trips', 'Actual Trips'[TripDate], ASC ),
  OR(
    ISBLANK( 'Actual Trips'[Passengers On] ),
    NOT( ISNUMBER( VALUE( TRIM( COALESCE( 'Actual Trips'[Passengers On], "" ) ) ) ) )
  )
)
```
Note: If your engine does not allow VALUE inside NOT/ISNUMBER directly in FILTER, create a calculated column flagging invalid rows for fast inspection.

---

## Power Query (M) Recommendation

Convert `Passengers On` to numeric during refresh to simplify DAX and improve performance.

Example transformation (replace `PreviousStepName` with the previous step name in your query):
```m
#"Changed Type" = Table.TransformColumns(
  PreviousStepName,
  { "Passengers On", each try Number.FromText(Text.Trim(Text.From(_))) otherwise null, type nullable number }
)
```
Why: fixes data issues upstream so you can use the simple `SUM()` measure in the semantic model (faster and cleaner).

---

## Interview Talking Points (short)

- Use `SUM()` when the column is numeric — it's the most performant aggregation.
- Use `SUMX()` with safe conversion only if the column contains text/dirty data.
- Prefer fixing data in Power Query to simplify DAX and improve model refresh/query performance.
- Provide an "All" measure with `CALCULATE(..., ALL(table))` for KPIs that ignore slicers/filters.
- Validate measures in DAX Studio with scalar rows, per-dimension comparisons, and sampling of bad rows.

---

## What I did and next steps

I consolidated the robust and performant DAX measure variants, ready-to-run DAX Studio validation queries, a Power Query M snippet to fix the source column, and concise interviewing talking points into a single `modeling.md` file for easy reference.

Next you can:
- Replace placeholder column/table names (e.g., `TripDate`, `PassengerID`) with the exact names in your model,
- I can produce a calculated column to flag non-numeric rows for quick inspection,
- Or I can produce a Power Query step adapted to your query step names. Tell me which you want and I'll update the file.
