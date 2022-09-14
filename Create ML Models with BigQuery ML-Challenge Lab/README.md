# Create ML Models with BigQuery ML: Challenge Lab

lab: https://www.cloudskillsboost.google/focuses/14294?parent=catalog

### Task 1. Create a dataset to store your machine learning models

*Cloud Shell*

```shell=
bq mk <name>
```

~~<name>: challenge~~
    
### Task 2. Create a forecasting BigQuery machine learning model
    
*BigQuery Editor*
    
```sql=
CREATE OR REPLACE MODEL challenge.model
OPTIONS
  (model_type='linear_reg', labels=['duration_minutes']) AS
-- incorporate the starting station name, the hour the trip started, the weekday of the trip
SELECT
    start_station_name,
    EXTRACT(HOUR FROM start_time) AS start_hour,
    EXTRACT(DAYOFWEEK FROM start_time) AS day_of_week,
    duration_minutes
-- address of the start station labeled as location
    address AS location
-- duration for bike trips
FROM
    `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS trips
JOIN
    `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS stations
ON
    trips.start_station_name = stations.name
-- You must also use Training Year data only to train this model.
WHERE
    EXTRACT(YEAR FROM start_time) = 2017
    AND duration_minutes > 0
```

### Task 3. Create the second machine learning model

*BigQuery Editor*

```sql=
CREATE OR REPLACE MODEL challenge.model2
OPTIONS
  (model_type='linear_reg', labels=['duration_minutes']) AS
-- incorporate the starting station name, the bike share subscriber type and the start time for the trip
SELECT
    start_station_name,
    subscriber_type,
    EXTRACT(HOUR FROM start_time) AS start_hour,
    duration_minutes
FROM `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS trips
WHERE EXTRACT(YEAR FROM start_time) = 2017
```
    
### Task 4. Evaluate the two machine learning models
    
*BigQuery Editor*
 
```sql=
SELECT
  SQRT(mean_squared_error) AS rmse,
  mean_absolute_error
FROM
  ML.EVALUATE(MODEL challenge.model, (
-- first model features
  SELECT
    start_station_name,
    EXTRACT(HOUR FROM start_time) AS start_hour,
    EXTRACT(DAYOFWEEK FROM start_time) AS day_of_week,
    duration_minutes
  FROM
    `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS trips
  JOIN
   `bigquery-public-data.austin_bikeshare.bikeshare_stations` AS stations
  ON
    trips.start_station_name = stations.name
-- Evaluate each of the machine learning models against Evaluation Year data
  WHERE EXTRACT(YEAR FROM start_time) = 2020)
)
```
    
```sql=
SELECT
  SQRT(mean_squared_error) AS rmse,
  mean_absolute_error
FROM
  ML.EVALUATE(MODEL challenge.model2, (
-- second model features
  SELECT
    start_station_name,
    EXTRACT(HOUR FROM start_time) AS start_hour,
    subscriber_type,
    duration_minutes
  FROM
    `bigquery-public-data.austin_bikeshare.bikeshare_trips` AS trips
-- Evaluate each of the machine learning models against Evaluation Year data
  WHERE
    EXTRACT(YEAR FROM start_time) = 2020)
)
```

### Task 5. Use the subscriber type machine learning model to predict average trip durations
    
*BigQuery Editor*

```sql=
SELECT
  start_station_name,
  COUNT(*) AS trips
FROM
  `bigquery-public-data.austin_bikeshare.bikeshare_trips`
WHERE
  EXTRACT(YEAR FROM start_time) = 2020
GROUP BY
  start_station_name
ORDER BY
  trips DESC
```
    
```sql=
-- predict average trip length
SELECT AVG(predicted_duration_minutes) AS average_predicted_trip_length
FROM ML.predict(MODEL royalbike.subscriber, (
SELECT
    start_station_name,
    EXTRACT(HOUR FROM start_time) AS start_hour,
-- that uses subscriber_type as a feature
    subscriber_type,
    duration_minutes
FROM
  `bigquery-public-data.austin_bikeshare.bikeshare_trips`
WHERE 
--  from the busiest bike sharing station in Evaluation Year
  EXTRACT(YEAR FROM start_time) = 2020
-- subscriber type is Single Trip
  AND subscriber_type = 'Single Trip'
  AND start_station_name = '21st & Speedway @PCL'))
```