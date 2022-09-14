# Predict Soccer Match Outcomes with BigQuery ML: Challenge Lab

lab: https://www.cloudskillsboost.google/focuses/37320?parent=catalog

if lack table reference: https://www.cloudskillsboost.google/focuses/23114?parent=catalog
~~is use same dataset~~
### Task 1. Data ingestion

#### Navigation menu -> BigQuery

#### create table


| Field | Value |
| -------- | -------- |
| Source | Cloud Storage |
| Select file from Cloud Storage bucket | spls/bq-soccer-analytics/events.json |
| File format | JSONL(Newline delimited JSON) |
| Table name | Table name |
| Schema | Check the box marked Schema Auto detect |

| Field | Value |
| -------- | -------- |
| Source | Cloud Storage |
| Select file from Cloud Storage bucket | spls/bq-soccer-analytics/tags2name.csv |
| File format | CSV |
| Table name | Table name |
| Schema | Check the box marked Schema Auto detect |
	
### Task 2. Analyze soccer data

*BigQuery Editor*

```sql=
SELECT
playerId,
(Players.firstName || ' ' || Players.lastName) AS playerName,
COUNT(id) AS numPKAtt,
SUM(IF(101 IN UNNEST(tags.id), 1, 0)) AS numPKGoals,
SAFE_DIVIDE(
SUM(IF(101 IN UNNEST(tags.id), 1, 0)),
COUNT(id)
) AS PKSuccessRate
FROM
-- Join the events855
`soccer.events855` Events
-- with the players table 
LEFT JOIN
`soccer.players` Players ON
Events.playerId = Players.wyId
-- Filter on penalty kicks
WHERE
eventName = 'Free Kick' AND
subEventName = 'Penalty'
-- Group by player ID and player name
GROUP BY
playerId, playerName
-- Player should attempt at least 5 penalty kicks
HAVING
numPkAtt >= 5
-- Order by penalty kick success rate
ORDER BY
PKSuccessRate DESC, numPKAtt DESC



```

### Task 3. Gain insight by analyzing soccer data

*BigQuery Editor*

```sql=
WITH
Shots AS
(
SELECT
*,
/* 101 is known Tag for 'goals' from goals table */
-- Add an isGoal field by looking "inside" the tags field
(101 IN UNNEST(tags.id)) AS isGoal,
SQRT(
-- Calculate shot distance using the midpoint of the goal mouth ( 110 , 45 ) as the ending location
POW(
  (100 - positions[ORDINAL(1)].x) * 110/45,      
  2) +
-- Calculate pass distance by x-coordinate and y-coordinate differences, then convert to estimated meters using the average dimensions of a soccer field ( 110 x 54 )
POW(
  (60 - positions[ORDINAL(1)].y) * 110/54,        
  2)
) AS shotDistance
FROM
`soccer.events855`
WHERE

eventName = 'Shot' OR
(eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
)
SELECT
ROUND(shotDistance, 0) AS ShotDistRound0,
COUNT(*) AS numShots,
SUM(IF(isGoal, 1, 0)) AS numGoals,
AVG(IF(isGoal, 1, 0)) AS goalPct
FROM
Shots
WHERE
shotDistance <= 50
GROUP BY
ShotDistRound0
ORDER BY
ShotDistRound0


```


### Task 4. Create a regression model using soccer data

*BigQuery Editor*

```sql=
CREATE FUNCTION `soccer.GetShotDistanceToGoal855`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 /* Translate 0-100 (x,y) coordinate-based distances to absolute positions
 using "average" field dimensions of 110x54 before combining in 2D dist calc */
 SQRT(
   POW((110 - x) * 110/100, 2) +
   POW((45 - y) * 54/100, 2)
   )
 );
```

