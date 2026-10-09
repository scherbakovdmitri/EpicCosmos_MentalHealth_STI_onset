# Final primary analysis: SQL and R

Primary analyses in both directions: STI testing → mental health diagnosis and mental health diagnosis → STI testing. Models adjust for clinical visits at ages 12–<14. Supplemental analyses are excluded.

## Requirements

- Run in the authorized Cosmos environment using an existing DBI connection named `con` to the saved project tables.
- Reuse the saved cohort. The historical cohort SQL below draws a random sample; rerunning it can change the cohort and overwrite project tables.
- Prepare the session-local eligible-cohort table described in the visit SQL, then load its complete output into R as `util_visits` on the appropriate connection. This preparation and connection setup are environment prerequisites rather than steps supplied by this script.
- Install the R packages `DBI`, `dplyr`, `survival`, `ggplot2`, and `scales`.
- Copy the R blocks into one script in the order shown.

## Cohort selection and original extraction (SQL)

Historical extraction retained for audit. The saved `dt.Value = 'F60%'` equality condition is preserved and requires review; it is not a SQL wildcard match. The original active-registry cutoff is also preserved.

```sql
-- ================================================================
-- Cosmos extraction: Patients with >=1 Dx AND >=1 Proc
-- using the saved DATEDIFF(YEAR) age 11-12 criterion in 2017,
-- in Active Data Mart. 200K random sample.
-- ================================================================

-- --------------------------------------------------------------
-- STEP 1: Patients with qualifying Dx (age 11-12 at Dx, 2017)
-- --------------------------------------------------------------
DROP TABLE IF EXISTS #TempPatients;

SELECT DISTINCT
    dx.PatientDurableKey,
    pd.ReliableSex,
    pd.FirstRace,
    pd.BirthDate,                              -- DATE type
    pd.Ethnicity                              AS Ethnicity,
    pd.Sex                                    AS Sex,
    pd.MaritalStatus                          AS MaritalStatus,
    pd.SviSocioeconomicPctlRankingByZip2018_X AS Svi2018,
    pd.SviSocioeconomicPctlRankingByZip2020_X AS Svi2020
INTO #TempPatients
FROM dbo.DiagnosisEventFact dx
INNER JOIN dbo.DiagnosisTerminologyDim dt
        ON dx.DiagnosisKey = dt.DiagnosisKey
INNER JOIN dbo.PatientDim pd
        ON dx.PatientDurableKey = pd.DurableKey
INNER JOIN dbo.ActivePatientRegistryDataMartX ap
        ON pd.DurableKey = ap.PatientDurableKey
WHERE dx.StartDateKey >= 20170101
  AND dx.StartDateKey <= 20171231
  AND ap.StartDateKey <= 20170101
  AND ap.EndDateKey >= 20260401
  AND dt.Type = 'ICD-10-CM'
  AND pd.BirthDate IS NOT NULL
  AND pd.UseInCosmosAnalytics_X = 1
  AND DATEDIFF(YEAR, pd.BirthDate,
        TRY_CONVERT(DATE, TRY_CONVERT(VARCHAR(8), dx.StartDateKey))) BETWEEN 11 AND 12
  -- Patient must ALSO have a procedure under same age/date logic
  AND EXISTS (
        SELECT 1
        FROM dbo.ProcedureEventFact pr
        INNER JOIN dbo.ActivePatientRegistryDataMartX ap2
                ON pr.PatientDurableKey = ap2.PatientDurableKey
        WHERE pr.PatientDurableKey = dx.PatientDurableKey
          AND pr.ProcedureStartDateKey >= 20170101
          AND pr.ProcedureStartDateKey <= 20171231
          AND ap2.StartDateKey <= 20170101
          AND ap2.EndDateKey >= 20260401
          AND DATEDIFF(YEAR, pd.BirthDate,
                TRY_CONVERT(DATE, TRY_CONVERT(VARCHAR(8), pr.ProcedureStartDateKey))) BETWEEN 11 AND 12
  );

-- --------------------------------------------------------------
-- STEP 2: 200K random sample from qualifying patients
-- --------------------------------------------------------------
DROP TABLE IF EXISTS #SampledPatients;

SELECT TOP (200000)
    PatientDurableKey,
    ReliableSex,
    FirstRace,
    BirthDate,
    Ethnicity,
    Sex,
    MaritalStatus,
    Svi2018,
    Svi2020
INTO #SampledPatients
FROM #TempPatients
ORDER BY NEWID();

-- --------------------------------------------------------------
-- STEP 3: Push patients to Projects
-- --------------------------------------------------------------
INSERT INTO PROJECTS.ProjectD217C48.dbo.DASTempTablePatientsDxProcBefore12CoxKorte
(PatientDurableKey, ReliableSex, FirstRace, BirthDate,
 Ethnicity, Sex, MaritalStatus, Svi2020, Svi2018)
SELECT
    PatientDurableKey, ReliableSex, FirstRace, BirthDate,
    Ethnicity, Sex, MaritalStatus, Svi2020, Svi2018
FROM #SampledPatients;

---Dx extract

DROP TABLE IF EXISTS #DASTempTableDxDxProcBefore12CoxKorte;
SELECT DISTINCT
    dx.PatientDurableKey,
    dx.StartDateKey,
    dx.EndDateKey,
    dx.DiagnosisKey,
    dt.DiagnosisTerminologyKey,
    dt.Type           AS DxType,
    dt.Value          AS DxValue,
    dt.DisplayString  AS DxDisplayString
INTO #DASTempTableDxDxProcBefore12CoxKorte
FROM dbo.DiagnosisEventFact dx
INNER JOIN dbo.DiagnosisTerminologyDim dt
        ON dx.DiagnosisKey = dt.DiagnosisKey
WHERE dx.PatientDurableKey IN (SELECT DISTINCT PatientDurableKey FROM PROJECTS.ProjectD217C48.dbo.DASTempTablePatientsDxProcBefore12CoxKorte)
  AND dt.Type = 'ICD-10-CM'
  AND (
        dt.Value LIKE 'A5%'        --
     OR dt.Value LIKE 'B20%'       --
     OR dt.Value LIKE 'F41%'       --
     OR dt.Value LIKE 'F32%'       --
     OR dt.Value LIKE 'F33%'       --
     OR dt.Value =    'F60%'       -- Saved extraction condition; requires review
     OR dt.Value LIKE 'F31%'       --
     OR dt.Value LIKE 'F43%'       --
  );
--F41|^F32|^F33|^F60.3|^F31|^F43

INSERT INTO PROJECTS.ProjectD217C48.dbo.DASTempTableDxDxProcBefore12CoxKorte
SELECT * FROM #DASTempTableDxDxProcBefore12CoxKorte;

----- PROC
-- ================================================================
-- Now get all procedures for these patients (STI/HIV CPT codes)
-- ================================================================
DROP TABLE IF EXISTS #DASTempTableProcDxProcBefore12CoxKorte;

SELECT DISTINCT
    pf.PatientDurableKey,
    pf.ProcedureStartDateKey,
    pr.CptCode,
    pr.CodeSet,
    pr.ShortName
INTO #DASTempTableProcDxProcBefore12CoxKorte
FROM ProcedureEventFact pf
INNER JOIN ProcedureDim pr ON pf.ProcedureDurableKey = pr.DurableKey
WHERE pr.CptCode IN (
        '87491',     -- Chlamydia NAA
        '87591',     -- Gonorrhea NAA
        '86592',     -- Syphilis non-trep qual (RPR/VDRL/ART)
        '86593',     -- Syphilis non-trep quant
        '86780',     -- Treponemal Ab
        '87800',     -- NAA multi-organism
        '86701',     -- HIV-1 Ab
        '86702',     -- HIV-2 Ab
        '86703',     -- HIV-1/HIV-2 single assay
        '87389'      -- HIV-1/HIV-2 Ag combo immunoassay
      )
  AND pf.PatientDurableKey IN (
        SELECT DISTINCT PatientDurableKey
        FROM PROJECTS.ProjectD217C48.dbo.DASTempTablePatientsDxProcBefore12CoxKorte
      );

INSERT INTO PROJECTS.ProjectD217C48.dbo.DASTempTableProcDxProcBefore12CoxKorte
SELECT * FROM #DASTempTableProcDxProcBefore12CoxKorte;
-- ================================================================
```

