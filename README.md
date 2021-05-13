# SQL Challenge Submissions

## Solution 1 - Compute these two metrics: sum of transactions per day, and count of distinct addresses that initiated a transaction (i.e. the “from” side).

### Part 1

```sql
SELECT date_trunc('days', et.block_timestamp) AS date,
    SUM(et.amount) AS total_amount
FROM ethereum_transactions AS et
GROUP BY 1
ORDER BY 1 ASC;
```

### Part 2

```sql
SELECT COUNT(DISTINCT et.address_from) AS num_unique_initiating_addresses
FROM ethereum_transactions AS et;
```

## Solution 2 - Compute the sum of transactions per day by label type.

```sql
SELECT date_trunc('days', et.block_timestamp) AS date,
    al.label_category AS label_type,
    SUM(et.amount) AS total_label_type_amount
FROM ethereum_transactions AS et
    LEFT JOIN address_labels AS al 
    -- Makes a row with transaction and address data for both the from and to addresses in a transaction
    ON (et.address_from = al.address)
    OR (et.address_to = al.address)
GROUP BY 1,
    2
ORDER BY 1,
    2 ASC;
```

## Solution 3 - Compute the USD denominated volume on Ethereum per day.

```sql
WITH converted_price_time_data AS (
    SELECT 
        -- Converts the String minute_price_candles timestamp field to a Datetime timestamp type
        CONVERT(DATETIME, mpc.timestamp, 102) AS converted_timestamp,
        /* Finds the subsequent candle start date for each candle by asset_id (defaulting to the
        current date for the last candle which will not have a following candle) */
        LEAD(converted_timestamp, 1, getdate()) OVER (
            PARTITION BY mpc.asset_id
            ORDER BY converted_timestamp ASC
        ) AS following_converted_timestamp,
        -- Calculates Typical Price for each candle
        (
            (mpc.price_high + mpc.price_low + mpc.price_close) / 3
        ) AS typical_candle_price,
        mpc.asset_id AS asset_id
    from minute_price_candles mpc
),
SELECT date_trunc('days', et.block_timestamp) AS date,
SUM(et.amount * cptd.typical_candle_price) AS total_usd_volume
FROM ethereum_transactions AS et
    LEFT JOIN converted_price_time_data AS cptd 
    -- Joins on the relationship that each transaction is in only one candle
    ON (et.tx_asset_id = cptd.asset_id)
    AND (
        et.block_timestamp >= cptd.converted_timestamp
        and et.block_timestamp < cptd.following_converted_timestamp
    ) 
-- Filters out all volume not on the Ethereum blockchain
WHERE et.blockchain = 'ethereum'
GROUP BY 1;
```

## Solution 4 - Identify the top 10 addresses per day ranked by transaction count.

```sql
WITH ranked_transaction_data AS (
    SELECT date_trunc('days', et.block_timestamp) AS date,
        -- Finds the rank of the top 10 addresses grouped by date
        RANK() OVER (
            PARTITION BY date,
            al.address
            ORDER BY COUNT(DISTINCT et.tx_hash)
        ) AS day_rank,
        al.address AS address
    FROM ethereum_transactions AS et
        LEFT JOIN address_labels AS al 
        -- Makes a row with transaction and address data for both the from and to addresses in a transaction
        ON (et.address_from = al.address)
        OR (et.address_to = al.address) 
    -- Filters all but the top 10 addresses per day
    WHERE day_rank <= 10
),
select rtd.date,
rtd.day_rank,
rtd.address
from ranked_transaction_data AS rtd
GROUP BY 1,
    2,
    3
ORDER BY 1,
    2 ASC;
```
