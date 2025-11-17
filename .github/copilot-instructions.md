# NYC Yellow Taxi Trip Data Analysis - AI Agent Instructions

## Project Overview
This is a data science project analyzing NYC Yellow Taxi trip data for February 2025. The project performs comprehensive data validation, cleansing, weather integration, and fare prediction using Jupyter notebooks with Python.

## Core Architecture

### Data Pipeline Flow
1. **Raw Data Ingestion** → Load Parquet files from `~/Downloads/TripData/`
2. **Validation & Cleansing** → Flag-based validation with detailed reporting
3. **Weather Integration** → Fetch/cache hourly OpenWeatherMap data
4. **Feature Engineering** → Calculate derived metrics (trip duration, fare per mile, etc.)
5. **ML Prediction** → Train fare prediction models with weather features

### Key Files
- `Projektarbeit.ipynb` - **Main analysis notebook** with full validation pipeline (~5,000+ lines)
- `Projektarbeit Starting.ipynb` - Earlier/alternative cleansing approach -> not needed anymore
- `taxi_zone_lookup.csv` - NYC taxi zone reference data (265 zones + EWR)
- `WeatherData/weather_data_2025_02.parquet` - Cached hourly weather data

## Critical Patterns & Conventions

### 1. NYC Taxi Data Schema (Feb 2025)
The project validates 19 distinct fields with specific NYC TLC fare rules:

**Core Trip Fields:**
- `VendorID`: {1,2,6,7} - Creative Mobile, Curb, Myle, Helix
- `tpep_pickup_datetime`, `tpep_dropoff_datetime` - Must be Feb 2025; auto-swap if reversed
- `passenger_count`: 1-5 (default to 1 if invalid)
- `trip_distance`: 0.1-500 miles
- `RatecodeID`: {1,2,3,4,5,6,99} - Standard, JFK flat, Newark, Nassau, Negotiated, Group, Unknown

**Fare Components (NYC TLC regulations June 2024):**
- `fare_amount`: $3.50 minimum (initial charge), JFK flat fare = $70.00 exact
- `extra`: Rush hour $2.50 (4pm-8pm weekdays, non-holidays) | Overnight $1.00 (8pm-6am)
- `mta_tax`: $0.50 for NYC trips (validated by borough lookup)
- `tip_amount`: Credit card only (cash tips = data error)
- `tolls_amount`: Bridge/tunnel fees
- `improvement_surcharge`: {0.00, 1.00} (was $0.30 pre-2015)
- `congestion_surcharge`: {0.00, 2.50}
- `Airport_fee`: $1.75 (JFK/LGA), $6.75 (LGA full), $5.00 (rush hour add-on)
- `cbd_congestion_fee`: {0.00, 0.75}

### 2. Validation Strategy: Flag-Based Architecture
**Never drop data immediately** - create boolean `is_invalid_*` flags first:

```python
# Pattern: Flag → Report → Filter (3-step process)
def create_flag(name, cond):
    """Create 'is_invalid_<name>' column where True = invalid record"""
    col = f"is_invalid_{name}"
    df[col] = ~cond  # Invert: True in flag means invalid
    report_flag(col)
    return col

# Example usage
create_flag("fare_amount", df['fare_amount'].between(3.50, 10000))
```

**Key flags to preserve:**
- `is_swapped_pickup_dropoff` - Auto-corrected but tracked
- `is_invalid_*` - One flag per validation rule
- `is_valid_record` - Composite flag: `~df[all_flags].any(axis=1)`

**Final datasets:**
- `df` - Original with all flag columns (NEVER modify in place)
- `df_clean` - Filtered subset where `is_valid_record == True`

### 3. Weather Data Integration (OpenWeatherMap API)
**Critical caching pattern** to avoid API rate limits:

```python
weather_cache_file = f"WeatherData/weather_data_{YEAR}_{MONTH:02d}.parquet"

# Always check cache first!
if os.path.exists(weather_cache_file):
    weather_df = pd.read_parquet(weather_cache_file)
else:
    # Fetch hourly data (24 API calls per day × days in month)
    # Rate limit: 1.1 seconds between calls (free tier = 60/min)
    weather_data = fetch_weather_data_openweather_hourly(date, api_key)
    weather_df.to_parquet(weather_cache_file)
```

**Hourly granularity:** Weather data merged on `['pickup_date', 'pickup_hour']` keys
**Metric units:** Temp (°C), Wind (km/h), Visibility (km) - all converted at fetch time

### 4. Constants & Thresholds
Define ALL validation thresholds at notebook start (after imports):

```python
# Example constants from main notebook
ALLOWED_VENDOR_IDS = {1, 2, 6, 7}
MIN_TRIP_DISTANCE = 0.1
MAX_TRIP_DISTANCE = 500
JFK_MANHATTAN_FLAT_FARE_BASE = 70.00
FEDERAL_HOLIDAYS_2025 = pd.to_datetime([...])  # For rush hour exemptions
```

