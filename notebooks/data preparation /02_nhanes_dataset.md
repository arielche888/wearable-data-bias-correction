# Notebook 1: Dataset Loading and Cleaning for the NHANES Dataset

This notebook handles loading, cleaning, and merging the comparative dataset from NHANES (National Health and Nutrition Examination Survey).

```r
library(tidyverse)
```

## Loading & Cleaning

### Demographics Data

```r
# Install and load the nhanesA package
install.packages("nhanesA")
library(nhanesA)

demo <- nhanes('DEMO_L', translated = FALSE)
demosub <- demo %>%
  select(SEQN, RIDAGEYR, RIDRETH3, RIAGENDR, DMDBORN4, DMDEDUC2, DMDMARTZ, DMQMILIZ)
```

### Health & Clinical Measurements

```r
# Load Body Measurements Data
bodymeasures <- nhanes('BMX_L', translated = FALSE)
bodymeasuressub <- bodymeasures %>%
  select(SEQN, BMXWT, BMXHT, BMXBMI)

# Load Blood Pressure Data
bp <- nhanes('BPXO_L', translated = FALSE)
bpsub <- bp %>%
  select(SEQN, BPXODI2, BPXOSY2, BPXOPLS2)

# Load Glycohemoglobin (HbA1c) Data
ghb <- nhanes('GHB_L', translated = FALSE)
ghbsub <- ghb %>%
  select(-WTPH2YR)

# Load Total Cholesterol Data
totalchol <- nhanes('TCHOL_L', translated = FALSE)
totalcholsub <- totalchol %>%
  select(SEQN, LBXTC)

# Load HDL Cholesterol Data
hdl <- nhanes('HDL_L', translated = FALSE)
hdlsub <- hdl %>%
  select(SEQN, LBDHDD)
```

### Behavioral Surveys
```r
# Load Smoking Behavior Survey
smoking <- nhanes('SMQ_L', translated = FALSE)
smokingsub <- smoking %>%
  select(SEQN, SMQ020)

# Load Alcohol Use Survey
alcohol <- nhanes('ALQ_L', translated = FALSE)
alcoholsub <- alcohol %>%
  select(SEQN, ALQ111)
```

### Activity Metrics
```r
# Load and process the Physical Activity Survey data
activity <- nhanes('PAQ_L') %>%
  select(
    SEQN,
    PAD680,   # Minutes of sedentary activity per day
    PAD790Q,  # Number of days of moderate activity
    PAD790U,  # Unit of time for moderate activity (Day/Week/Month/Year)
    PAD800,   # Minutes of moderate activity per day
    PAD810Q,  # Number of days of vigorous activity
    PAD810U,  # Unit of time for vigorous activity (Day/Week/Month/Year)
    PAD820    # Minutes of vigorous activity per day
  ) %>%
  # Standardize Moderate Activity to minutes per day based on unit time
  mutate(mod_min_day = case_when(
    PAD790U == "Day"   ~ PAD790Q * PAD800,
    PAD790U == "Week"  ~ (PAD790Q * PAD800) / 7,
    PAD790U == "Month" ~ (PAD790Q * PAD800) / 30.44,
    PAD790U == "Year"  ~ (PAD790Q * PAD800) / 365,
    TRUE               ~ NA
  )) %>%
  # Standardize Vigorous Activity to minutes per day based on unit time
  mutate(vig_min_day = case_when(
    PAD810U == "Day"   ~ PAD810Q * PAD820,
    PAD810U == "Week"  ~ (PAD810Q * PAD820) / 7,
    PAD810U == "Month" ~ (PAD810Q * PAD820) / 30.44,
    PAD810U == "Year"  ~ (PAD810Q * PAD820) / 365,
    TRUE               ~ NA
  )) %>%
  # Sum moderate and vigorous daily minutes to match the AoU MVPA concept
  mutate(
    active_minutes = rowSums(as.matrix(select(., mod_min_day, vig_min_day)), na.rm = TRUE)
  ) %>%
  select(SEQN, PAD680, active_minutes)
```

## NHANES Dataset Merging

```r
# Consolidate all independent sub-tables into a list
list_of_tables <- list(demosub, bodymeasuressub, bpsub, ghbsub, totalcholsub, hdlsub, smokingsub, alcoholsub, activity)

# Joint execution of full outer / left merges
nhanes <- list_of_tables %>%
  reduce(left_join, by = "SEQN")

# Filter for target subset criteria (Adults with valid Medical Examination Center weights)
nhanes <- nhanes %>%
  filter(RIDAGEYR >= 18) %>%
  filter(!is.na(WTMEC2YR) & WTMEC2YR > 0)

#saving the dataset
destination_filename2 <- "nhanes.csv"
write_dataframe_to_google_storage(nhanes_final, destination_filename2) 
```
