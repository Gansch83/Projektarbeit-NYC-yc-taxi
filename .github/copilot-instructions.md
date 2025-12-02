# NYC Yellow Taxi Fare Prediction - AI Agent Instructions

## Project Overview
This is a **data science project** analyzing NYC Yellow Taxi trip data (February 2025) with comprehensive data validation, feature engineering, weather enrichment, and ML-based fare prediction. The project follows a structured pipeline from raw data to trained models.

**Primary artifact**: `NEW Projektarbeit.ipynb` - main analysis notebook (~2,450 lines)  
**Language**: Python with Jupyter notebooks  
**Dataset**: NYC TLC Yellow Taxi trip data (Parquet format, ~3-5M rows/month)

## Architecture & Data Pipeline

### Three-Stage Pipeline
1. **Data Validation & Cleaning** (Lines 1-1300)
   - Domain validation against NYC TLC regulations (fare structure, rate codes, surcharges)
   - Comprehensive flag system: `is_invalid_*` columns track violations
   - Creates `df` (all data with flags) and `df_clean` (filtered, valid records)
   - Retention rate typically 85-95% after validation

2. **Feature Engineering & Weather Enrichment** (Lines 1300-1900)
   - Temporal features: `pickup_hour`, `pickup_day_of_week`, `is_weekend`, `is_rush_hour`, `is_overnight`
   - Geospatial: Borough-level aggregation using `taxi_zone_lookup.csv`
   - Weather API integration: OpenWeatherMap hourly data with **caching in `WeatherData/`**
   - Derived metrics: `trip_duration_min`, `fare_per_mile`, `tip_to_fare`, `median_trip_distance`

3. **Machine Learning Models** (Lines 1900-2451)
   - Three models compared: Linear Regression, Random Forest, Gradient Boosting
   - Two evaluation strategies: 70/30 split + 5-fold cross-validation
   - Feature importance analysis across all models
   - Visualizations: correlation matrices, performance comparisons, prediction scatter plots

### Critical Configuration Block
**Lines 132-215** in `NEW Projektarbeit.ipynb` define all validation thresholds:
```python
ALLOWED_VENDOR_IDS = {1, 2, 6, 7}  # NYC TLC registered vendors
ALLOWED_RATECODES = {1}  # Standard rate only (excludes airport flat fares)
ALLOWED_PAYMENT_TYPES = {1, 2}  # Credit card and cash only
MIN_FARE = 3.50  # NYC base fare as of June 2024
FARE_PER_MILE_MIN = 2.0, FARE_PER_MILE_MAX = 25.0
```
**When modifying validation rules**, update these constants rather than hardcoding values in validation cells.

## Development Workflows

### Running the Analysis
```powershell
# 1. Ensure data file exists at expected path:
~/Downloads/TripData/yellow_tripdata_2025-02.parquet

# 2. Weather data caching (important!):
# - First run: Fetches from OpenWeatherMap API (~30 min, 672 API calls)
# - Subsequent runs: Loads from WeatherData/weather_data_2025_02.parquet (instant)
# - To refresh weather data: Delete cache file and re-run weather cells

# 3. Run notebook cells sequentially (no cell skipping!)
# Order matters: validation → feature engineering → modeling
```

### Working with Weather Data
- **API Key Required**: Set `OPENWEATHER_API_KEY` in config cell (Line 206)
- **Rate Limiting**: 1.1 second delay between API calls (free tier: 60/min)
- **Cache Location**: `WeatherData/weather_data_2025_02.parquet`
- **Data Structure**: Hourly records with temp (°C), humidity (%), wind (km/h), weather conditions
- **Merging**: Joins to `df_clean` on `pickup_date` + `pickup_hour`

### Helper Functions Pattern
Five core utility functions (Lines 237-447):
```python
create_flag(name, condition, explicit=True)  # Creates boolean flag column
report_flag(flag_col)                        # Prints formatted violation count
pct_str(count, total)                        # Formats "X (Y%)" strings
print_unique_values(column, sort_by, sort)   # Inspects categorical columns
iqr_show(data_series)                        # IQR outlier analysis
```
**Usage**: Always call `create_flag()` after validation logic to maintain audit trail.