### 5. Reporting Pattern
Use consistent formatting for user-facing output:

```python
def pct_str(count, total):
    """Format: '1,234 (12.34%)'"""
    percentage = (count / total * 100) if total > 0 else 0
    return f"{count:<20,} ({percentage:>6.2f}%)"

# Always reference INITIAL_ROWS for percentage tracking
print(f"Flagged: {pct_str(invalid_count, INITIAL_ROWS)}")
```

## Developer Workflows

### Running the Analysis
1. **Data Placement:** Download Parquet to `~/Downloads/TripData/yellow_tripdata_YYYY-MM.parquet`
2. **Update Config:** Set `REPORTING_YEAR` and `REPORTING_MONTH` at top of notebook
3. **API Key (Optional):** Set `OPENWEATHER_API_KEY` for weather enrichment (delete cache to refresh)
4. **Run All Cells:** Produces `df` (flagged) and `df_clean` (filtered) dataframes

### Key Cell Sections (Projektarbeit.ipynb)
- **Loading Data** → Initial Parquet load + setup
- **Configurable Thresholds** → All constants (ALWAYS update here, not in code)
- **Helper Functions** → Utility functions (create_flag, print_unique_values, fetch_weather)
- **Validation Sections** → 18 numbered validation steps (VendorID through cbd_congestion_fee)
- **Flag Reporting** → Combine flags, create df_clean
- **Enrichment** → Derived metrics (tip_per_person, fare_per_mile, vendor totals)
- **Visualization** → Matplotlib/Seaborn plots
- **Fare Prediction** → LightGBM/RandomForest with weather features

### Testing New Validations
1. Add constants to "CONFIGURABLE THRESHOLDS" cell
2. Create new validation cell with `create_flag()` call
3. Add flag name to `explicit_flags` list in "Flag Reporting" cell
4. Verify with `report_flag()` output

## Project-Specific Quirks

### Date/Time Handling
- **Pickup can be January, dropoff must be February 2025** (billing month = dropoff)
- **Auto-swap logic:** If dropoff < pickup, swap silently and flag with `is_swapped_pickup_dropoff`
- Federal holidays list hardcoded (not fetched) for rush hour exemption logic

### Fare Validation Complexity
- **JFK flat fare:** Must be exactly $70.00 (no range tolerance) when `RatecodeID == 2`
- **Total amount reconciliation:** Allows ±$2.00 tolerance for floating-point arithmetic
- **Cash tip paradox:** `payment_type == 2` (cash) should NEVER have `tip_amount > 0`

### Borough Validation (MTA Tax)
Uses LEFT JOIN with `taxi_zone_lookup.csv` to validate NYC trips:
```python
df = df.merge(zone_lookup[['LocationID', 'Borough']], 
              left_on='PULocationID', right_on='LocationID')
```
**Boroughs:** Manhattan, Brooklyn, Bronx, Queens, Staten Island (NOT "EWR")

### Weather Feature Engineering
Categorical flags created from continuous metrics:
- `is_rainy_snowy`, `is_extreme_weather`, `is_taxi_favorable`
- `is_cold` (<10°C), `is_freezing` (<0°C), `is_hot` (>30°C)
- `is_windy` (>40 km/h), `is_poor_visibility` (≤8 km)

## Dependencies
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
import requests  # Weather API
from datetime import datetime, timedelta
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error, r2_score
import lightgbm as lgb  # Optional: Falls back to RandomForest
```

## Common Pitfalls to Avoid

1. **DO NOT drop rows without flagging first** - Always create `is_invalid_*` flags for audit trail
2. **DO NOT modify df in place** - Use `.copy()` when filtering: `df_clean = df[mask].copy()`
3. **DO NOT assume column existence** - Check with `if col in df.columns` before accessing
4. **DO NOT fetch weather without checking cache** - Each API call costs time/quota
5. **DO NOT use generic validation ranges** - This project has NYC-specific thresholds from TLC regulations
6. **DO NOT ignore INITIAL_ROWS** - Always calculate percentages against original count, not current df size

## Data Sources & References
- NYC TLC Trip Record Data: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page
- NYC Taxi Fare Structure (June 2024): https://www.nyc.gov/site/tlc/passengers/taxi-fare.page
- OpenWeatherMap API: https://openweathermap.org/api/one-call-3
- Taxi Zone Lookup: NYC TLC official zone mapping (265 zones)

## Questions to Clarify

1. **Alternative notebook usage?** `Projektarbeit Starting.ipynb` seems to be an earlier approach - should this be maintained or deprecated? -> deprecated
2. **Weather data scope:** Currently fetches all 24 hours per day - is hourly granularity always needed or could daily aggregates suffice for some analyses? -> please keep hourly
3. **Model evaluation criteria:** What R² or RMSE thresholds constitute "success" for fare prediction?
4. **Production deployment:** Is this intended for one-off analysis or will it become a pipeline for ongoing monthly reports?