```sql=
CREATE FUNCTION `soccer.GetShotAngleToGoal855`(x INT64, y INT64)
RETURNS FLOAT64
AS (
 SAFE.ACOS(
   /* Have to translate 0-100 (x,y) coordinates to absolute positions using
   "average" field dimensions of 110x54 before using in various distance calcs */
   SAFE_DIVIDE(
     ( /* Squared distance between shot and 1 post, in meters */
       (POW(110 - (x * 110/100), 2) + POW(27 + (7.32/2) - (y * 54/100), 2)) +
       /* Squared distance between shot and other post, in meters */
       (POW(110 - (x * 110/100), 2) + POW(27 - (7.32/2) - (y * 54/100), 2)) -
       /* Squared length of goal opening, in meters */
       POW(7.32, 2)
     ),
     (2 *
       /* Distance between shot and 1 post, in meters */
       SQRT(POW(110 - (x * 110/100), 2) + POW(27 + 7.32/2 - (y * 54/100), 2)) *
       /* Distance between shot and other post, in meters */
       SQRT(POW(110 - (x * 110/100), 2) + POW(27 - 7.32/2 - (y * 54/100), 2))
     )
    )
  /* Translate radians to degrees */
  ) * 180 / ACOS(-1)
 )
;
```

```sql=
CREATE MODEL `soccer.xg_logistic_reg_model_855`                       
OPTIONS(
model_type = 'LOGISTIC_REG',
input_label_cols = ['isGoal']
) AS
SELECT
Events.subEventName AS shotType,
(101 IN UNNEST(Events.tags.id)) AS isGoal,
`soccer.GetShotDistanceToGoal855`(Events.positions[ORDINAL(1)].x,       
Events.positions[ORDINAL(1)].y) AS shotDistance,
`soccer.GetShotAngleToGoal855`(Events.positions[ORDINAL(1)].x,      
Events.positions[ORDINAL(1)].y) AS shotAngle
FROM
`soccer.events855` Events
LEFT JOIN
`soccer.matches` Matches ON
Events.matchId = Matches.wyId
LEFT JOIN
`soccer.competitions` Competitions ON
Matches.competitionId = Competitions.wyId
WHERE
Competitions.name != 'World Cup' AND
(
eventName = 'Shot' OR
(eventName = 'Free Kick' AND subEventName IN ('Free kick shot', 'Penalty'))
)
;


```

### Task 5. Make predictions from new data with the BigQuery model

*BigQuery Editor*

```sql=
SELECT
predicted_isGoal_probs[ORDINAL(1)].prob AS predictedGoalProb,
* EXCEPT (predicted_isGoal, predicted_isGoal_probs),
FROM
ML.PREDICT(
MODEL `soccer.xg_logistic_reg_model_855`,                                   
(
SELECT
   Events.playerId,
   (Players.firstName || ' ' || Players.lastName) AS playerName,
   Teams.name AS teamName,
   CAST(Matches.dateutc AS DATE) AS matchDate,
   Matches.label AS match,
   CAST((CASE
     WHEN Events.matchPeriod = '1H' THEN 0
     WHEN Events.matchPeriod = '2H' THEN 45
     WHEN Events.matchPeriod = 'E1' THEN 90
     WHEN Events.matchPeriod = 'E2' THEN 105
     ELSE 120
     END) +
     CEILING(Events.eventSec / 60) AS INT64)
     AS matchMinute,
   Events.subEventName AS shotType,
   (101 IN UNNEST(Events.tags.id)) AS isGoal,

   `soccer.GetShotDistanceToGoal855`(Events.positions[ORDINAL(1)].x,         
    Events.positions[ORDINAL(1)].y) AS shotDistance,
   `soccer.GetShotAngleToGoal855`(Events.positions[ORDINAL(1)].x,             
    Events.positions[ORDINAL(1)].y) AS shotAngle
FROM
   `soccer.events855` Events
LEFT JOIN
   `soccer.matches` Matches ON
    Events.matchId = Matches.wyId
LEFT JOIN
   `soccer.competitions` Competitions ON
    Matches.competitionId = Competitions.wyId
LEFT JOIN
   `soccer.players` Players ON
    Events.playerId = Players.wyId
LEFT JOIN
   `soccer.teams` Teams ON
    Events.teamId = Teams.wyId
WHERE
   Competitions.name = 'World Cup' AND(
     eventName = 'Shot' OR
     (eventName = 'Free Kick' AND subEventName IN ('Free kick shot'))
   ) AND
   (101 IN UNNEST(Events.tags.id))
))
ORDER BY
predictedgoalProb


```