## Project-Specific Conventions

### Flag Management System
- **Three flag categories** tracked in lists: `enhancement_flag`, `reporting_flags`, `explicit_flags`
- **Explicit flags** trigger row removal (e.g., `is_invalid_VendorID`, `is_invalid_trip_distance`)
- **Reporting flags** are informational only (e.g., `is_fare_negative`, `is_cash_with_tip`)
- **Naming convention**: `is_invalid_*` for validation, `is_*` for boolean features
- **Final consolidation** (Line 1084): `df['is_valid_record'] = ~df[explicit_flags].any(axis=1)`

### NYC Taxi Domain Knowledge
**Airport Trips** (complex fare logic):
- JFK flat fare: $70 base (RatecodeID=2, no separate airport fee)
- JFK metered: $1.75 airport access fee (RatecodeID=1)
- LGA trips: $6.75 total ($1.75 + $5.00 surcharge)
- Rush hour airport surcharge: +$5.00 (4pm-8pm weekdays, non-holidays)

**Surcharge Validation** (Lines 783-990):
- `extra`: Overnight ($1) or rush hour ($2.50) surcharges
- `congestion_surcharge`: $2.50 for Manhattan CBD pickups
- `cbd_congestion_fee`: $0.75 additional CBD fee
- Cross-validation: Verify surcharges match pickup time/location

### Feature Engineering Patterns
When adding features:
1. Create new column in `df` immediately after validation
2. Add column name to `enhancement_flag` list
3. Ensure feature propagates to `df_clean` after filtering
4. Include in `X` feature matrix for modeling (Line 2069)

Example:
```python
df['new_metric'] = df['fare_amount'] / df['trip_distance']
enhancement_flag.append('new_metric')
# After df_clean creation:
# Feature automatically available in df_clean
```

## Common Debugging Scenarios

### "KeyError: column not found"
- **Cause**: Running cells out of order or skipping validation cells
- **Fix**: Restart kernel and run from top; flag columns only exist after `create_flag()` calls

### "Weather merge produces NaN values"
- **Cause**: Missing hourly weather records or date mismatch
- **Fix**: Check `weather_df` completeness; verify `pickup_date` is datetime64 type

### "Model performance suddenly degrades"
- **Cause**: Changed validation thresholds without retraining on new `df_clean`
- **Fix**: Re-run entire pipeline from data loading to ensure consistency

### "API quota exceeded" (OpenWeatherMap)
- **Cause**: Deleted cache and re-fetching weather data multiple times
- **Fix**: Free tier allows 1,000 calls/day; wait 24 hours or use cached file

## Key File Locations
- **Main notebook**: `NEW Projektarbeit.ipynb` (active development)
- **Legacy notebooks**: `Projektarbeit.ipynb` (previous version), `ML intro.ipynb` (minimal example)
- **Lookup data**: `taxi_zone_lookup.csv` (LocationID → Borough mapping, 265 zones)
- **Weather cache**: `WeatherData/weather_data_2025_02.parquet` (auto-generated, ~670 records)
- **Source data**: `~/Downloads/TripData/yellow_tripdata_2025-02.parquet` (not in repo)

## Best Practices for AI Agents

1. **Always check configuration block first** (Lines 132-215) before suggesting validation changes
2. **Preserve flag tracking**: When modifying validation, update both `create_flag()` call and explicit_flags list
3. **Respect cell execution order**: Never suggest running modeling cells before validation completes
4. **Use helper functions**: Don't reinvent `pct_str()` or `report_flag()` - they maintain consistent formatting
5. **Weather data is expensive**: Always check for cached file before suggesting API calls
6. **Feature importance varies**: Random Forest prioritizes `trip_distance` (40-50%), but Linear Regression weighs borough features heavily
7. **Validation is strict**: 10-15% of records typically fail validation - this is expected and documented

## External Resources
- **NYC TLC Fare Structure**: https://www.nyc.gov/site/tlc/passengers/taxi-fare.page
- **OpenWeatherMap API**: Requires free tier API key (1,000 calls/day limit)
- **Dataset Source**: NYC TLC Trip Record Data (https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
