# New York City Housing Maintenance Violations: End-to-End Data Pipeline & Dashboards

**Author:** Jalil Cooper, Data Analytics Consultant
**Project Type:** Data School Capstone
**Completed:** October 2026
**Live Dashboards:** [NYC Housing Preservation and Development 2022 Maintenance Report (Tableau Public)](https://public.tableau.com/app/profile/jalil.cooper/viz/NYCHousingPreservationandDevelopment2022MaintenanceReport/ExecutiveDashboard)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [Technology Stack](#3-technology-stack)
4. [Data Sources](#4-data-sources)
5. [Data Ingestion](#5-data-ingestion)
6. [Data Modeling and Table Creation](#6-data-modeling-and-table-creation)
7. [Data Validation](#7-data-validation)
8. [Dashboards](#8-dashboards)
9. [Findings and Policy Recommendations](#9-findings-and-policy-recommendations)
10. [Known Limitations and Next Steps](#10-known-limitations-and-next-steps)

---

## 1. Project Overview

### 1.1 Purpose

This project delivers a fully functional data pipeline, from public data sources to a business intelligence (BI) layer, that supports evidence-based decisions about housing maintenance in New York City. Two dashboards were developed from the pipeline: an **Executive Dashboard** for high-level policy decisions and an **Analyst Dashboard** for building-level and neighborhood-level investigation.

### 1.2 Objectives

- Build an end-to-end data pipeline into a BI tool of choice (Tableau).
- Develop two dashboards: one Executive view and one Analyst view.
- Determine which boroughs suffer most in terms of housing violation **count** and **severity**.
- Identify housing developments for maintenance in an **equitable** manner, using neighborhood income and demographic context.
- Inform housing policy that mitigates landlord power and improves tenant agency.

### 1.3 Stakeholders

| Stakeholder | Role | Primary Use |
|---|---|---|
| New York City Department of Housing Preservation and Development (HPD) | Sponsoring agency | Overall housing maintenance strategy |
| Executive View Stakeholder: City-Level Politician | Executive decision-maker | Borough-level prioritization and policy direction |
| Housing Maintenance Associate | Analyst view stakeholder | Building-level and neighborhood-level investigation |

---

## 2. Repository Structure

```text
.
├── README.md
├── data/                 # Source extracts (or samples) as downloaded
├── sql/                  # Snowflake table creation and validation scripts
├── alteryx/              # Alteryx workflows
├── dashboards/           # Tableau packaged workbook
└── docs/
    ├── images/           # Screenshots used in this README
    └── presentation/     # Capstone slide deck
```

---

## 3. Technology Stack

| Layer | Tool |
|---|---|
| Source data | US Census Bureau American Community Survey (ACS) 5-Year API; NYC Open Data |
| Extraction and transformation | Alteryx Designer |
| Data warehouse | Snowflake |
| Data modeling and validation | SQL |
| Visualization | Tableau (published on Tableau Public) |

---

## 4. Data Sources

| Source | Description | Volume |
|---|---|---|
| **US Census American Community Survey (ACS) 5-Year Estimates (2017–2022)** | Five-year estimates based on extensive survey data collection from the American people and institutions. Provides neighborhood (census tract) demographics, income, education, and household characteristics. | 2,327 rows |
| **NYC Open Data: Housing Maintenance Code Violations** | Dataset containing every housing maintenance code violation recorded in New York City. | One-year extract (2022): approximately 857,000 rows at source. A random 1-in-100 sample (approximately 8,570 rows) was loaded into Snowflake (see [Section 5.2](#52-nyc-housing-maintenance-code-violations)). |

---

## 5. Data Ingestion

Data ingestion is the first stage of the pipeline and was built in Alteryx, with Snowflake as the destination warehouse.

### 5.1 Census ACS Demographic Data

The ACS workflow performs the following steps:

1. Reads the base URL, API endpoint, and API key from a configuration file.
2. Builds the full request URL for the ACS API.
3. Downloads the requested demographic measures (for example, male population) from the API.
4. Removes columns that are no longer needed (for example, the base URL).
5. Flattens the returned JSON array so that each row represents one county/tract combination with all requested measures.
6. Appends a unique ID to each county/tract pair so that rows are grouped accurately after pivoting.
7. Flattens a second array so that each measure within each tract becomes an individual row.
8. Runs a validation check confirming that the `DownloadData_idx` field contains only measure names and column headers.

<img width="752" height="266" alt="Screenshot 2026-10-30 142840" src="https://github.com/user-attachments/assets/3c54fa6a-ef17-47bf-9f25-b21a6ee4179d" />


### 5.2 NYC Housing Maintenance Code Violations

The violations workflow reads the NYC Open Data CSV export, applies a random 1-in-100 row sample, and writes the result to Snowflake.

**Sampling rationale:** The 2022 extract contains approximately 857,000 rows. Because of limited storage capacity in the Snowflake data warehouse, the dataset was reduced to a random 1-in-100 sample (approximately 8,570 rows) before loading. Random sampling was used so that the loaded data remains broadly representative of the full extract. All counts shown in the dashboards therefore reflect the sample and not the full population of violations.

<img width="545" height="182" alt="Screenshot 2026-10-30 143044" src="https://github.com/user-attachments/assets/8ebb3c0a-acab-45dc-b3a3-418f64910ac4" />


### 5.3 Connecting Alteryx to Snowflake

Alteryx connects to Snowflake through the Snowflake configuration dialog (data source, user, server, database, schema, and warehouse). **Credentials and API keys are not stored in this repository.**

### 5.4 Ingestion Challenges and Resolutions

| Challenge | Detail | Resolution |
|---|---|---|
| Lost workflow | The Alteryx flow was lost late in the working day, and all work done after lunch was lost. | Rebuilt the flow. Workflow files are now saved to the `alteryx/` folder and version-controlled. |
| cURL error 56 | The error "Failure when receiving data from the peer" indicates that the TCP or TLS connection was abruptly closed or reset by the remote server, a proxy, or a firewall before the full response was received. | Re-ran the affected API calls and validated that the complete response had been received before downstream processing. |

---

## 6. Data Modeling and Table Creation

### 6.1 Schema Design

The data was modeled in Snowflake as a dimensional schema with two fact tables and five dimension tables. Neighborhood-level Census data is linked to building-level violation data through a shared 11-digit **GEOID**.

| Table | Type | Description |
|---|---|---|
| `neighborhood_dim` | Dimension | Median income, rent-burdened households, vacant units, average household size, SNAP-receiving households, and tract, county, and state identifiers. Keyed by `GEOID`. |
| `demographics_dim` | Dimension | Median age, total, female, and male population, population by race/ethnicity, and educational attainment (high school and college graduates). Keyed by `GEOID`. |
| `building_dim` | Dimension | Street code, longitude, latitude, council district, and `GEOID`. Keyed by `building_id`. |
| `house_dim` | Dimension | House number and apartment, linked to `building_dim` through `building_id`. Keyed by `house_id`. |
| `violation_class_dim` | Dimension | Violation class (hazard severity). Keyed by `violation_class_id`. |
| `violation_notice_fact` | Fact | Notice of violation (NOV) issue date and description. Keyed by `nov_id`. |
| `violations_fact` | Fact | One row per violation, with inspection, approval, certification, and correction dates, NOV status, and a rent-impairing flag. Links to `house_dim`, `violation_class_dim`, and `violation_notice_fact`. |

<img width="1518" height="865" alt="Screenshot 2026-10-30 143628" src="https://github.com/user-attachments/assets/bf5c8318-c6bc-4768-894b-32849ed2b646" />


### 6.2 SQL Scripts

All table creation and validation scripts used in Snowflake are stored in the [`/sql`](./sql) folder. Examples include creating the house and building dimension tables (with primary and foreign key constraints) and deriving the `GEOID` used to join violations to demographics.

### 6.3 Table Creation Challenge: Census Tract Misalignment

The `GEOID` is critical to the model, and the two source datasets encode census tract numbers differently:

| Dataset | Tract "65" is represented as |
|---|---|
| American Community Survey | `65.00`, which becomes `6500` once the decimal is removed |
| NYC Open Data | `65` |

**Resolution:** Tract values were standardized in SQL using length checks and left-padding so that both datasets produce the same census tract code. The resulting `GEOID` is cast to a **string rather than an integer** so that all 11 digits (including leading zeros) are preserved.

---

## 7. Data Validation

Validation was performed in SQL after table creation and before any dashboard development.

- **Key integrity:** Primary and foreign key relationships were checked between the fact and dimension tables.
- **Join coverage:** `GEOID` matches between violations and demographic records were confirmed.
- **Source table code changes:** The 2024 ACS table codes were updated. Each of the 19 fields was reviewed one at a time to confirm that the correct measure was being ingested.

---

## 8. Dashboards

Two dashboards were built in Tableau on top of the Snowflake model. The published workbook is available on [Tableau Public](https://public.tableau.com/app/profile/jalil.cooper/viz/NYCHousingPreservationandDevelopment2022MaintenanceReport/ExecutiveDashboard).

### 8.1 Executive Dashboard

Designed for borough-level prioritization and policy decisions made at the highest level. The ideal user for this dashboard would be a city-level politician. It summarizes violation counts and severity by borough and tracks the time taken to correct violations.

<img width="1241" height="1012" alt="Screenshot 2026-11-10 161523" src="https://github.com/user-attachments/assets/a5512f80-58d3-4f1a-bfa2-247aefcdc1df" />


### 8.2 Analyst Dashboard

Designed for the Housing Maintenance Associate. A control panel allows filtering by borough, neighborhood, maximum income level, and violation status, and users can toggle between a map view and a statistical view.

**Map View:** Selecting a building on the map returns a profile of its neighborhood, including median income, median age, average household size, rent-burdened households, households receiving SNAP benefits, gender and racial breakdowns, and educational attainment.

<img width="1582" height="1008" alt="Screenshot 2026-11-10 162432" src="https://github.com/user-attachments/assets/6965ebab-d8eb-4a02-a897-1b974050afda" />


**Statistical View:** Plots average median income against number of violations per neighborhood (with a trend line, p < 0.05) for up to five selected boroughs, alongside a Top N ranking of the buildings with the most hazardous violations.

<img width="1582" height="896" alt="Screenshot 2026-11-10 162516" src="https://github.com/user-attachments/assets/dd131976-c7fd-4171-b011-5e9ee4e3e3ab" />


### 8.3 Dashboard Challenge

The demographic measures could have been pivoted into a long format for easier charting. This was deliberately avoided because pivoting would have multiplied the number of rows (an "explosion") in the dataset and slowed the dashboards.

---

## 9. Findings and Policy Recommendations

### 9.1 Maintenance Prioritization

- **Target the Bronx first for mass housing maintenance.** It is the second smallest borough by size yet has the most violations.
- **For equitable resource distribution, focus on neighborhoods where median income is below $50,000.** The Statistical View shows violation counts trending downward as neighborhood income rises.
- **Priority neighborhood example:** Allerton, in the Bronx (majority Black population).

### 9.2 Housing Policy

- Strictly enforce civil penalties on landlords who fail to correct violations, especially hazardous ones.
- **Expected outcome:** A decrease in the time taken to correct violations, as measured in the Executive Dashboard.

---

## 10. Known Limitations and Next Steps

### 10.1 Limitations

- The violations data covers one year (2022) only, and a random 1-in-100 sample of that year was loaded because of data warehouse storage limits. Dashboard counts are sample-based and are not the full number of violations recorded by the city. Loading the full dataset in a larger warehouse environment would allow exact counts.
- ACS figures are five-year estimates and carry a margin of error at the census tract level.
- The statistical relationship between income and violation counts is correlational and does not establish causation.

### 10.2 Next Steps

- Complete additional dashboard formatting.
- Clean up dashboard interactivity.
- Add data validation and visualization of "DZV" items (to be defined).

---

## Contact

Jalil Cooper, Data Analytics Consultant
[GitHub](https://github.com/) · [Tableau Public](https://public.tableau.com/app/profile/jalil.cooper)
