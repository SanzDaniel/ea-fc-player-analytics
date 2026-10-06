# EA FC Player Analytics — Databricks Free Edition & Power BI Pipeline

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD8?style=for-the-badge&logo=delta&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)

## Overview

This project implements a complete data engineering pipeline for EA Sports FC player data using **Databricks Free Edition, Delta Lake, Auto Loader, SCD1/SCD2 modeling, and Power BI**.

The project is designed as a portfolio project for **Databricks Data Engineer Associate certification preparation** and **Data Engineering interviews**.

---

## Why This Project

I created this project to demonstrate practical experience with:

- **Databricks** and **Delta Lake** for building a lakehouse architecture.
- **Auto Loader** for incremental, idempotent ingestion.
- **SCD1 and SCD2** for dimensional modeling.
- **Data Quality** validation as a first-class citizen.
- **Databricks Workflows** for pipeline orchestration.
- **Power BI** for business-facing visualization.

The end goal is a complete, reproducible pipeline that ingests raw CSV files, transforms them into a dimensional model, validates the result, and exposes an interactive dashboard.

---
## Project Architecture

The project follows the **Medallion Architecture** (Bronze → Silver → Gold-ready):


                 CSV Files in volumen fifa_landig
                     │
                     ▼
              ┌─────────────┐
              │ Auto Loader │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │    Bronze   │
              │    Layer    │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │    Silver   │
              │    Layer    │
              │             │
              │ SCD1 / SCD2 │
              │ Dimensions  │
              │ Bridges     │
              │ Facts       │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │ Validations │
              └──────┬──────┘
                     │
                     ▼
              ┌─────────────┐
              │   Power BI  │
              │  Dashboard  │
              └─────────────┘



---

## Tech Stack

| Layer | Technology |
|---|---|
| Ingestion | Databricks Auto Loader (`cloudFiles`) |
| Storage | Delta Lake, Unity Catalog Volumes |
| Transformation | PySpark, Spark SQL |
| Orchestration | Databricks Workflows (Jobs with DAG) |
| Data Quality | Custom PySpark validation notebook |
| Visualization | Power BI Desktop (OAuth connection) |
| Version Control | Git + GitHub |

---
### Design Decisions

**Why Unity Catalog Volumes instead of S3 or SharePoint?**
Databricks Free Edition only provides access to Unity Catalog Volumes. External storage (S3, ADLS, SharePoint) requires paid cloud resources. The pipeline uses `cleanSource = MOVE` to archive processed files after 1 day, keeping the landing folder small and reducing storage costs.

**Why Auto Loader instead of COPY INTO or CTAS?**
Auto Loader provides:
- Automatic detection of new files in the landing folder.
- Schema evolution (`addNewColumns`) without breaking the pipeline.
- Idempotency via checkpoint (no duplicate ingestion).
- Automatic metadata capture (`_metadata.file_name`, `_ingestion_timestamp`).
- Efficient scaling to millions of files.

See `01_ingest_bronze.py` for the full implementation.

