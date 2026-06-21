# Notebook 1: Dataset Export and Cleaning for the AoU Dataset

Here is the code I used to set up the Google Storage functions:

```r
write_dataframe_to_google_storage <- function(df, destination_filename){
  # Get the bucket name automatically
  bucket <- Sys.getenv('WORKSPACE_BUCKET')

  # Create a temporary local file
  tmp_file <- destination_filename
  write_csv(df, tmp_file)

  # Copy the local file to the bucket
  system(paste0("gsutil cp ./", tmp_file, " ", bucket, "/", destination_filename),

  # Clean up the local temporary file
  system(paste0("rm ./", tmp_file))

  Print(paste0("File saved to bucket: ", bucket, "/", destination_filename))
}
```

## Exporting the AoU Data

**NOTE:** Some of the automated queries were cut off in the workspace export PDF

### Person Bucket
```r
library(tidyverse)
library(bigrquery)

dataset_51646978_person_sql <- paste("
  SELECT
    person.person_id,
    person.birth_datetime as date_of_birth,
    p_sex_at_birth_concept.concept_name as sex_at_birth
  FROM
    `person` person
  LEFT JOIN
    `concept` p_sex_at_birth_concept
      ON person.sex_at_birth_concept_id = p_sex_at_birth_concept.concept_id
  WHERE
    person.PERSON_ID IN (SELECT
      distinct person_id
    FROM
      `cb_search_person` cb_search_person
    WHERE
      cb_search_person.person_id IN (SELECT
        person_id
      FROM
        `cb_search_person` p
      WHERE
        DATE_DIFF(CURRENT_DATE, dob, YEAR) - IF(EXTRACT(MONTH FROM dob)*100... [truncated]
        AND NOT EXISTS (    SELECT
              `x`
        FROM
          'death` d
        WHERE
          d.person_id = p.person_id ) )
      AND cb_search_person.person_id IN (SELECT
        person_id
      FROM
        `cb_search_person` p
      WHERE
        has_ehr_data = 1 )
      AND cb_search_person.person_id IN (SELECT
        criteria.person_id
      FROM
        (SELECT
          DISTINCT person_id, entry_date, concept_id
        FROM
          `cb_search_all_events`
        WHERE
          (concept_id IN (903118, 903115)
          AND is_standard = 0
          OR concept_id IN (903133, 903124, 903121)
          AND is_standard = 0 )) criteria )
      AND cb_search_person.person_id IN (SELECT
        person_id
      FROM
        `cb_search_person` p
      WHERE
        has_fitbit_activity_summary = 1 ) )", sep= "")

