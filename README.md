# ChurnRates-sqlite
Calculating subscription churn rates of two segments using sqlite. 

Exploring the subscriptions table reveals that there are columns for id, subscription_start, subscription_end, and segment.

![1](https://user-images.githubusercontent.com/98862584/223104418-36e48065-ca66-407f-98cb-02ed6643f3d8.png)

Using MIN and MAX as follows will provide the range of months I will calculate the churn for.

SELECT MIN(subscription_start),
MAX(subscription_start) FROM subscriptions;

![2](https://user-images.githubusercontent.com/98862584/223109600-ce39355c-fd66-441b-b6ac-52bcfa0ff588.png)

Based on this information, I'll calculate churn rates for the first three months of 2017 - December won't be included as there are no subscription end values yet. 

First, I'll make a temporary table 'months' to use later so that I can see the subscriptions that are active for each month. 

WITH months AS
(SELECT 
  '2017-01-01' as first_day,
  '2017-01-31' as last_day
UNION
SELECT 
  '2017-02-01' as first_day,
  '2017-02-28' as last_day
UNION
SELECT 
  '2017-03-01' as first_day,
  '2017-03-31' as last_day
)

![monthstable](https://user-images.githubusercontent.com/98862584/223111139-8d51f839-03cb-41f6-bcf1-515dc7100cdb.png)

Then, I'll make a temporary table 'cross_join' from the subscriptions and months tables.

cross_join AS
(SELECT * FROM subscriptions
  CROSS JOIN months
) 

![cross_join](https://user-images.githubusercontent.com/98862584/223112027-b13fcde2-b839-4dd4-be84-df2dc2d7cd39.png)

The next temporary table needed will be called 'status,' that will allow us to compare users' active status between the two segments for each of the three months. This can be seen in the following snippet. Cases are used to find users from each segment who existed prior to the beginning of the month, returning 1 if true. Cases are also used to tell us if the user canceled their subscription during the month, which will return a 1 in is_canceled columns. 

status AS (
  SELECT 
  id,
  first_day AS month,
  CASE
    WHEN (subscription_start < first_day) AND 
   (subscription_end > first_day OR subscription_end IS null) AND (segment=87) THEN 1 
      ELSE 0 
  END AS is_active_87,
  CASE 
    WHEN (subscription_start < first_day) AND 
   (subscription_end > first_day OR subscription_end IS null)
    AND (segment=30) THEN 1   
      ELSE 0
  END AS is_active_30,
  CASE 
    WHEN (subscription_end BETWEEN first_day AND last_day) AND (segment=87)
    THEN 1 
      ELSE 0
  END AS  is_canceled_87,
  CASE 
    WHEN (subscription_end BETWEEN first_day AND last_day) AND (segment=30)
    THEN 1 
      ELSE 0
  END AS is_canceled_30
FROM cross_join)

Another temporary table, status_aggregate, will provide us with the sums of active and canceled subscriptions for each segment for each month.

status_aggregate AS
(SELECT month,
SUM(is_active_87) AS sum_active_87,
SUM(is_active_30) AS sum_active_30,
SUM(is_canceled_87) AS sum_canceled_87,
SUM(is_canceled_30) AS sum_canceled_30
FROM status
GROUP BY month
) SELECT * FROM status_aggregate;

![status_agg](https://user-images.githubusercontent.com/98862584/223114480-bc3eff60-916b-4253-a1bd-239c63288e9a.png)

Finally, we have our information in the right place to calculate the churn rates. 

SELECT month, 
1.0*sum_canceled_87/sum_active_87 AS churn_rate_87,
1.0*sum_canceled_30/sum_active_30 AS churn_rate_30
   from status_aggregate;

![churnrates](https://user-images.githubusercontent.com/98862584/223114902-12d1121d-ce85-4883-b918-679efbb987b6.png)

Based on the results, it is clear that segment 30’s churn rate is lower, therefore less users are unsubscribing. From this information, we can look at what segment 30 is doing successfully and what segment 87 could improve upon.  
