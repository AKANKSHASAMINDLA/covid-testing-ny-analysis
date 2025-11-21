# covid-testing-ny-analysis
Analysis of Johns Hopkins COVID testing data for New York State
## Overview
This project is based on COVID testing data from Johns Hopkins Coronavirus Resource Center via the public API.
The analysis walks through the entire workflow, starting from API retrieval to data cleaning, exploratory analysis, and visualization. It then highlights how testing patterns changed in New York State between March 2020 and the end of 2021.
## Data Source
**API Endpoint :** https://jhucoronavirus.azureedge.net/api/v1/testing/daily.json <br>
**Data Provider :** Johns Hopkins University <br>
**Format :** Json
## Methodolgy
### 1. Data Acquistion
   I retrieved the data from the API and loaded it into a pandas DataFrame. The raw dataset contained daily testing records for all U.S. states over time.
   #### Intial Exploration
    - Multiple testing and case metrics per state/date
    - Mixed data types that needed standardization
    - Significant missingness in some fields
    - date stored as an integer in YYYYMMDD format
### 2. Data Cleaning
#### Type Conversions
The raw data required several type conversions for proper analysis:

```python
df['date'] = pd.to_datetime(df['date'].astype(str), format='%Y%m%d')
for column in numeric_columns: <br>
    df[column] = pd.to_numeric(df[column], errors='coerce').astype('Int64')
```
I chose Int64 instead of standard int64 because it supports NA values
#### Handling Missing Data
- Checked for null values across all columns using df.isna().sum() to understand which metrics were missing and at what scale.
- Dropped columns with 100% missing data for New York (for example, certain antigen test breakdowns that New York did not report).
- Kept partially populated columns that still provided useful information instead of removing them entirely.
- Used dropna() only when calculating specific metrics (such as averages) to avoid unnecessarily shrinking the dataset or artificially inflating counts.
#### Quality Checks
- Made sure there was only one record per (state, date) combination by using drop_duplicates(subset=['state', 'date']).
- Confirmed that cumulative metrics such as tests_combined_total were generally non-decreasing over time, while allowing for occasional corrections where values were adjusted.