## Clinical visit extraction (SQL)

```sql
-- Run on the Cosmos server, database Cosmos.
-- Session-local #VisitCohort_20261005 contains the 150,881 eligible July IDs
-- and their birth dates, loaded with DBI::dbWriteTable on this connection.
-- A unique clustered index on PatientDurableKey avoids the cross-server join.
-- Patient records stay inside the Cosmos project.
SELECT
  CAST(e.PatientDurableKey AS varchar(20)) AS PatientDurableKey,
  CAST(e.EncounterKey AS varchar(20)) AS EncounterKey,
  TRY_CONVERT(date, CONVERT(varchar(8), e.DateKey), 112) AS VisitDate,
  e.[Type] AS EncounterType
FROM dbo.EncounterFact e
JOIN #VisitCohort_20261005 p
  ON e.PatientDurableKey = p.PatientDurableKey
WHERE e._IsDeleted = 0
  AND e.DateKey BETWEEN 20170101 AND 20260526
  AND e.DateKey >= CAST(CONVERT(char(8), DATEADD(day, 4383, p.BirthDate), 112) AS bigint)
  AND (e.IsOutpatientFaceToFaceVisit = 1 OR e.IsEdVisit = 1
    OR e.IsHospitalOutpatientVisit = 1
    OR e.[Type] IN ('Office Visit', 'Hospital Encounter', 'Hospital',
      'Telemedicine', 'E-Visit', 'Urgent Care', 'Walk-In'))
  AND e.[Type] NOT IN ('Telephone', 'Refill', 'Patient Message', 'Orders Only',
    'Scanned Document', 'Documentation', 'Abstract', 'Lab', 'Nurse Triage',
    'Results Follow-Up', 'Ancillary Orders', 'Patient Outreach', 'Legacy Encounter',
    'Lab Requisition', 'Case Management', 'Pre-Op/Pre-procedure Orders', 'Research Encounter');
```

