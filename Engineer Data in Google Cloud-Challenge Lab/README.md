# Engineer Data in Google Cloud: Challenge Lab

### Task 1: Clean your training data

#### Navigation menu -> BigQuery


```sql=
-- create table
CREATE OR REPLACE TABLE
taxirides.taxi_training_data_800 AS
SELECT
-- add tolls_amount and fare_amount to fare_amount_252 
(tolls_amount + fare_amount) AS fare_amount_252,
pickup_datetime,
pickup_longitude AS pickuplon,
pickup_latitude AS pickuplat,
dropoff_longitude AS dropofflon,
dropoff_latitude AS dropofflat,
passenger_count AS passengers,
FROM
taxirides.historical_taxi_rides_raw
WHERE
-- sample the dataset to less than 1 Million rows
RAND() < 0.001
-- Ensure trip_distance is greater than 4
AND trip_distance > 4
-- Remove rows were fare_amount is very small
AND fare_amount >= 2.5
-- latitudes and longitudes are reasonable for the use case
AND pickup_longitude > -75
AND pickup_longitude < -71
AND dropoff_longitude > -75
AND dropoff_longitude < -71
AND pickup_latitude > 40
AND pickup_latitude < 45
AND dropoff_latitude > 40
AND dropoff_latitude < 45
-- Ensure passenger_count is greater than 4
AND passenger_count > 4
```

### Task 2: Create a BQML model

```sql=
-- create model
CREATE or REPLACE MODEL taxirides.fare_model_236
-- TRANSFORM() clause will be passed to the model
TRANSFORM(
-- * EXCEPT(feature_to_leave_out) to pass some or all of the features without explicitly calling them
* EXCEPT(pickup_datetime),
-- ST_distance() and ST_GeogPoint() GIS functions in BigQuery can be used to easily calculate euclidean distance
ST_Distance(ST_GeogPoint(pickuplon, pickuplat), ST_GeogPoint(dropofflon, dropofflat)) AS euclidean,
CAST(EXTRACT(DAYOFWEEK FROM pickup_datetime) AS STRING) AS dayofweek, 
CAST(EXTRACT(HOUR FROM pickup_datetime) AS STRING) AS hourofday
)
--  RMSE of 10 or less to complete the task
OPTIONS(input_label_cols=['fare_amount_252'], model_type='linear_reg')
AS
SELECT * FROM taxirides.taxi_training_data_800
```

### Task 3: Perform a batch prediction on new data

```sql=
-- create tabe
CREATE or REPLACE table taxirides.2015_fare_amount_predictions AS
-- Use ML.PREDICT and your model to predict Fare amount and store your results
SELECT * FROM ML.PREDICT(MODEL taxirides.fare_model_236,(SELECT * FROM taxirides.report_prediction_data))
```

