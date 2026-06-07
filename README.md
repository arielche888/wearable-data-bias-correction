# Correcting Selection Bias in Wearable Data: A Mass Imputation Framework for U.S. Physical Activity Surveillance

## Project Overview
National health surveillance lacks objective, population-level benchmarks for specific movement metrics such as daily step counts and activity calories because surveys like the National Health and Nutrition Examination Survey (NHANES) do not collect wearable sensor data. While volunteer cohorts such as the All of Us (AoU) Research Program provide high-resolution Fitbit data, they are subject to significant selection bias. 

This study addresses this gap by implementing a mass imputation framework designed to correct selection bias and transport objective activity metrics from the AoU volunteer cohort ($n=15,313$) to a representative NHANES sample ($n=4,914$) to estimate the national mean physical activity levels.

---

## Key Research Objectives
1. **Quantify Selection Bias:** Evaluate the extent to which demographically adjusted national estimates for daily steps and calories differ from raw volunteer averages across age, race, and gender.
2. **Assess Model Complexity:** Compare non-linear machine learning models against traditional linear regression to identify stable consensus ranges for population-level movement and energy expenditure.
3. **Compare Frameworks:** Evaluate whether a Standard Mass Imputation (MI) framework or a refined Matched Mass Imputation (MMI) framework provides more biologically plausible results for distinct demographic groups. 

---

## Data Sources & Feature Harmonization
To ensure valid data integration and transportability, a shared feature space ($X$) was aligned and harmonized across both cohorts: 

* **Source Pool ($S_B$)**: All of Us Registered Tier Dataset v8 ($n=15,313$ adults providing active wrist-worn Fitbit data).
* **Representative Sample ($S_A$)**: NHANES August 2021–August 2023 cycle ($n=4,914$ adults).

### Harmonized Feature Space ($X$)
* **Age:** Continuous (Restricted to adults 18+).
* **Gender:** Binary (Recoded to Male/Female factor).
* **Height & Weight:** Standardized continuous metrics (cm and kg).
* **Behavioral Predictors:** Continuous active and sedentary minutes. 

---

## Methodological Framework
Two primary mass imputation integration techniques were compared:

1. **Standard Mass Imputation (MI):** Utilizes the full volunteer pool as the training source pool.
2. **Matched Mass Imputation (MMI):** Implements a 1:1 nearest-neighbor matching step via Euclidean distance to refine the training data to only the most demographically similar volunteer counterparts.

Ensemble predictions from five distinct modeling architectures (Linear Regression, XGBoost, Random Forest, Neural Networks, and Generalized Additive Models) were integrated into a consensus estimate. Statistical inference and 95% confidence intervals were derived using a robust dual-resampling bootstrap procedure over 100 replicates to account for cumulative sampling and algorithmic uncertainty.

---

## Project Structure & Reading Guide
The compiled PDF exports of the Jupyter Notebooks used throughout this analysis are organized chronologically below:

├── Project_Report.pdf                         # Final Written Capstone Document
└── notebooks/                                 # Organized Project Phases (PDF Exports)
    ├── 01_Data_Preparation/
    │   ├── 01_aou_preprocessing.pdf           # Filtering valid wear days and removing outliers
    │   ├── 02_nhanes_harmonization.pdf        # Survey weight adjustments and variable alignment
    ├── 02_Exploratory_Analysis/
    │   ├── 03_distributional_overlap.pdf      # Visualizing step/calorie distributions & matching balance
    │   ├── 04_demographic_audits.pdf          # Baseline volunteer vs. representative baseline checks
    └── 03_Statistical_Modeling/
        ├── 05_standard_mi_pipeline.pdf        # Running the 5 ML baseline architectures under Standard MI
        ├── 06_matched_mmi_pipeline.pdf        # Executing 1:1 nearest-neighbor matching and model execution
        └── 07_dual_bootstrap_inference.pdf    # Running the 100-replicate dual-resampling uncertainty loop

---

## Major Research Findings
* **National Benchmarks:** The MMI framework tightened the machine learning consensus to precise national ranges of 8,236 to 8,263 daily steps and 1,170 to 1,173 activity calories after adjusting for the healthy-volunteer bias.
* **Reverse Selection Bias:** A striking "reverse bias" was discovered in the 65+ demographic. Corrected national averages exceeded the raw volunteer benchmarks, revealing that digital health volunteer cohorts systematically underrepresent active, community-dwelling seniors.
* **Metric Resilience & Compensatory Effect:** Selection bias impacted step counts more heavily than caloric expenditure. Imputation successfully decoupled movement from energy expenditure, capturing a nuanced biological profile where lower national step counts were offset by a higher average body mass.
* **Model Stability via Matching:** Restricting the source pool via demographic matching stabilized simpler models (bringing linear regression directly into the center of the ML consensus) and reduced ensemble variance from a 165-step spread to a tight 27-step range.

---

## Software & Environments
* **Environment:** R (v4.1) via Jupyter Notebooks on the All of Us Researcher Workbench.
* **Core Libraries:** tidyverse & nhanesA (Data Orchestration), FNN (Nearest-Neighbor Matching), xgboost, randomForest, mgcv, and nnet (Predictive Architecture Construction).  