## Final primary analysis (R)

### Setup

```r
# === Setup ===
library(DBI)
library(dplyr)
library(survival)
library(ggplot2)

# Run inside Cosmos with the existing project connection `con`.
# `util_visits` is the complete clinical encounter extract produced by
# clinical_visit_extract.sql for the eligible cohort.
# Reuse the saved cohort; do not draw another random sample.
output_dir <- "Final_primary_analysis"
dir.create(output_dir, showWarnings = FALSE)
study_end_date <- as.Date("2026-05-26")

```

### Load the saved cohort

```r
# === Load the saved cohort ===
patients <- dbGetQuery(
  con, "SELECT * FROM DASTempTablePatientsDxProcBefore12CoxKorte"
) %>%
  distinct() %>%
  filter(ReliableSex %in% c("Female", "Male")) %>%
  mutate(
    PatientDurableKey = as.character(PatientDurableKey),
    BirthDate = as.Date(BirthDate)
  )

# Keep the saved diagnosis extraction and filtering used in the final run.
# The extraction SQL's F60 condition is preserved, not silently corrected.
dx <- dbGetQuery(
  con, "SELECT * FROM DASTempTableDxDxProcBefore12CoxKorte"
) %>%
  distinct() %>%
  filter(grepl("^A5|^B20|^F41|^F32|^F33|^F60.3|^F31|^F43", DxValue)) %>%
  mutate(PatientDurableKey = as.character(PatientDurableKey))

proc <- dbGetQuery(
  con, "SELECT * FROM DASTempTableProcDxProcBefore12CoxKorte"
) %>%
  distinct() %>%
  mutate(PatientDurableKey = as.character(PatientDurableKey))

```

### Diagnosis and testing dates

```r
# === Diagnosis and testing dates ===
full_data <- patients %>%
  left_join(dx, by = "PatientDurableKey", relationship = "one-to-many") %>%
  mutate(
    DxDate = as.Date(as.character(StartDateKey), format = "%Y%m%d"),
    AgeAtDx = as.numeric(DxDate - BirthDate) / 365.25,
    MH = ifelse(
      !is.na(DxValue) &
        grepl("^F41|^F32|^F33|^F60.3|^F31|^F43", DxValue),
      1L, 0L
    )
  )

proc <- proc %>%
  mutate(
    TestDate = as.Date(as.character(ProcedureStartDateKey), format = "%Y%m%d")
  ) %>%
  left_join(
    patients %>% select(PatientDurableKey, BirthDate),
    by = "PatientDurableKey"
  ) %>%
  mutate(AgeAtTest = as.numeric(TestDate - BirthDate) / 365.25)

first_sti_test <- proc %>%
  filter(AgeAtTest >= 14) %>%
  group_by(PatientDurableKey) %>%
  summarise(
    First_STI_Test_Age = min(AgeAtTest, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  mutate(
    First_STI_Test_Age = ifelse(
      is.infinite(First_STI_Test_Age), NA_real_, First_STI_Test_Age
    )
  )

sti_test_before_14 <- proc %>%
  group_by(PatientDurableKey) %>%
  summarise(
    Had_STITest_Before_14 = any(AgeAtTest < 14, na.rm = TRUE),
    .groups = "drop"
  )

```

### Eligibility and follow up

