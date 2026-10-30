# Global Displacement & Armed Conflict Analysis (2000–2026)

This repository contains the data extraction scripts, SQL transformation models, and visualization logic for analyzing the relationship between armed conflict and global migration patterns. The project tracks the flow of UNHCR "Persons of Concern" (refugees, asylum seekers, internally displaced persons) and models their displacement against the UCDP Armed Conflict Dataset.

The workflow utilizes an ELT (Extract, Load, Transform) architecture, shifting away from legacy ETL tools (like Alteryx) in favor of Python for extraction, Snowflake for cloud-native transformations, and Tableau for geospatial and network visualization.

## Architecture & Tech Stack

* **Python:** API extraction, pagination, and local file staging.
* **Snowflake (SQL):** Data warehousing, pre-aggregation, window functions, and native geospatial data generation.
* **Tableau:** End-state visualization, spatial network graphing, and parameter-driven dashboarding.

## Data Sources

1. **UNHCR Population API:** Retrieves core demographic figures for all "Persons of Concern" globally. Fields include country of origin, country of asylum, and specific demographic volumes (Refugees, Asylum Seekers, IDPs, Stateless Persons, Others of Concern).
2. **UCDP (Uppsala Conflict Data Program) Armed Conflict Dataset:** Provides historical records of conflict locations, intensity levels (e.g., minor conflict vs. war), and specific actors involved (`side_a` for government forces, `side_b` for non-state actors/militias).

## Methodology & Data Pipeline

### 1. Extraction (Python)

The extraction process uses a custom Python script to query the `api.unhcr.org/population/v1/population/` endpoint.

* **Chunking & Throttling:** Requests are batched year-by-year with built-in time delays to bypass API rate limits and prevent timeout/corrupt file errors.
* **Dynamic Schema Mapping:** The script dynamically parses the JSON response to ensure all demographic columns are captured cleanly before exporting to a consolidated CSV for staging.

### 2. Transformation (Snowflake)

The core analytical model is built in Snowflake using Common Table Expressions (CTEs).

* **Pre-Aggregation to Prevent Fan-Out:** Because UCDP tracks data at the conflict level and UNHCR tracks at the country level, UCDP data is pre-aggregated to the country-year level. The `LISTAGG()` function is used to consolidate multiple armed actors operating in the same country into a single row, preventing the mathematical duplication of migration volumes.
* **Geospatial Processing:** Snowflake's native spatial functions (`ST_MAKEPOINT`, `ST_MAKELINE`) convert standard latitude and longitude coordinates into spatial geographic objects (points and paths). `ST_ASWKT` is utilized to format these objects as Well-Known Text to ensure clean ingestion into Tableau via flat files.
* **Displacement Lag Calculation:** A `LAG()` window function pushes conflict intensity metrics forward by one year, allowing the model to accurately correlate a spike in violence in Year *T* with the resulting wave of out-migration in Year *T+1*.

### 3. Visualization (Tableau)

The final views are built to visualize geographic burden and causality.

* **Network Mapping:** Using the Snowflake-generated spatial lines, Tableau maps migration routes from Origin nodes to Asylum nodes, visualizing both the volume of displaced people and the physical distance traveled.
* **Dynamic Filtering:** Employs Level of Detail (LOD) expressions, Sets, and Top N parameter filters to calculate true global percentages of displacement (e.g., comparing regional vs. global burden) without breaking dimension filters.
* **Actor & Intensity Correlation:** Visualizes how specific conflict environments (and the armed actors operating within them) drive differing volumes of refugees using lagged time-series overlays and categorical distributions.

## Setup & Execution

1. **Python Extraction:** Run `unhcr_extraction.py` to pull the 2000–2026 data. The script will output a consolidated CSV to your local working directory.
2. **Database Loading:** Ingest the UNHCR and UCDP CSVs into raw staging tables in Snowflake.
3. **SQL Transformation:** Execute the master `CREATE OR REPLACE TABLE final_unhcr_poc_ucdp_conflict_table` script in Snowflake to build the final joined and spatially-enabled view.
4. **Tableau Connection:** Connect Tableau directly to the newly created Snowflake table using the native Snowflake connector to immediately leverage the spatial and lagging logic.
