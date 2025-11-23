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
![End-to-End Workflow](https://github.com/AKANKSHASAMINDLA/covid-testing-ny-analysis/blob/main/Workflow.png)

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
- I dropped columns that were completely null for New York.
- I computed column completeness (% non-null) to see which metrics are reliable for NY.
###### Case breakdown logic:
cases_confirmed and cases_probable are only populated for part of the timeline.To avoid this, I applied a simple rule:<br>
If a case column has < 60% completeness for NY, I drop it from the core NY frame.<br>
As a result, cases_confirmed / cases_probable are treated as secondary and removed, and I rely on the more stable cumulative cases_conf_probable series (cases_conf_probable_cum) for New York.<br>
#### Duplicate Metric Cleanup
- For New York, I compared encounters_viral_total and tests_combined_total:<br>
They were exactly identical for all 654 NY records (same value for every date).Since they represent the same testing series, I dropped encounters_viral_total and kept tests_combined_total as the primary cumulative test metric.<br>
- At the same time, I kept both:(people_viral_positive, cases_conf_probable)<br>
Even though they are numerically equal here, because in the JHU schema they represent different concepts (positive people vs reported cases). I used their equality more as a consistency check than a reason to drop either.<br>
#### Renaming
- Renamed cumulative fields: I renamed key cumulative columns with a _cum suffix for clarity, e.g.
tests_combined_total → tests_cum, people_viral_positive → people_viral_pos_cum, cases_conf_probable → cases_conf_probable_cum
This makes it obvious which series are cumulative vs. daily.
#### Feature Engineering
- Daily Testing (tests_daily), Because tests_combined_total is cumulative, I derived the daily number of tests as:
```python
ny_df = ny_df.sort_values('date')
ny_df['tests_daily'] = ny_df['tests_combined_total'].diff()
ny_df['tests_daily'] = ny_df['tests_daily'].fillna(0)
```
This step converts the cumulative counts into an approximate number of tests done each day, which helps me identify peak testing days, short-term ups and downs, and periods when testing activity was rising or falling over time

On some dates, the daily value is exactly zero because the reported cumulative total does not change compared to the previous day. I left these zeros as-is, since they likely reflect days when no new tests were recorded or reported in the source data (for example holidays)

- tests_daily_7dma = 7-day moving average of tests_daily (with min_periods=1) to smooth weekend/holiday effects. I rounded this to 2 decimals for readability.

### 5. Analysis and Visualizations

#### Summary Statistics

For New York, the cleaned dataset contains 654 daily records. The date range runs from March 2020 through the end of 2021, and data completeness is high for the main testing and case metrics such as tests_combined_total and cases_conf_probable, but lower for some of the more detailed breakdown fields (for example, certain antigen-related metrics).

Key New York testing metrics from the analysis:

- Total cumulative tests: approximately 83,884,958 (final value of `tests_combined_total`).
- Average of cumulative totals (tests_combined_total): around 33822544. This is the mean of a cumulative series, so it does not really tell us “average tests per day”.
- Average daily tests (tests_daily): around 128,264 tests per day, using the daily series I created.
- Peak daily testing: a few days with very high daily values compared to the rest, showing the busiest testing days in the dataset.(peak testing valies is 424,199)

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
   ![Cumulative tests over time](https://github.com/AKANKSHASAMINDLA/covid-testing-ny-analysis/blob/main/ny_cumulative_tests.png)
   I created a line chart of `tests_combined_total` versus `date` for New York.  
   This plot shows:
   - Early pandemic (March–June 2020): testing started from very low levels and ramped up gradually as capacity was still limited.
   - Rapid expansion (roughly July 2020–March 2021): much steeper slope, reflecting aggressive testing scale-up.
   - Stabilization (April 2021 onward): continued growth but with a slightly flatter slope, as cumulative tests approach ~84M.
  I chose a line chart because this is time-series data, rotated the x-axis labels for readability, and added clear axis labels and a descriptive title.

3. Daily testing volume  (Used Jupyter Notebook and Tableau for clear Visualization)
    ![ny_daily_tests](https://github.com/AKANKSHASAMINDLA/covid-testing-ny-analysis/blob/main/ny_daily_tests.png)
    ![ny_daily_tests](https://github.com/AKANKSHASAMINDLA/covid-testing-ny-analysis/blob/main/using_tableau_ny_daily_tests.png)
   I also plotted `tests_daily` (the day-to-day difference of `tests_combined_total`) over time.  
   This chart highlights:
   - Trend over time: The blue 7-day average line rises through 2020, stays very high around late 2020–early 2021, dips in the middle of 2021, and then climbs again towards the end of      2021. This tells me overall testing first ramped up, then slowed a bit, and later picked up again.
   - Spikes and dips: A few very tall spikes and big drops are likely due to “zero testing” days, so I use the smoother blue line to understand the true testing trend.<br>
   
   While the cumulative chart helps me see the total volume of tests and the long-term growth, the daily chart is more helpful for understanding how testing actually changed day to day and how those levels shifted across different stages of the pandemic.

## Limitations & Notes

- Reporting artifacts: Some days show zero due to absence of testing.
- Cumulative averages: The mean of `tests_combined_total` is reported for completeness but is not used as a main interpretive metric.
- Missing breakdowns: Several detailed metrics (e.g., some antigen-related fields) are never reported for New York and are dropped from the NY subset.
- State-level view: All results are at the state level and do not capture county or city-level variation.

## Future Work

- Compare New York testing patterns with other large states.
- Normalize testing metrics by population to get per-capita testing rates.
- Combine testing data with case and hospitalization data to analyze positivity and severity.
- Apply time-series models to forecast future testing demand or detect structural breaks.