```r
# === Eligibility and follow up ===
patient_summary <- full_data %>%
  group_by(PatientDurableKey) %>%
  summarise(
    BirthDate = first(BirthDate),
    ReliableSex = first(ReliableSex),
    FirstRace = first(FirstRace),
    Ethnicity = first(Ethnicity),
    Svi2020 = first(Svi2020),
    Had_MH_Before_14 = any(MH == 1 & AgeAtDx < 14, na.rm = TRUE),
    First_MH_Age = min(AgeAtDx[MH == 1 & AgeAtDx >= 14], na.rm = TRUE),
    Last_Observed_Date = max(DxDate, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  left_join(first_sti_test, by = "PatientDurableKey") %>%
  left_join(sti_test_before_14, by = "PatientDurableKey") %>%
  mutate(
    First_MH_Age = ifelse(is.infinite(First_MH_Age), NA_real_, First_MH_Age),
    Had_STITest_Before_14 = coalesce(Had_STITest_Before_14, FALSE),
    # Preserve the censoring calculation used in the final analysis.
    Censor_Date = pmax(Last_Observed_Date, study_end_date, na.rm = TRUE),
    Censor_Age = as.numeric(Censor_Date - BirthDate) / 365.25,
    Eligible = !Had_STITest_Before_14 & !Had_MH_Before_14
  ) %>%
  filter(Eligible)

```

### Clinical visits and complete cases

```r
# === Clinical visits and complete cases ===
visit_birth_dates <- patients %>%
  select(PatientDurableKey, BirthDate) %>%
  distinct() %>%
  semi_join(patient_summary, by = "PatientDurableKey")
stopifnot(!anyDuplicated(visit_birth_dates$PatientDurableKey))
stopifnot(!anyNA(visit_birth_dates$BirthDate))

visits <- util_visits %>%
  transmute(
    PatientDurableKey = as.character(PatientDurableKey),
    EncounterKey = as.character(EncounterKey),
    VisitDate = as.Date(VisitDate)
  ) %>%
  semi_join(patient_summary, by = "PatientDurableKey") %>%
  distinct()
stopifnot(!anyNA(visits$EncounterKey), !anyNA(visits$VisitDate))
stopifnot(all(nzchar(visits$EncounterKey)))
stopifnot(!anyDuplicated(visits[c("PatientDurableKey", "EncounterKey")]))

visits <- visits %>%
  left_join(visit_birth_dates, by = "PatientDurableKey") %>%
  mutate(VisitAge = as.numeric(VisitDate - BirthDate) / 365.25)
stopifnot(!anyNA(visits$VisitAge), all(visits$VisitAge >= 0))
visits <- visits %>% filter(VisitAge >= 12)

baseline_visits <- visits %>%
  filter(VisitAge < 14) %>%
  count(PatientDurableKey, name = "Visits12_14")

people <- patient_summary %>%
  left_join(baseline_visits, by = "PatientDurableKey") %>%
  mutate(Visits12_14 = coalesce(Visits12_14, 0L)) %>%
  filter(complete.cases(FirstRace, Ethnicity, Svi2020)) %>%
  mutate(
    FirstRace = relevel(factor(FirstRace), ref = "White"),
    Ethnicity = relevel(factor(Ethnicity), ref = "Not Hispanic or Latino")
  )
visits <- visits %>% semi_join(people, by = "PatientDurableKey")

```

### STI testing to mental health intervals

```r
# === STI testing to mental health intervals ===
forward <- people %>%
  mutate(
    Event = as.integer(!is.na(First_MH_Age)),
    End_Age = ifelse(Event == 1, First_MH_Age, Censor_Age)
  )

forward_unexposed <- forward %>%
  filter(is.na(First_STI_Test_Age) | First_STI_Test_Age >= End_Age) %>%
  mutate(Start_Age = 14, Stop_Age = End_Age, STITest_TV = FALSE)

forward_before_test <- forward %>%
  filter(!is.na(First_STI_Test_Age), First_STI_Test_Age < End_Age) %>%
  mutate(
    Start_Age = 14, Stop_Age = First_STI_Test_Age,
    STITest_TV = FALSE, Event = 0L
  )

forward_after_test <- forward %>%
  filter(!is.na(First_STI_Test_Age), First_STI_Test_Age < End_Age) %>%
  mutate(
    Start_Age = First_STI_Test_Age, Stop_Age = End_Age,
    STITest_TV = TRUE
  )

forward_data <- bind_rows(
  forward_unexposed, forward_before_test, forward_after_test
) %>%
  filter(Stop_Age > Start_Age) %>%
  mutate(
    ExposureState = factor(
      paste(ifelse(STITest_TV, "Exposed", "Unexposed"), ReliableSex, sep = "_"),
      levels = c(
        "Exposed_Male", "Unexposed_Male", "Unexposed_Female", "Exposed_Female"
      )
    )
  ) %>%
  arrange(PatientDurableKey, Start_Age)

```