# Cloud Storage Destination path for the data exported from BigQuery
person_51646978_path <- file.path(
  Sys.getenv("WORKSPACE_BUCKET"),
  "bq_exports",
  Sys.getenv(OWNER_EMAIL"),
  strftime(lubridate::now(), "%Y%m%d"),
  "person_51646978"
  "person_51646978_*.csv")
message(str_glue('The data will be written to {person_51646978_path}. USE this path... [truncated]
                  'the data into your notebooks in the future.'))

# Query Export the dataset to Cloud Storage as csv files.
bq_table_save(
  bq_dataset_query(Sys.getenv("WORKSPACE_CDR"), dataset_51646978_person_sql, billin... [truncated]
  person_51646978_path,
  destination_format = "CSV")

# Read the data directly from Cloud Storage into memory
read_bq_export_from_workspace_bucket <- function(export_path) {
  col_types <- cols(sex_at_birth = col_character())
  bind_rows(
    map(system2('gsutil', args = c('ls', export_path), stdout = TRUE, stderr = TRUE),
    function(csv) {
      message(str_glue('Loading {csv}.'))
      chunk <- read_csv(pipe(str_glue('gsutil cat {csv}')), col_types = col_types)
      if (is.null(col_types)) {
        col_types <- spec(chunk)
      }
      chunk
    })
  )
}
dataset_person <- read_bq_export_from_workspace_bucket(person_51646978_path)
```

### Survey Bucket

### FITBIT Bucket
```r
my_ds <- Sys.getenv("WORKSPACE_CDR")

# Master Activity Query
sql_query <- paste0("
WITH daily_hr AS (
  SELECT person_id, date, SUM(minute_in_zone) as total_wear_minutes
  FROM `", my_ds, ".heart_rate_summary`
  WHERE minute_in_zone > 0
  GROUP BY person_id, date
  HAVING total_wear_minutes >= 600
),
daily_steps AS (
  SELECT person_id, DATE(datetime) as date, SUM(steps) as sum_steps
  FROM `", my_ds, ".steps_intraday`
  GROUP BY person_id, date
  HAVING sum_steps > 0 AND sum_steps < 100000
),
device_info AS (
  SELECT person_id, device_version
  FROM (
    SELECT person_id, device_version,
            ROW_NUMBER() OVER(PARTITION BY person_id ORDER BY last_sync_time DESC) as rn
    FROM `", my_ds, ".device`
    WHERE device_version IS NOT NULL
  ) WHERE rn = 1
)
SELECT
  act.person_id,
  act.date,
  act.activity_calories,
  act.sedentary_minutes,
  act.very_active_minutes,
  act.fairly_active_minutes,
  (act.very_active_minutes + act.fairly_active_minutes) AS total_active_minutes,
  s.sum_steps,
  d.device_version
FROM `", my_ds, ".activity_summary` act
INNER JOIN daily_steps s ON act.person_id = s.person_id AND act.date = s.date
INNER JOIN daily_hr hr ON act.person_id = hr.person_id AND act.date = hr.date
LEFT JOIN device_info d ON act.person_id = d.person_id
WHERE act.date BETWEEN '2021-08-01' AND '2023-08-31'
AND act.activity_calories > 0
AND (act.very_active_minutes + act.fairly_active_minutes) > 0
")

project <- Sys.getenv("GOOGLE_PROJECT")
daily_data_cleaned <- bq_project_query(project, sql_query) %>%
  bq_table_download()
```

### Measurement Bucket

## Cleaning the data

### Fitbit Data
```r
# Apply physical activity caps & best practices
daily_data_cleaned_v2 <- daily_data_cleaned %>%
  filter(
    total_active_minutes > 0,
    (sedentary_minutes + total_active_minutes) <= 1440,
    total_active_minutes < 960  # 16-hour active cap
  )

# Compute Fitbit Averages per participant
average_fitbit <- daily_data_cleaned_v2 %>%
  group_by(person_id) %>%
  mutate(days_count = n()) %>%
  filter(days_count >= 7) %>%  # Keep people with at least 7 valid days
  filter(date >= "2021-08-01" & date <= "2023-08-31") %>%
  filter(sum_steps > 0 & activity_calories > 0) %>%
  filter(!is.na(device_version)) %>%
  summarize(
    fitbit_start_date = min(date, na.rm = TRUE),
    avg_activity_calories = mean(activity_calories, na.rm = TRUE),
    avg_steps = mean(sum_steps, na.rm = TRUE),
    avg_sedentary_minutes = mean(sedentary_minutes, na.rm = TRUE),
    avg_active_minutes = mean(total_active_minutes, na.rm = TRUE)
  )

# Outlier removal
average_fitbit2 <- average_fitbit %>% 
  filter(
    avg_activity_calories >= 50 & avg_activity_calories <= 3000,
    avg_steps >= 1000 & avg_steps <= 50000,
    avg_sedentary_minutes <= 1140,
    avg_active_minutes > 0
  )
```

### Survey Data
```r
library(lubridate)
library(tidyr)

# Filter survey date ranges
survey_df <- dataset_survey %>%
  mutate(survey_date = as.Date(survey_datetime)) %>%
  filter(survey_date >= "2021-08-01" & survey_date <= "2023-08-31")

# Pivot from long to wide format
survey_wide <- survey_df %>%
  group_by(person_id, question) %>%
  slice_max(order_by = survey_date, n = 1, with_ties = FALSE) %>% # Keep latest response
  ungroup() %>%
  select(-survey_datetime) %>%
  pivot_wider(
    names_from = question,
    values_from = answer
  )

# Clean up long string survey column names
survey_wide <- survey_wide %>%
  rename(
    alcohol = `Alcohol: Alcohol Participant`,
    education_level = `Education Level: Highest Grade`,
    marital_status = `Marital Status: Current Marital Status`,
    race = `Race: What Race Ethnicity`,
    smoking = `Smoking: 100 Cigs Lifetime`,
    birthplace = `The Basics: Birthplace`
  )
```

### Person Data
```r
dataset_person = dataset_person %>%
  mutate(birthdate = as.Date(date_of_birth)) %>%
  select(-date_of_birth) %>%
  relocate(birthdate, .before = sex_at_birth)
```

### Measurement Data
```r
# Process clinical measurements
measurement_df <- dataset_measurement %>%
  mutate(measurement_date = as.Date(measurement_datetime)) %>%
  select(-measurement_datetime) %>%
  filter(measurement_date >= "2021-08-01" & measurement_date <= "2023-08-31")

# Aggregate measurements per person and pivot wide
measurement_wide <- measurement_df %>%
  group_by(person_id, standard_concept_name) %>%
  summarize(
    avg_value = mean(value_as_number, na.rm = TRUE),
    .groups = "drop"
  ) %>%
  pivot_wider(
    names_from = standard_concept_name,
    values_from = avg_value
  )

# Clean clinical column headers
measurement_wide <- measurement_wide %>%
  rename(
    height = `Body height`,
    BMI = `Body mass index (BMI) [Ratio]`,
    weight = `Body weight`,
    total_cholesterol = `Cholesterol [Mass/volume] in Serum or Plasma`,
    diastolic_bp = `Diastolic blood pressure`,
    HbA1c = `Hemoglobin A1c/Hemoglobin.total in Blood`,
    systolic_bp = `Systolic blood pressure`,
    heart_rate = `Heart rate`,
    cholesterol_hdl = `Cholesterol in HDL [Mass/volume] in Serum or Plasma`
  )
```

## AoU Dataset Merge
```r
# Left join all domains together
AoU_combined <- dataset_person %>%
  left_join(survey_wide, by = "person_id") %>%
  left_join(measurement_wide, by = "person_id") %>%
  left_join(average_fitbit2, by = "person_id")

# Aggregate into final analytics dataset (1 row per participant)
AoU <- AoU_combined %>%
  group_by(person_id) %>%
  summarize(
    birthdate = first(na.omit(birthdate)),
    sex_at_birth = first(na.omit(sex_at_birth)),
    survey_date = first(na.omit(survey_date)),
    alcohol = first(na.omit(alcohol)),
    education_level = first(na.omit(education_level)),
    marital_status = first(na.omit(marital_status)),
    race = first(na.omit(race)),
    smoking = first(na.omit(smoking)),
    birthplace = first(na.omit(birthplace)),
    
    # Clinical averages
    height = mean(height, na.rm = TRUE),
    BMI = mean(BMI, na.rm = TRUE),
    weight = mean(weight, na.rm = TRUE),
    total_cholesterol = mean(total_cholesterol, na.rm = TRUE),
    diastolic_bp = mean(diastolic_bp, na.rm = TRUE),
    HbA1c = mean(HbA1c, na.rm = TRUE),
    systolic_bp = mean(systolic_bp, na.rm = TRUE),
    heart_rate = mean(heart_rate, na.rm = TRUE),
    cholesterol_hdl = mean(cholesterol_hdl, na.rm = TRUE),
    
    # Fitbit variables
    fitbit_date = first(na.omit(fitbit_start_date)),
    avg_activity_calories = mean(avg_activity_calories, na.rm = TRUE),
    avg_steps = mean(avg_steps, na.rm = TRUE),
    avg_sedentary_minutes = mean(avg_sedentary_minutes, na.rm = TRUE),
    avg_active_minutes = mean(avg_active_minutes, na.rm = TRUE),
    .groups = 'drop'
  ) %>%
  drop_na(avg_steps, avg_activity_calories) # Enforce focus cohort filter

# Save to Google Cloud Bucket
destination_filename <- "AoU.csv"
write_dataframe_to_google_storage(AoU, destination_filename)
```
