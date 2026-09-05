Capital Bikeshare Demand Forecasting Pipeline
Overview

End-to-end machine learning pipeline to forecast hourly bike demand and dock availability across Washington DC's Capital Bikeshare network. Built to help operations teams proactively dispatch rebalancing vans by predicting station depletion or overflow before it happens.

Problem Statement

Predict two targets per station per hour:

Demand: Total bikes rented from a station in a 1-hour window
Dock Availability: Reconstructed real-time count of available docks
Data Sources

Three external sources integrated via spatial and temporal joins:

Source	Description	Volume
Capital Bikeshare S3	12 months of trip records (Apr 2025 - Mar 2026)	8.8M+ events
GBFS Live API	Station metadata - name, lat/lon, dock capacity	800+ stations
Open-Meteo API	Hourly weather - temperature, precipitation, wind, snowfall	Hourly granularity
Pipeline Architecture
Raw Data Ingestion
├── Trip CSVs from Capital Bikeshare S3 bucket
├── GBFS station information API
└── Open-Meteo historical weather API
        ↓
Data Cleaning
├── Filter trips < 60 seconds (false starts)
├── Standardize station IDs
├── Forward-fill missing weather values
└── Save as Parquet (processed layer)
        ↓
Data Integration
├── Spatial Join: Trips + GBFS (on station_id)
└── Temporal Join: Trips + Weather (on rounded hour)
        ↓
Dock Availability Reconstruction
└── Bounded state machine on 8.8M chronological events
        ↓
Feature Engineering
├── Geographic clustering (K-Means on lat/lon)
├── Temporal features (hour, day, is_weekend)
├── Lag features and rolling averages (strict shift())
└── Environmental features (temperature, precipitation)
        ↓
Modeling
├── Decision Tree (baseline)
├── Random Forest
├── CatBoost
└── LSTM
Key Engineering Decisions
Dock Availability Reconstruction

Since GBFS only provides current capacity, historical dock availability was mathematically reconstructed by treating every trip as a dock transaction:

Bike rental = +1 available dock (dock freed up)
Bike return = -1 available dock (dock occupied)

All 8.8M events were sorted chronologically per station and processed via a bounded NumPy state machine, clamping values between 0 and max capacity to self-correct for phantom rebalancing van movements.

Geographic Dimensionality Reduction

800+ unique stations were clustered into geographic zones using K-Means on latitude/longitude coordinates. The elbow method was used to find optimal K (5-8 clusters), reducing dimensionality while preserving regional behavioral patterns (Downtown, Suburban, Tourist Hub, etc.).

Preventing Data Leakage

All temporal features use strict shift() operations ensuring only historical data (t-1, t-2, etc.) is used to predict state t. Train/test split is strictly chronological - training on Apr-Jan, testing on Feb-Mar.

Evaluation Metrics
RMSE (primary): Penalizes large errors heavily - missing by 10 bikes causing an empty station is exponentially worse than missing by 1 bike ten times
MAE (secondary): Human-interpretable average error in physical bikes
Tech Stack
Language: Python
Data Processing: Pandas, NumPy
Storage Format: Parquet
APIs: requests, Capital Bikeshare GBFS, Open-Meteo
ML Models: scikit-learn, CatBoost, PyTorch (LSTM)
Clustering: scikit-learn KMeans
Visualization: Matplotlib, Seaborn
Project Structure
capital_bikeshare/
├── raw_trips/              # Downloaded trip CSVs (12 months)
├── raw_gbfs/               # Raw GBFS station data
├── raw_weather/            # Raw Open-Meteo weather data
├── processed/
│   ├── cleaned_trips.parquet
│   ├── cleaned_gbfs.parquet
│   ├── cleaned_weather.parquet
│   ├── spatial_joined_trips.parquet
│   ├── master_joined_data.parquet
│   └── validated_dock_tally.parquet
└── capital_bikeshare_final.ipynb
Setup and Usage
Prerequisites
bash
pip install pandas numpy scikit-learn catboost torch requests
Running the Pipeline

Execute cells in order in capital_bikeshare_final.ipynb:

Data ingestion - downloads trip CSVs and fetches APIs
Data cleaning - standardizes and validates all three sources
Spatial join - merges trips with station metadata
Temporal join - attaches hourly weather to trips
Dock reconstruction - builds availability history
Feature engineering - creates ML-ready features
Modeling - trains and evaluates all four models
Dock reconstruction - builds availability history
Feature engineering - creates ML-ready features
Modeling - trains and evaluates all four models