### Mental health to STI testing intervals

```r
# === Mental health to STI testing intervals ===
reverse <- people %>%
  mutate(
    Event = as.integer(!is.na(First_STI_Test_Age)),
    End_Age = ifelse(Event == 1, First_STI_Test_Age, Censor_Age)
  )

reverse_unexposed <- reverse %>%
  filter(is.na(First_MH_Age) | First_MH_Age >= End_Age) %>%
  mutate(Start_Age = 14, Stop_Age = End_Age, MH_TV = FALSE)

reverse_before_mh <- reverse %>%
  filter(!is.na(First_MH_Age), First_MH_Age < End_Age) %>%
  mutate(
    Start_Age = 14, Stop_Age = First_MH_Age,
    MH_TV = FALSE, Event = 0L
  )

reverse_after_mh <- reverse %>%
  filter(!is.na(First_MH_Age), First_MH_Age < End_Age) %>%
  mutate(Start_Age = First_MH_Age, Stop_Age = End_Age, MH_TV = TRUE)

reverse_data <- bind_rows(
  reverse_unexposed, reverse_before_mh, reverse_after_mh
) %>%
  filter(Stop_Age > Start_Age) %>%
  mutate(
    ExposureState = factor(
      paste(ifelse(MH_TV, "Exposed", "Unexposed"), ReliableSex, sep = "_"),
      levels = c(
        "Exposed_Male", "Unexposed_Male", "Unexposed_Female", "Exposed_Female"
      )
    )
  ) %>%
  arrange(PatientDurableKey, Start_Age)

```

### Final primary models

```r
# === Final primary models ===
# Visits enter as log(1 + baseline count), not a raw count.
# References: exposed males, White, and Not Hispanic or Latino.
forward_fit <- coxph(
  Surv(Start_Age, Stop_Age, Event) ~
    ExposureState + FirstRace + Ethnicity + Svi2020 +
    log1p(Visits12_14) + cluster(PatientDurableKey),
  data = as.data.frame(forward_data),
  ties = "efron", x = TRUE, model = TRUE
)

reverse_fit <- coxph(
  Surv(Start_Age, Stop_Age, Event) ~
    ExposureState + FirstRace + Ethnicity + Svi2020 +
    log1p(Visits12_14) + cluster(PatientDurableKey),
  data = as.data.frame(reverse_data),
  ties = "efron", x = TRUE, model = TRUE
)

summary(forward_fit)
summary(reverse_fit)
cox.zph(forward_fit)
cox.zph(reverse_fit)

# Check against the saved final analytic totals.
stopifnot(n_distinct(forward_data$PatientDurableKey) == 149640L)
stopifnot(n_distinct(reverse_data$PatientDurableKey) == 149640L)
stopifnot(sum(forward_data$Event) == 59312L)
stopifnot(sum(reverse_data$Event) == 37680L)

```

### Hazard ratios and within sex comparisons

```r
# === Hazard ratios and within sex comparisons ===
model_hazard_ratios <- function(fit, direction) {
  ci <- summary(fit)$conf.int
  tibble(
    Direction = direction,
    Term = rownames(ci),
    HR = ci[, "exp(coef)"],
    Lower95 = ci[, "lower .95"],
    Upper95 = ci[, "upper .95"]
  )
}

within_sex_hazard_ratios <- function(fit, direction) {
  b <- coef(fit)
  V <- vcov(fit)  # Patient-clustered covariance from the fitted model.
  bind_rows(lapply(c("Female", "Male"), function(sex) {
    L <- setNames(numeric(length(b)), names(b))
    L[paste0("ExposureStateUnexposed_", sex)] <- -1
    if (sex == "Female") L["ExposureStateExposed_Female"] <- 1
    used <- which(L != 0)
    estimate <- sum(L[used] * b[used])
    se <- sqrt(drop(t(L[used]) %*% V[used, used, drop = FALSE] %*% L[used]))
    tibble(
      Direction = direction, ReliableSex = sex,
      Comparison = "Exposed versus unexposed within sex",
      HR = exp(estimate),
      Lower95 = exp(estimate - 1.96 * se),
      Upper95 = exp(estimate + 1.96 * se)
    )
  }))
}

forest_values <- bind_rows(
  model_hazard_ratios(forward_fit, "STI_to_MH"),
  model_hazard_ratios(reverse_fit, "MH_to_STI")
)
within_sex_values <- bind_rows(
  within_sex_hazard_ratios(forward_fit, "STI_to_MH"),
  within_sex_hazard_ratios(reverse_fit, "MH_to_STI")
)
write.csv(forest_values, file.path(output_dir, "primary_HRs.csv"), row.names = FALSE)
write.csv(within_sex_values, file.path(output_dir, "within_sex_HRs.csv"), row.names = FALSE)

```