**Why SCD1 for some dimensions and SCD2 for others?**
See the [Data Model](#data-model) section for the full breakdown.

---

## Repository Structure

ea-fc-player-analytics/

├── README.md

├── .gitignore

├── notebooks/

│ ├── bronze/

│    │ ├── 01_ingest_bronze.py

│ ├── silver/

│    │ ├── validation/

│       │ ├── validations_silver.py

│    │ ├── dim_edition.py

│    │ ├── dim_gender.py

│    │ ├── dim_position.py

│    │ ├── dim_nationality.py

│    │ ├── dim_league.py

│    │ ├── dim_club.py

│    │ ├── dim_playstyle.py

│    │ ├── dim_player.py

│    │ ├── bridge_player_position.py

│    │ ├── bridge_player_playstyle.py

│    │ ├── fact_player_snapshot.py

│    │ ├── fact_player_facets.py


├── images/

│ ├── pbix/

│    │ ├── overview.png

│    │ ├── players.png

│    │ ├── playstyles.png

│    │ ├── Model.png

│ ├── databricks/

│    │ ├── pipeline.png


└── powerbi/

└── EA_FC27_Player_Analytics.pbix

---

## Data Model

### What is "grain"?

The **grain** of a table is what a single row represents. For example, in `fact_player_snapshot`, one row represents **one player in one snapshot**. In `dim_edition`, one row represents **one game edition**.

### Table Grains

| Table | Grain | Type |
|---|---|---|
| `dim_edition` | `edition_id` | SCD1 |
| `dim_gender` | `gender_id` | SCD1 |
| `dim_position` | `position_id` | SCD1 |
| `dim_nationality` | `nationality_id` | SCD1 |
| `dim_league` | `league_code` | SCD2 |
| `dim_club` | `club_code + gender_id + league_id` | SCD2 |
| `dim_playstyle` | `playstyle_code + is_plus` | SCD2 |
| `dim_player` | `player_id + edition_id` | SCD2 |
| `bridge_player_position` | `player_snapshot_key + position_id + position_type` | Bridge |
| `bridge_player_playstyle` | `player_snapshot_key + playstyle_id` | Bridge |
| `fact_player_snapshot` | `player_snapshot_key` | Fact |
| `fact_player_facets` | `player_snapshot_key + faceta` | Fact |

### Key Design: `player_snapshot_key`

Both `dim_player` and `fact_player_snapshot` use a composite key:

```python
player_snapshot_key = concat(player_id, "-", snapshot_date)
```

This solves two problems:

1. Power BI relationships: player_id alone has duplicates across snapshots.

2. CD2 correctness: each player-version pair is uniquely identifiable.
---

## Validations

A dedicated notebook (validations_silver.py) runs 30+ checks on the Silver layer:

- Primary key uniqueness

- Foreign key integrity

- Valid ranges (1-99 for metrics)

- Completeness (no NULLs in critical metrics)

- Fact table grain uniqueness

- SCD2 consistency (End_date > Start_date, single Is_current)

- Multi-Edition Testing (FC 26 + FC 27)

---
## Multi-Edition Testing (FC 26 + FC 27)
### The pipeline was validated with two editions:


|Edition	|Source	|Snapshot|	Rows|
|---|---|---|---|
|FC 27|	EA Sports FC 27 Player Ratings (Kaggle)	|2026-09-12|	19,789
FC 26	|Simulated from FC 27 (modified edition, snapshot, ratings, and some clubs)	|2025-09-12	|10|

Why FC 26 was simulated: the public FC 26 dataset (SoFIFA scraping) has an incompatible schema. Instead of mapping 40+ columns, the FC 26 snapshot was simulated to test SCD2 with multiple editions.

## Power BI Dashboard
**Page 1: Overview**

- KPIs: Total Players, Average Rating, Max Rating, % with PlayStyle

- Top 10 Leagues by player count

- World map of player nationalities

- Rating distribution by position

**Page 2: Players**

- Top players table with club, league, position, nationality, rating

- Skills and Weak Foot cards (with empty state when no player is selected)

- Facets table showing the 6 card facets of the selected player(with empty state when no player is selected)

**Page 3: PlayStyles**

- Top 10 base PlayStyles

- Top 10 PlayStyles+

- Players with most PlayStyles

- Base vs Plus distribution



# How to Reproduce
## 1. Create the Catalog and Schemas

You can do this either via SQL commands or through the Databricks UI.

```sql
CREATE CATALOG IF NOT EXISTS fifa;

CREATE SCHEMA IF NOT EXISTS fifa.bronze;

CREATE SCHEMA IF NOT EXISTS fifa.silver;
```
---
## 2. Create the Volume

You can do this either via SQL commands or through the Databricks UI.

```sql
CREATE VOLUME IF NOT EXISTS fifa.default.fifa_landing;
```

Create two folders inside the volume: landing/ and archive/.

Upload paises.csv to fifa_landing. If you want to add or modify countries, you may need to update the schema on the second run.

Upload the source CSV files to:
```md
/Volumes/fifa/default/fifa_landing/landing/
```

Expected files:
```md
players_20260930.csv
players_20260924.csv
```
These files are also included in the repository under data/:
```md
players_20260924.csv: from Kaggle — EA Sports FC 27 Player Ratings.
```
```md
players_20260930.csv: 10 first players from the previous CSV, with edition changed to fc26, snapshot date changed, and some ratings and clubs modified.

```
```md
paises.csv: from the lifeles666 GitHub gist, with some countries fixed (e.g. United Kingdom split into England, Wales, Scotland) and an ea_name column added.
```


---
## 3. Run the Pipeline

Run the notebooks in the required dependency order or launch the Databricks Workflow Job.

The pipeline will:

1. Ingest the source files into Bronze.
2. Build and update the dimensions.
3. Apply SCD1/SCD2 logic.
4. Build bridge tables.
5. Populate fact tables.
6. Run data quality validations.

---
## 4. Run Data Quality Validation

Run:
```md
validations_silver.py
```
All validation checks should pass before consuming the Silver layer from Power BI.

---
## 5. Connect Power BI

Connect Power BI to the:
```md
fifa.silver
```
Connect Power BI to the fifa.silver schema. You can authenticate using OAuth or a Personal Access Token. OAuth is recommended for interactive development.


---
## Author
Portfolio project created for:

 Databricks Data Engineer Associate certification preparation

 Data Engineering interviews

 Demonstrating practical experience with:

- Databricks

- Delta Lake

- Auto Loader

- SCD1 / SCD2

- Data Quality

- Databricks Workflows

- Power BI

----
## License

Educational and non-commercial use.

EA Sports FC player data belongs to Electronic Arts.

