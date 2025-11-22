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
### 1.Data Acquistion
   I retrieved the data from the API and loaded it into a pandas DataFrame. The raw dataset contained daily testing records for all U.S. states over time.
   #### Intial Exploration
    - Multiple testing and case metrics per state/date
    - Mixed data types that needed standardization
    - Significant missingness in some fields
    - date stored as an integer in YYYYMMDD format
### 2.Data Cleaning
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
### 3.New York State Focus
I filtered the cleaned dataset to isolate New York records:
```python
ny_df = df[df['state'] == 'NY'].copy()
```
Using .copy() was intentional as it prevents SettingWithCopyWarning and ensures my NY dataframe is independent from the original dataset.
After filtering, I removed columns that were entirely empty for NY to reduce noise and focus on available metrics.

### 4.Feature Engineering
Daily Testing (tests_daily)
Because tests_combined_total is cumulative, I derived the daily number of tests as:
```python
ny_df = ny_df.sort_values('date')
ny_df['tests_daily'] = ny_df['tests_combined_total'].diff()
ny_df['tests_daily'] = ny_df['tests_daily'].fillna(0)
```
This step converts the cumulative counts into an approximate number of tests done each day, which helps me identify peak testing days, short-term ups and downs, and periods when testing activity was rising or falling over time

On some dates, the daily value is exactly zero because the reported cumulative total does not change compared to the previous day. I left these zeros as-is, since they likely reflect days when no new tests were recorded or reported in the source data (for example holidays)

### 5. Analysis and Visualizations

#### Summary Statistics

For New York, the cleaned dataset contains 654 daily records. The date range runs from March 2020 through the end of 2021, and data completeness is high for the main testing and case metrics such as `tests_combined_total` and `cases_conf_probable`, but lower for some of the more detailed breakdown fields (for example, certain antigen-related metrics).

Key New York testing metrics from the analysis:

- Total cumulative tests: approximately 83,884,958 (final value of `tests_combined_total`).
- Average of cumulative totals (`tests_combined_total`): around 33822544. This is the mean of a cumulative series, so it does not really tell us “average tests per day”.
- Average daily tests (`tests_daily`): around 62,193 tests per day, using the daily series I created.
- Peak daily testing: a few days with very high daily values compared to the rest, showing the busiest testing days in the dataset.

#### Trends Observed

Looking at the time period as a whole:

- Early pandemic (March–June 2020): testing started from very low levels and ramped up gradually as capacity was still limited.
- Mid-period (roughly July 2020–March 2021): testing volume increased significantly, and the cumulative curve became much steeper, reflecting sustained high testing activity.
- Later period (April 2021 onward): testing remained high but more stable, with visible surges and plateaus that align with different phases of the pandemic.

The cumulative view shows the overall growth in testing, while the daily series makes it easier to see short-term increases and slowdowns.

#### Statistical Considerations

The assignment specifically requested the average of `tests_combined_total` for New York, so I computed that value. However, `tests_combined_total` is a cumulative series, which means:

- Its mean is heavily influenced by the larger values later in the time range.
- The mean of a cumulative series does not correspond to a meaningful “average per day” in the way a daily series would.

Because of this, I treated the average cumulative value mainly as a required output for the task and focused interpretation on more meaningful metrics:

- Daily testing counts (`tests_daily`, derived via `.diff()`).
- Peak testing capacity (maximum of `tests_daily`).
- The final cumulative total (maximum of `tests_combined_total`), which represents the total number of tests over the period.

#### Visualizations
1. Cumulative tests over time  
   I created a line chart of `tests_combined_total` versus `date` for New York.  
   This plot shows:
   - A slow start in early 2020 while testing capacity was still developing.
   - A steep increase through late 2020 and early 2021 as testing scaled up.
   - Continued growth afterward, with the slope becoming somewhat less steep as cumulative totals get larger.

   I chose a line chart because this is time-series data, rotated the x-axis labels for readability, and added clear axis labels and a descriptive title.

2. Daily testing volume  
   I also plotted `tests_daily` (the day-to-day difference of `tests_combined_total`) over time.  
   This chart highlights:
   - Peak testing days where daily volume is much higher than surrounding days.
   - Short-term fluctuations in testing activity, including periods where testing increases or levels off.
   - Days with a daily value of zero, which occur when the reported cumulative total does not change compared to the previous day (for example, weekends or holidays when no new tests are recorded or reported).

   While the cumulative plot is useful for understanding overall scale and long-term growth, the daily plot provides more insight into changes in operational testing levels over time and how activity varied across different phases of the pandemic.

## Limitations & Notes

- Reporting artifacts: Some days show zero due to absence of testing.
- Cumulative averages: The mean of `tests_combined_total` is reported for completeness but is not used as a main interpretive metric.
- Missing breakdowns: Several detailed metrics (e.g., some antigen-related fields) are never reported for New York and are dropped from the NY subset.
- State-level view: All results are at the state level and do not capture county or city-level variation.

## Future Work

- Compare New York testing patterns with other large states (CA, TX, FL).
- Normalize testing metrics by population to get per-capita testing rates.
- Combine testing data with case and hospitalization data to analyze positivity and severity.
- Apply time-series models to forecast future testing demand or detect structural breaks.

