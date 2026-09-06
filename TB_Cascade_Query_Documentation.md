# TB Cascade Query Documentation

## Purpose

This Microsoft Access query creates a TB cascade extract for visits from 1 July 2026 through 30 September 2026. It returns visit, patient, screening, TB-result, breastfeeding, TB-treatment, facility, and ART-start information.

The query returns one row per visit. A patient can therefore appear more than once if they attended multiple visits during the reporting period.

## Query file

The SQL is saved in `tb_cascade_2026_Q3_access.sql`.

## How the query works

Think of the query as starting with a list of visits and then attaching patient details, TB test results, TB-treatment evidence, and ART start dates.

### 1. Output columns

The `SELECT` section determines which columns appear in the results:

- `PatientID`: the client's identifier from `tblVisits`.
- `VisitTypeCode`: the type of visit from `tblVisits`.
- `HFRCode`: the facility HFR code from the first row of `tblConfig`.
- `NumDaysDispensed`: the number of ART days dispensed at the visit.
- `TBScreeningIDSameDayScreening`: the initial same-day TB screening outcome.
- `Weight`: the weight recorded at the visit.
- `TBScreeningID`: the TB investigation or result, such as `CXR+` or `GX-ve`.
- `ScreenedDate`: `tblVisits.VisitDate`, renamed because the database has no separate TB-screening date field.
- `Sex`: the patient's sex from `tblPatients`.
- `Breastfeeding`: a binary breastfeeding indicator.
- `TB_POS`: marks confirmed positive TB evidence.
- `POS_Started_TB_Treatment`: shows whether a positive client has evidence of starting TB treatment.
- `DateStartART`: the earliest documented ART start date.

`V`, `P`, `T`, `TP`, `VT`, `X`, and `A` are short aliases used to make the SQL easier to read.

### 2. Breastfeeding binary

```sql
IIf(UCase(Trim(V.NowBreastfeeding)) = "YES", 1, 0) AS Breastfeeding
```

This expression:

1. Removes extra spaces with `Trim`.
2. Converts the value to uppercase with `UCase`.
3. Returns `1` when the cleaned value is `YES`.
4. Returns `0` for every other value.

Consequently, explicit `No`, blank, and missing values are all returned as `0`. In the inspected reporting period, every source breastfeeding value was blank, so every output row currently has `Breastfeeding = 0`.

### 3. Identifying a positive TB case

The query checks three possible sources of positive evidence.

#### Visit-level TB result

It checks `tblVisits.TBScreeningID` for these positive codes:

- `CXR+`
- `GX+ve`
- `SS+`
- `TB LAM +ve`
- `Ped SC+ve`
- `TB SC +ve`
- `Conf/Yes`

#### Same-day screening field

It checks `tblVisits.TBScreeningIDSameDayScreening` for the same positive codes. This supports databases that store a positive result in the same-day field.

#### Laboratory tests

The `TP` subquery checks `tblTests` for TB-related test types:

- `CXR`: chest X-ray
- `GX`: GeneXpert
- `SS`: sputum smear
- `LAM`: TB-LAM

A test is treated as positive when `ResultNumeric = 1`, or when the cleaned written result is `POS`, `POSITIVE`, or `DETECTED`. Inspection of the database confirmed that `ResultNumeric = 1` represents positive and `ResultNumeric = 2` represents negative for the TB tests.

If any of the three checks finds positive evidence, `TB_POS` is `POS`. Otherwise, it remains blank. A blank does not necessarily mean negative; it can also mean pending, not tested, or missing.

`SELECT DISTINCT` in the `TP` subquery prevents multiple positive laboratory tests from multiplying the visit rows.

The laboratory result is normally matched using `RefVisitDate`. If `RefVisitDate` is missing, the query uses `TestDate`.

### 4. Checking whether a positive client started TB treatment

`POS_Started_TB_Treatment` is evaluated only for rows identified as `POS`.

The query searches the same patient's visits on or after the positive screening date. Treatment evidence includes:

- `START TB`
- `CTN TB`, meaning continued TB treatment
- `CPLT TB`, meaning completed TB treatment
- `STOP TB`, which proves treatment had previously started
- A recorded `AntiTBStartDate`
- More than zero TB-drug days dispensed

The output is:

- `1`: positive client with TB-treatment evidence.
- `0`: positive client without TB-treatment evidence.
- Blank: the client was not identified as positive, so the treatment indicator is not applicable.

In the inspected database, the reporting period contains one positive case. That client has `START TB` recorded on 18 August 2026 and received 14 TB-drug days, so the indicator is `1`.

### 5. Linking visits to patients

```sql
tblVisits AS V
INNER JOIN tblPatients AS P
    ON V.PatientID = P.PatientID
```

This joins each visit to its patient record so the query can return sex.

Because it uses an `INNER JOIN`, a visit without a matching row in `tblPatients` would be excluded. All visits in the inspected reporting period had matching patient records.

### 6. Calculating ART start date

The query collects possible ART start dates from three places:

1. `tblVisits.VisitDate` where `ARVStatusCode = 2`, meaning Start ARV.
2. `tblStartARTanotherClinic.DateStartARTAtAnotherClinic` for clients who began ART elsewhere.
3. `tblTreatmentBaselineValues.DateStartART` where baseline treatment data is available.

`UNION ALL` combines those lists. `Min(DateStartART)` then selects the earliest documented ART date for each patient.

### 7. Reporting-period filter

```sql
WHERE V.VisitDate >= #07/01/2026#
  AND V.VisitDate <  #10/01/2026#
```

Microsoft Access date literals use `#` signs and month/day/year order.

The filter starts at 1 July 2026 and stops immediately before 1 October 2026. It therefore includes every visit through 30 September 2026, including records with a time component.

### 8. Sorting

```sql
ORDER BY V.VisitDate, V.PatientID;
```

The results are ordered by screening date and then by patient ID.

## Validation results

The query was executed successfully against `KIDELEKO HC DATASET 3 SEPT 2026.mdb` using the Microsoft Access database engine.

- Result rows: 169
- Distinct patients: 128
- Facility HFR code: `102656-6`
- Positive TB cases: 1
- Positive cases with TB-treatment evidence: 1 of 1
- Rows with an ART start date: 167
- Breastfeeding values equal to 1: 0
- Earliest returned visit: 1 July 2026
- Latest available visit in the database: 3 September 2026

The SQL still covers the full requested period through 30 September 2026; the source database simply contained no later visits when inspected.

## Running the query in Microsoft Access

1. Open the facility database.
2. Select **Create** and then **Query Design**.
3. Close the table-selection window if it appears.
4. Switch to **SQL View**.
5. Paste the contents of `tb_cascade_2026_Q3_access.sql`.
6. Save the query with a suitable name, such as `qryTBCascadeQ3_2026`.
7. Run the query.

The query can be reused with other facility databases that have the same table and field structure. `HFRCode` is read from each database's own `tblConfig` record.
