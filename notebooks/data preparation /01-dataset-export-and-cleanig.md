# Notebook 1: Dataset Export and Cleaning

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

### Survey Data

### Person Data

### Measurement Data

## Creating One AoU Dataset

## Exporting and cleaning NHANES Dataset