### Table 1

```r
# === Table 1 ===
# Testing summaries use all testing dates age 14 through the study cutoff.
# Survival models use the first outcome and first exposure date only.
table_proc <- proc %>%
  semi_join(people, by = "PatientDurableKey") %>%
  filter(AgeAtTest >= 14, TestDate <= study_end_date)

testing_days <- table_proc %>%
  distinct(PatientDurableKey, TestDate) %>%
  count(PatientDurableKey, name = "TestingDays")

followup_visits <- visits %>%
  filter(VisitAge >= 14, VisitDate <= study_end_date) %>%
  count(PatientDurableKey, name = "Visits14_cutoff")

people_table1 <- people %>%
  left_join(testing_days, by = "PatientDurableKey") %>%
  left_join(followup_visits, by = "PatientDurableKey") %>%
  mutate(
    TestingDays = coalesce(TestingDays, 0L),
    Visits14_cutoff = coalesce(Visits14_cutoff, 0L),
    Tested = TestingDays > 0L
  )

cohort_summary <- people_table1 %>%
  group_by(ReliableSex) %>%
  summarise(
    Patients = n(),
    Patients_tested = sum(Tested),
    Percent_tested = 100 * mean(Tested),
    Testing_dates = sum(TestingDays),
    Baseline_visits = sum(Visits12_14),
    Baseline_visits_Q1 = quantile(Visits12_14, 0.25),
    Baseline_visits_median = median(Visits12_14),
    Baseline_visits_Q3 = quantile(Visits12_14, 0.75),
    Followup_visits = sum(Visits14_cutoff),
    Followup_visits_median = median(Visits14_cutoff),
    SVI_Q1 = quantile(Svi2020, 0.25),
    SVI_median = median(Svi2020),
    SVI_Q3 = quantile(Svi2020, 0.75),
    .groups = "drop"
  )
race_counts <- people %>% count(ReliableSex, FirstRace)
ethnicity_counts <- people %>% count(ReliableSex, Ethnicity)

followup_summary <- function(intervals, direction) {
  intervals %>%
    group_by(PatientDurableKey, ReliableSex) %>%
    summarise(
      Years = sum(Stop_Age - Start_Age),
      Events = sum(Event), .groups = "drop"
    ) %>%
    group_by(ReliableSex) %>%
    summarise(
      Direction = direction, Patients = n(), Events = sum(Events),
      Followup_Q1 = quantile(Years, 0.25),
      Followup_median = median(Years),
      Followup_Q3 = quantile(Years, 0.75),
      .groups = "drop"
    )
}
followup_table <- bind_rows(
  followup_summary(forward_data, "STI_to_MH"),
  followup_summary(reverse_data, "MH_to_STI")
)
write.csv(cohort_summary, file.path(output_dir, "cohort_summary.csv"), row.names = FALSE)
write.csv(race_counts, file.path(output_dir, "race_counts.csv"), row.names = FALSE)
write.csv(ethnicity_counts, file.path(output_dir, "ethnicity_counts.csv"), row.names = FALSE)
write.csv(followup_table, file.path(output_dir, "followup_summary.csv"), row.names = FALSE)

```

### Primary forest plots

