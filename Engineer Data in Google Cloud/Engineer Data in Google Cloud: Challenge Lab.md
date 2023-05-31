# Engineer Data in Google Cloud: Challenge Lab

## Challenge scenario

You have started a new role as a Data Engineer for TaxiCab Inc. You are expected to import some historical data to a working BigQuery dataset, and build a basic model that predicts fares based on information available when a new ride starts. Leadership is interested in building an app and estimating for users how much a ride will cost. The source data will be provided in your project.

You are expected to have the skills and knowledge for these tasks, so don't expect step-by-step guides to be provided.

## Your challenge

As soon as you sit down at your desk and open your new laptop you receive your first assignment: build a basic BQML fare prediction model for leadership. Perform the following tasks to import and clean the data, then build the model and perform batch predictions with new data so that leadership can review model performance and make a go/no-go decision on deploying the app functionality.

---

1. Clean your training data

    ```sql
    CREATE OR REPLACE TABLE
    `taxirides.taxi_training_data_909`
    AS (
    SELECT
        pickup_datetime
        , pickup_longitude AS pickuplon
        , pickup_latitude AS pickuplat
        , dropoff_longitude AS dropofflon
        , dropoff_latitude AS dropofflat
        , passenger_count AS passengers
        , (tolls_amount + fare_amount) AS fare_amount_264
    FROM
        `taxirides.historical_taxi_rides_raw`
    WHERE
        trip_distance > 3
        AND (tolls_amount + fare_amount) > 2.5
        AND pickup_longitude > -75
        AND pickup_longitude < -73
        AND dropoff_longitude > -75
        AND dropoff_longitude < -73
        AND pickup_latitude > 40
        AND pickup_latitude < 42
        AND dropoff_latitude > 40
        AND dropoff_latitude < 42
        AND passenger_count > 3
        AND RAND() < 999999 / (SELECT COUNT(*) FROM `taxirides.historical_taxi_rides_raw`)
    )
    ```

1. Create a BigQuery ML model

    Train:

    ```sql
    CREATE OR REPLACE MODEL
    `taxirides.fare_model_454`
    OPTIONS(
    model_type='linear_reg'
    , labels=['fare_amount_264']
    ) AS (
    SELECT
        *
        , ST_Distance(
        ST_GeogPoint(pickuplon, pickuplat)
        , ST_GeogPoint(dropofflon, dropofflat)
        ) AS euclidean
    FROM
        `taxirides.taxi_training_data_909`
    )
    ```

    Evaluate:

    ```sql
    SELECT
    SQRT(mean_squared_error) AS rmse
    FROM
    ML.EVALUATE(
        MODEL `taxirides.fare_model_454`
        , (
        SELECT
            *
            , ST_Distance(
            ST_GeogPoint(pickuplon, pickuplat)
            , ST_GeogPoint(dropofflon, dropofflat)
            ) AS euclidean
        FROM
            `taxirides.taxi_training_data_909`
        )
    )
    ```

1. Perform a batch prediction on new data

    ```sql
    CREATE OR REPLACE TABLE
    `taxirides.2015_fare_amount_predictions`
    AS (
    SELECT
        *
    FROM
        ML.PREDICT(
        MODEL `taxirides.fare_model_454`
        , (
            SELECT
            *
            , ST_Distance(
                ST_GeogPoint(pickuplon, pickuplat)
                , ST_GeogPoint(dropofflon, dropofflat)
            ) AS euclidean 
            FROM
            `taxirides.report_prediction_data`
        )
        )
    )
    ```