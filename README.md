# Real-Time Flight Tracking Pipeline

An end-to-end data engineering and ML pipeline on Google Cloud, progressing from historical batch analysis to true real-time streaming predictions on live flight data.

## Overview

Three parts, each reducing latency on the same underlying modeling questions:

1. **Batch processing.** Downloads, cleans, and loads historical U.S. flight on-time performance data (BTS.gov, 2024) into BigQuery, then trains three BigQuery ML models: linear regression (predict arrival delay), logistic regression (classify significant delays), and K-means (cluster flights by delay/distance profile).
2. **Micro-batch ingestion.** Polls live flight state vectors from the OpenSky Network API, lands them in Cloud Storage, and loads them into BigQuery, then retrains the same class of models (velocity regression, on-ground classification) on live-sourced data.
3. **True real-time streaming.** A Pub/Sub producer/consumer architecture: a producer publishes live OpenSky flight states to a topic, and a subscriber pulls each message, inserts it into BigQuery, and immediately runs `ML.PREDICT` on it, scoring individual flight events with no batch delay.

## Key Findings

- **A real debugging story is documented, not hidden.** The Part II on-ground classification model initially failed with "requires at least 2 unique labels" despite the raw data containing both classes (383 `TRUE`, 4,117 `FALSE`). A diagnostic query traced the cause: the model's `WHERE` clause excluded `NULL` feature values, and every single `on_ground = TRUE` record happened to have a `NULL` altitude or velocity, so the filter silently removed the entire positive class. Fixed with `COALESCE` instead of exclusion.
- **The same modeling pattern holds across all three parts:** predicting a continuous target (delay, velocity) with linear regression and a related binary outcome (delayed/not, on-ground/not) with logistic regression, applied consistently as the data source moves from static historical batch to live streaming.
- **Part III closes the loop end to end:** live data published to Pub/Sub is consumed, stored, and scored by a trained ML model within the same pipeline run, including a flight-path visualization pulled directly from the streamed BigQuery table for a tracked callsign.

## Limitations

1. OpenSky's free-tier API has rate limits and incomplete global coverage, live pulls are a sample of visible aircraft, not complete air traffic.
2. The historical model (Part I) and the live-data models (Parts II/III) use different feature sets and data sources (BTS on-time performance vs. OpenSky state vectors), so their metrics aren't directly comparable, each demonstrates the same workflow on the data available at that stage.
3. The streaming pipeline (Part III) runs for a fixed timeout window rather than as an always-on service; it demonstrates the architecture end-to-end but isn't deployed via Cloud Functions or Dataflow as originally scoped.
4. No monitoring or alerting on data quality or model drift, both necessary for a production deployment.

## Tech Stack

Google Cloud (Cloud Storage, BigQuery, BigQuery ML, Pub/Sub) · Python (pandas, requests) · OpenSky Network API · Google Colab

## Repository Contents

- `real_time_flight_tracking_pipeline.ipynb`: full three-part pipeline (batch, micro-batch, real-time streaming)
- `README.md`: this file