```r
# === Primary forest plots ===
reference_rows <- tibble(
  Term = c("ExposureStateExposed_Male", "FirstRaceWhite", "EthnicityNot Hispanic or Latino"),
  HR = 1, Lower95 = 1, Upper95 = 1
)
forest_plot_data <- bind_rows(lapply(c("STI_to_MH", "MH_to_STI"), function(direction) {
  bind_rows(
    forest_values %>% filter(Direction == direction),
    reference_rows %>% mutate(Direction = direction)
  )
})) %>%
  mutate(
    Label = case_when(
      grepl("^ExposureState", Term) ~ sub("^ExposureState", "", Term),
      grepl("^FirstRace", Term) ~ sub("^FirstRace", "", Term),
      grepl("^Ethnicity", Term) ~ sub("^Ethnicity", "", Term),
      Term == "log1p(Visits12_14)" ~ "Visits ages 12 to <14 log1p count",
      TRUE ~ Term
    ),
    Label = gsub("_", " ", Label),
    Direction = factor(
      Direction, levels = c("STI_to_MH", "MH_to_STI"),
      labels = c("STI testing to MH diagnosis", "MH diagnosis to STI testing")
    )
  )

primary_forest <- ggplot(forest_plot_data, aes(HR, Label)) +
  geom_vline(xintercept = 1, linetype = 2, color = "grey50") +
  geom_segment(aes(x = Lower95, xend = Upper95, yend = Label)) +
  geom_point(size = 2) +
  scale_x_log10() +
  facet_wrap(~ Direction) +
  labs(x = "Adjusted hazard ratio with 95% CI", y = NULL) +
  theme_minimal(base_size = 11)
ggsave(
  file.path(output_dir, "primary_forest.png"), primary_forest,
  width = 12, height = 8, dpi = 300, device = grDevices::png, type = "cairo"
)

```

### Age specific analyses in both directions

