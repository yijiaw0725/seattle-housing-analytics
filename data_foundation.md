# Data Foundation
### Sources, Preparation, and Exploratory Findings — King County Housing Price Analysis

This document covers what data was used, how it was prepared, and what the exploratory analysis revealed. For full acquisition details and code, see [`data_acquisition.md`](data_acquisition.md).

---

## 1. Data Overview

Four public datasets were combined for this analysis. Each is described briefly below, including the time period covered and what was excluded before analysis.

| Dataset | Period | What was kept |
|---------|--------|---------------|
| KC Assessor — Property Sales | 2015 – 2024 | Voluntary open-market single-family home sales above $10,000, on properties with a residential building |
| KC Assessor — Building & Parcel | Full history | Single-family buildings (1 unit, 200–15,000 sq ft, built 1870–2024); all King County parcels |
| SPD Crime Incidents | 2015 – 2024 | Seattle city limits only; incidents with valid GPS coordinates (~85% of all records) |
| WA OSPI School Assessments | 2023 – 2024 | King County public schools (K–12); overall student population; main state tests (SBAC, WCAS) |
| KC GIS — School Districts & Parcel Coordinates | Current snapshot | School district boundary polygons; lat/lon for all parcels (used to assign each property to a district) |

---

## 2. Housing Price EDA

**Source:** `kc_housing_eda.ipynb` · KC Assessor (RPSale, ResBldg, Parcel, LookUp)

### 2.1 Data Preparation

| Step | What was done |
|------|---------------|
| Deduplication | Kept the most recent sale per parcel (PIN); kept the first building record per PIN |
| Arms-length filter | SaleReason=1, PropertyClass=8, SalePrice>$10K → 881K records retained (36.5% of all sales) |
| SFR filter | NbrLivingUnits=1, SqFtTotLiving 200–15,000, YrBuilt 1870–2024 → ~479K buildings |
| Code decoding | SaleReason, HeatSystem, WfntLocation decoded via LookUp table |
| Feature engineering | 94.9% of buildings have YrRenovated=0; created `EffectiveAge = SaleYear − max(YrBuilt, YrRenovated)` |

### 2.2 Charts

- **Median SFR sale price by year (1990–2024)** — long-run price trajectory
- **Transaction volume + median price by month** — seasonality pattern
- **SaleReason distribution** — proportion of arms-length vs. non-market sales
- **Correlation heatmap** — SalePrice vs. living area, grade, waterfront footage, traffic noise, and view
- **SalePrice vs. SqFtTotLiving** — scatter with trend line
- **Median price and record count by building grade** — dual-axis bar + line
- **Price distribution by building grade** — box plots
- **Median price by traffic noise level** — bar chart
- **Median price per sq ft by year** — $/sqft trend line

### 2.3 Key Findings

- Median SFR price grew from ~$150K (1990) to a peak near $900K (2022), driven by two distinct run-ups: 2012–2018 and 2020–2022. A modest correction followed in 2023–2024.
- Transaction volume peaked in 2005–2007 and again in 2020–2021; declined post-2022.
- Living area is the strongest single predictor of price (r ≈ 0.55+). Price roughly doubles from Grade 7 (Average, ~$500K) to Grade 10 (~$1.2M+).
- Waterfront properties carry a premium of ~150%+ over non-waterfront comparables.
- Severe traffic noise is associated with a ~10–15% price discount.
- Spring and summer see the highest transaction volumes; prices are marginally higher than fall and winter.

---

## 3. School Quality EDA

**Source:** `kc_schools_housing.ipynb` · WA OSPI, KC GIS

### 3.1 Data Preparation

| Step | What was done |
|------|---------------|
| Suppressed data exclusion | Schools with withheld pass rates (small enrollment) excluded before averaging |
| Composite score | Each school's Math, ELA, and Science pass rates averaged into one score |
| District aggregation | All schools within a district averaged into a single district-level score |
| Spatial join | Parcel lat/lon (from KC GIS) matched to school district boundary polygons — 100% of parcels assigned |
| District name normalization | Capitalization and punctuation standardized to align OSPI names with GIS district names |

### 3.2 Charts

- **Pass rate histograms — Math, ELA, Science** — distribution across all King County schools; bimodal shape (clear high/low performer split)
- **Top 15 districts by subject pass rate** — grouped bar chart (Mercer Island, Lake Washington, Bellevue lead)
- **Composite score distribution — all schools** — cumulative ranking curve
- **Math vs. ELA correlation** — scatter plot
- **School quality vs. median house price by district** — bubble chart (bubble size = transaction volume)
- **Median SFR price by school quality quartile** — bar chart (Q1 lowest to Q4 highest)

### 3.3 Key Findings

- Pass rates range from ~20% to ~90% across schools, with a bimodal distribution suggesting a clear split between high- and low-performing schools.
- Math and ELA pass rates are highly correlated (r > 0.9), supporting the composite average as a reasonable single quality signal.
- Top districts (Mercer Island, Lake Washington, Bellevue) consistently outperform the county average across all three subjects.
- School quality and median home prices show strong district-level correlation (r = 0.77).
- Properties in the top school quality quartile carry a median price premium of approximately **+98.6%** over the bottom quartile.

---

## 4. Crime EDA

**Source:** `kc_crime_housing.ipynb` · SPD via Seattle Open Data Portal

### 4.1 Data Preparation

| Step | What was done |
|------|---------------|
| Coordinate filtering | Retained only records with valid GPS coordinates (~85% of all SPD incidents) |
| Geographic restriction | Analysis limited to Seattle city limits; King County suburbs excluded |
| Deduplication | Most recent sale per PIN; first building record per PIN |
| Crime score computation | BallTree haversine query: counted serious crimes within 500m of each property in the 12 months before sale date |
| Neighborhood stability filter | Neighborhoods with fewer than 20 transactions excluded from aggregate analysis |

### 4.2 Charts

- **Crime heatmap — Seattle city limits** — geographic density map showing concentration by area
- **Annual crime volume 2015–2024** — dual-axis (total incidents bar + serious crime line)
- **Average monthly crime volume** — seasonality bar chart
- **Top neighborhoods by serious crime density** — bar chart
- **SalePrice vs. nearby crime count** — scatter plot with trend line (500m radius, 12-month window)
- **Median SFR price by crime exposure quintile** — bar chart (Q1 lowest to Q5 highest exposure)
- **Neighborhood crime exposure vs. price per sq ft** — bubble chart (bubble size = transaction volume)

### 4.3 Key Findings

- Total crime volume was stable at 70–77K incidents per year from 2015–2024. Serious crime rose ~9% (2015 to 2022 peak), with violent crime up ~45% over the same period.
- The direct correlation between nearby crime and sale price is real but modest: r = −0.105 (serious crime) and r = −0.156 (violent crime).
- The raw price gap between the highest and lowest crime quintiles is only about −1.1%, partly because high-crime areas tend to be denser urban neighborhoods with smaller homes — making price per sq ft a more informative comparison.
- Urban amenities (walkability, proximity to employment and services) partially offset the crime discount in Seattle's most active neighborhoods, such as Capitol Hill and First Hill.
- Low-crime neighborhoods such as Magnolia and Laurelhurst correlate with higher prices, but their relative isolation is also a contributing factor.
