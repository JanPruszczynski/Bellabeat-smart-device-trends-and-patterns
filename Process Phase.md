
![[Pasted image 20261007183021.png]]

# Bellabeat capstone — data cleaning log

**Phase:** Process  
**Tools:** Microsoft Excel and Google BigQuery Sandbox  
**Dataset:** Fitbit Fitness Tracker Data, published by Möbius on Kaggle  
**BigQuery dataset:** `able-goods-510915-t9.Bellabeat`  
**Log date:** October 9, 2026

This log records the work reported during the cleaning process. It does not represent an independent inspection of the tables.

## 1. Data included

| Data | Source tables |
|---|---|
| Daily activity | `daily_activities_mar_apr`, `daily_activities_apr_may` |
| Hourly intensity | `hourly_intensities_mar_apr`, `hourly_intensities_apr_may` |
| Hourly steps | `hourly_steps_mar_apr`, `hourly_steps_apr_may` |
| Daily sleep | `sleep_day_apr_may` |

Both collection periods were included for activity and hourly data. Sleep data was prepared from the April–May file.

## 2. Excel cleaning and CSV export

Reviewed the tables in Excel and exported the cleaned files as CSV for import into BigQuery.

**Correction to the earlier record:** Only 33 rows were deleted from the April–May daily activity file during Excel cleaning. No deletion of 180 rows is recorded.

The 33 rows were reported to have been deleted as entire Excel rows. However, a subsequent BigQuery check still found 33 entirely empty records in the uploaded April–May source table. The reason they remained in that upload was not established. These records were excluded during SQL processing; this is not recorded as a second, separate set of 33 observations removed.

## 3. Import settings corrected

Initial uploads produced date/time conversion errors because BigQuery inferred TIMESTAMP fields for strings containing dates and AM/PM times.

After the Excel export, the CSV delimiter and header settings were corrected to match the files:
- Semicolon (`;`) delimiter.
- One header row skipped.
- Date/time columns loaded as STRING for explicit conversion in SQL.

This resolved the import problems and allowed the records to load into separate columns.

## 4. Schema and date checks

Checked column names, column positions, and consistency between the two source periods before combining tables.

### Daily activity dates

Converted `ActivityDate` from STRING to DATE using:

```sql
SAFE.PARSE_DATE('%d.%m.%Y', TRIM(ActivityDate))
```

Example: `12.04.2016` became `2016-04-12`.

An earlier attempt using `SAFE_CAST(... AS DATE)` returned NULL because the input used day.month.year formatting. Recreated the merged output using the matching parse format.

### Hourly dates and times

Converted `ActivityHour` from STRING to DATETIME using:

```sql
SAFE.PARSE_DATETIME('%d.%m.%Y %H:%M', TRIM(ActivityHour))
```

Example: `12.04.2016 03:00` became `2016-04-12 03:00:00`.

### Sleep dates and times

Converted `SleepDay` using `PARSE_DATETIME`, as reported during cleaning. The exact format expression and final sleep table name were not recorded in the conversation.

Checked date ranges and conversion results. The checks were reported to pass after the parsing expressions were corrected.

## 5. Combining the collection periods

Combined corresponding March–April and April–May tables with `UNION ALL`, using `UNION ALL BY NAME` in the daily activity query to align columns by name.

Converted daily activity `Id` values to STRING during the merge so they remained identifiers rather than measurements.

Activity and sleep tables were not joined as part of this cleaning step. Combining periods adds rows; joining different measurements is a separate analysis operation.

## 6. Empty daily activity records excluded

The latest source-table check returned:

| Source | Uploaded rows | Rows with missing Id |
|---|---:|---:|
| March–April daily activity | 457 | 0 |
| April–May daily activity | 940 | 33 |

Inspected the 33 April–May records with missing IDs and confirmed that every column was NULL.

Because the work used BigQuery Sandbox, replaced the proposed DELETE operation with a filtered table creation. Added:

```sql
WHERE Id IS NOT NULL
```

to both source queries when recreating `daily_activities_merged`.

This condition was justified by the inspection confirming that the missing-ID records were entirely empty. The uploaded source tables were left unchanged by this SQL step.

**Row reconciliation before resolving overlaps:**  
457 + (940 − 33) = **1,364 records**.

## 7. Conflicting daily activity records resolved

Checked uniqueness using `Id` and `ActivityDate`. Found **22 user–date combinations with multiple records**, all dated **April 12, 2016**.

The activity values differed between sources. One inspected example was:

| Source | Id | Date | Steps | Calories |
|---|---|---|---:|---:|
| March–April | 1503960366 | 2016-04-12 | 224 | 50 |
| April–May | 1503960366 | 2016-04-12 | 13,162 | 1,985 |

The earlier record appeared to cover only part of the day, but this explanation was not independently confirmed.

**Cleaning decision:** Prefer the April–May record when both sources contain the same user and date. Keep March–April records without a matching April–May record.

Implemented this rule with a `source_table` label and a `NOT EXISTS` condition when recreating the merged table. The source label preserves the origin of each retained daily activity record.

This was a source-priority rule, not a rule to keep whichever row had more steps. It may affect April 12 results and should be documented with the analysis.

The subsequent duplicate check was reported to pass.

**Expected final daily activity count:**  
1,364 − 22 = **1,342 records**, assuming each of the 22 conflicting combinations contained exactly two records. The final count was not explicitly provided in the conversation.

## 8. Hourly intensity duplicates removed

Checked for multiple records with the same `Id` and `ActivityHour`.

Inspected the duplicated records and reported that their values were identical. Used `SELECT DISTINCT *` to retain one copy of each exact row. A query was provided to create `hourly_intensities_clean` from `hourly_intensities_merged`.

The final working table name and the number of rows removed were not explicitly recorded. The user subsequently reported that all datasets were cleaned.

## 9. Hourly steps duplicates removed

Applied:

```sql
CREATE OR REPLACE TABLE
  `able-goods-510915-t9.Bellabeat.hourly_steps_merged` AS
SELECT DISTINCT *
FROM `able-goods-510915-t9.Bellabeat.hourly_steps_merged`;
```

This replaced the merged hourly steps table with its distinct rows.

A follow-up uniqueness check on `Id` and `ActivityHour` was provided. The user subsequently reported that all datasets were cleaned. The number of exact duplicate rows removed was not recorded.

## 10. Validation

The following daily activity checks were run and reported to pass:

- No remaining duplicated user–date combinations.
- No missing IDs or activity dates.
- No missing steps, calories, or activity-minute fields.
- No negative steps, total distance, calories, or activity-minute values.
- No combined active and sedentary minutes above 1,440 per day.

Date ranges were also checked and reported to be correct.

Hourly user–hour duplicate checks were performed, and exact-row deduplication was applied as described above. Sleep date/time conversion was completed. The user reported completion of cleaning for all included datasets; detailed sleep validation outputs and final hourly/sleep row counts were not supplied.

Zero values were not treated as missing values merely because they were zero. No additional row deletions are recorded without supporting evidence.

## 11. Cleaning outcome and records to retain

The tables were prepared for analysis through import corrections, date/time conversion, exclusion of entirely empty records, resolution of conflicting daily records, and removal of exact hourly duplicates.