```r
# === Age specific analyses in both directions ===
# Same age-band method and reference profile as the final age-figure code.
# Restrict these figures to true one-year bands from 14-15 through 20-21.
make_age_figures <- function(intervals, exposure, direction, title) {
  age_data <- intervals %>%
    mutate(
      Event = ifelse(Stop_Age > 21, 0L, Event),
      Stop_Age = pmin(Stop_Age, 21)
    ) %>%
    filter(Start_Age < Stop_Age)

  age_intervals <- survSplit(
    Surv(Start_Age, Stop_Age, Event) ~ .,
    data = as.data.frame(age_data), cut = 15:20, episode = "AgeEpisode"
  ) %>%
    mutate(AgeBand = floor(Start_Age + 1e-8))

  probabilities <- list()
  hazards <- list()
  counts <- list()
  for (sex in c("Female", "Male")) {
    for (age in 14:20) {
      band_data <- age_intervals %>%
        filter(ReliableSex == sex, AgeBand == age)

      model_formula <- as.formula(paste(
        "Surv(Start_Age, Stop_Age, Event) ~", exposure,
        "+ FirstRace + Ethnicity + Svi2020 + log1p(Visits12_14)",
        "+ cluster(PatientDurableKey)"
      ))
      band_fit <- coxph(
        model_formula, data = as.data.frame(band_data),
        ties = "efron", model = TRUE, x = TRUE
      )

      reference <- data.frame(
        FirstRace = factor(rep("White", 2), levels = levels(band_data$FirstRace)),
        Ethnicity = factor(
          rep("Not Hispanic or Latino", 2), levels = levels(band_data$Ethnicity)
        ),
        Svi2020 = rep(median(people$Svi2020), 2),
        Visits12_14 = rep(6L, 2)
      )
      reference[[exposure]] <- c(FALSE, TRUE)
      stopifnot(!anyNA(reference))
      curves <- survfit(band_fit, newdata = reference, conf.int = 0.95)
      at_band_end <- summary(curves, times = age + 1, extend = TRUE)

      probabilities[[length(probabilities) + 1]] <- tibble(
        Direction = direction, ReliableSex = sex, AgeBand = age,
        Exposed = c(FALSE, TRUE),
        Probability = as.numeric(1 - at_band_end$surv),
        Lower95 = as.numeric(1 - at_band_end$upper),
        Upper95 = as.numeric(1 - at_band_end$lower)
      )

      coefficient <- paste0(exposure, "TRUE")
      estimate <- coef(band_fit)[coefficient]
      se <- sqrt(vcov(band_fit)[coefficient, coefficient])
      stopifnot(is.finite(estimate), is.finite(se))
      hazards[[length(hazards) + 1]] <- tibble(
        Direction = direction, ReliableSex = sex, AgeBand = age,
        HR = exp(estimate),
        Lower95 = exp(estimate - 1.96 * se),
        Upper95 = exp(estimate + 1.96 * se)
      )
      counts[[length(counts) + 1]] <- band_data %>%
        summarise(
          Direction = direction, ReliableSex = sex, AgeBand = age,
          Patients = n_distinct(PatientDurableKey), Events = sum(Event)
        )
    }
  }

  probabilities <- bind_rows(probabilities) %>%
    mutate(
      ReliableSex = factor(ReliableSex, levels = c("Female", "Male")),
      Status = factor(
        ifelse(Exposed, "Prior exposure", "No prior exposure"),
        levels = c("No prior exposure", "Prior exposure")
      )
    )
  hazards <- bind_rows(hazards) %>%
    mutate(ReliableSex = factor(ReliableSex, levels = c("Female", "Male")))
  counts <- bind_rows(counts)

  status_labels <- if (direction == "MH_to_STI") {
    c("No prior MH diagnosis", "Prior MH diagnosis")
  } else {
    c("No prior STI testing", "Prior STI testing")
  }
  caption <- paste0(
    "Adjusted for race, ethnicity, SVI and visits at ages 12 to <14.\n",
    "Reference profile: White, non-Hispanic, SVI ",
    round(median(people$Svi2020), 4), ", 6 baseline visits."
  )

  probability_plot <- ggplot(
    probabilities,
    aes(AgeBand, Probability, color = Status, fill = Status, group = Status)
  ) +
    geom_ribbon(aes(ymin = Lower95, ymax = Upper95), alpha = 0.15, color = NA) +
    geom_line(linewidth = 1) +
    geom_point(size = 2.8) +
    geom_errorbar(aes(ymin = Lower95, ymax = Upper95), width = 0.12, alpha = 0.55) +
    facet_wrap(~ ReliableSex) +
    scale_color_manual(values = c("#328DA8", "#E94150"), labels = status_labels) +
    scale_fill_manual(values = c("#328DA8", "#E94150"), labels = status_labels) +
    scale_x_continuous(breaks = 14:20) +
    scale_y_continuous(labels = scales::label_percent(), limits = c(0, NA)) +
    labs(
      title = title,
      subtitle = "Model-estimated probability in each one-year age band with 95% CI",
      x = "Age at start of band", y = "Probability of first event",
      color = NULL, fill = NULL, caption = caption
    ) +
    theme_minimal(base_size = 13) +
    theme(legend.position = "bottom", plot.caption = element_text(hjust = 0))

  hazard_plot <- ggplot(hazards, aes(AgeBand, HR)) +
    geom_hline(yintercept = 1, linetype = 2, color = "grey45") +
    geom_ribbon(aes(ymin = Lower95, ymax = Upper95), fill = "#328DA8", alpha = 0.15) +
    geom_line(color = "#328DA8", linewidth = 1) +
    geom_point(color = "#328DA8", size = 2.8) +
    geom_errorbar(aes(ymin = Lower95, ymax = Upper95), width = 0.12) +
    facet_wrap(~ ReliableSex) +
    scale_x_continuous(breaks = 14:20) +
    labs(
      title = paste(title, "age-specific hazard ratios"),
      subtitle = paste(status_labels[2], "versus", status_labels[1], "within sex"),
      x = "Age at start of band", y = "Adjusted hazard ratio", caption = caption
    ) +
    theme_minimal(base_size = 13) +
    theme(plot.caption = element_text(hjust = 0))

  for (plot_name in c("probability", "hazard_ratio")) {
    plot <- if (plot_name == "probability") probability_plot else hazard_plot
    stem <- file.path(output_dir, paste0(direction, "_", plot_name))
    ggsave(paste0(stem, ".png"), plot, width = 10, height = 6.5, dpi = 300, device = grDevices::png, type = "cairo")
    ggsave(paste0(stem, ".pdf"), plot, width = 10, height = 6.5)
  }
  write.csv(
    probabilities, file.path(output_dir, paste0(direction, "_probabilities.csv")),
    row.names = FALSE
  )
  write.csv(
    hazards, file.path(output_dir, paste0(direction, "_age_HRs.csv")),
    row.names = FALSE
  )
  write.csv(
    counts, file.path(output_dir, paste0(direction, "_band_counts.csv")),
    row.names = FALSE
  )
  list(probabilities = probabilities, hazards = hazards, counts = counts)
}

forward_age <- make_age_figures(
  forward_data, "STITest_TV", "STI_to_MH", "First MH diagnosis by age band"
)
reverse_age <- make_age_figures(
  reverse_data, "MH_TV", "MH_to_STI", "First STI testing by age band"
)
```

## Verification

The R analysis was rerun on the server on October 9, 2026, using the saved cohort and encounter extract. Both directions included 149,640 patients; the forward analysis had 59,312 first mental health events and the reverse analysis had 37,680 first STI testing events. Primary forest estimates agreed at manuscript precision, and the age-specific results reproduced the prior outputs. The tested Cairo PNG export fix is included.

One manuscript rounding correction remains: the lower 95% confidence bound for the reverse within-female hazard ratio rounds to **1.626**, rather than 1.627.
