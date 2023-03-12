#  ASK
## Defining a key question
The marketing team concluded that prioritizing the number of annual members is the key to future growth. The strategy is to however convert the casual riders into members instead of trying to appeal to new audience.
To achieve that we're asked to find out how annual members and casual riders differ. We have a historical trip data to find an answer to this question. 
## The business task
Analyse Cyclistic's historical trip data in order to identify trends which would help the design team to develop a converting strategy.

# Prepare
The historical trip data has the following structure:
```
ride_id,
rideable_type,
started_at,
ended_at,
start_station_name,
start_station_id,
end_station_name,
end_station_id,
start_lat,
start_lng,
end_lat,
end_lng,
member_casual
```
## Features and limitations 
The data presented gives us information on the number of rides taken, the type of bike used, the timestamps of start and the end of the ride, the coordinates and the status
of the rider. The data doesn't come from a third party, is internal and is consistent in its schema. 

What's crucial to understand is that the number of rides doesn't represent the number of riders.
In short: we can later see that the number of rides done by annual members is higher than this of casual riders but when we take into consideration that annual members may use the bike-sharing service for
everyday commute it might be that the number of casuals is higher. In any case, with the data presented it's impossible to make any conclusions on the actual number of people in either group. We can however try to identify trends within 
the groups and try to find the similarities and differences only in the context of data we have. 

I imported the data first to my Google Drive and trimmed whitespaces in Sheets before importing the data to Big Query, I made sure to have the conventional naming in all files with MMM_YYYY style.
ride_id the primary ley of all tables iis unique, therefore the data contains no duplicates.

# Process
## Combining the tables
Since the tables have the same schema it's possible to use UNION ALL.
```
CREATE TABLE trip_data.year_trip_data AS
SELECT * 
FROM (
  SELECT * FROM `trip_data.FEB_2022`
  UNION ALL
  SELECT * FROM `trip_data.MAR_2022`
  UNION ALL
  ... 
  SELECT * FROM `trip_data.JAN_2023`
)
```
Now we have the complete dataset with all the historical data in one. We cannot call it clean however, to determine what should remain and what should go let's look at our columns.
## rideable_type 
there are 3 values in this column "classic_bike", "electric_bike" and "docked" bike. I was confused about the docked_bike and decided to check the company's website to see whether I can find any answers there.
Unfortunately, both classic and electric bikes have a docking station so I didn't want to rush an assumption that docked bike is a classic or an electric bike. I'm not going to filter out the docked_bike out for now. But I will filter it out when I'll analyse the trends in bike preferences later on.
## started_at, ended_at
These are the timestamps crucial for our analysis and one of a main criteria for cleaning our data. After checking the pricicng on the company's websute I concluded that the appropriate length of the ride should be longer than 1 minute (shorter than that might just be maintanance checkss, false rides etc.) and shorter than 12 hours,
The pricing is very explicit about rides being charged 0.17 cents per minute after 3 hours and though it's against company's policy to rent a bike for longer than 24 hours I feel as if even the 12 hour ride is a stretch.

In order to filter the rides we'll use the following syntax
```
SELECT
DISTINCT * 
FROM `year_trip_data`
WHERE timestamp_diff(ended_at, started_at, minute) >=1
AND timestamp_diff(ended_at, started_at, hour) <= 12
```
At this point we can already run a simple calculation to find the mean ride_length of both casual and member rides with the following 
```
SELECT
AVG(timestamp_diff(ended_at, started_at, minute)), member_casual 
FROM `clean_year_trip_data` 
GROUP BY member_casual
```
| AVG_ride_length  | member_casual |
| ------------- | ------------- |
| 19.102563     | casual  |
| 11.976172     | member  |

From what we can see on average casual rides last longer than those of annual members.

Just from the timestamp column we'll also be able to determine the month, workdays/weekends and part of the day, more on this in the next section

## station_names and station_IDs
A string column which shows the station names and their IDs. A tricky column, some of the values are missing and not all of the columns have the appropriate naming conventions. The classic bikes should be docked at the station and ebikes can also be docked at the station as well as left on the rack within the service area.
In short: this section can be filtered with the following syntax:
```
SELECT COUNT(ride_id) FROM `clean_year_trip_data` 
WHERE rideable_type NOT IN ("electric_bike")
AND start_station_name IS NULL
```
The query showed 0 instances when the rides with classic or docked type didn't start at the station and only 62 of them didn't end at the station.  
## Coordinates 
Self-explaantory, coordinates can be used to find the most active areas when we visualise them on the map.

# Analyse 
## Notes on analysis 
Keeping in mind that the rides do not represent the individual customers comparing them in one area won't yield a great result. For example if we would like to see which category is more likely
to use the bikes during night having the overall number of night rides and calculating the percentage of members and casuals would lead to false insights. That's why I think it would be wiser to run calculations inside a group for the most part and then look at the results independently.

## Ride_length 
We've covered the average ride_length of both groups before and found out that on average casual rides last longer (19 minutes) the possible insight is that casual riders might use the bike-sharing for leisure and more prolonged trips, not only as the mode of transportation. It could also be suggested that the annnual members use bikes for the commute.

## Activity through the year
To find high and low seasons overall I'm going to use the following syntax:
```
SELECT 
 m_total_trips,
 c_total_trips,
 m_total_trips + c_total_trips AS total_trips,
 ROUND(c_total_trips/(m_total_trips+c_total_trips),2)*100 AS casual_percentage,
 ROUND(m_total_trips/(m_total_trips+c_total_trips),2)*100 AS member_percentage
  FROM (
  SELECT 
  COUNTIF((EXTRACT(MONTH from started_at) BETWEEN 1 AND 12) AND member_casual = 'member') AS m_total_trips,
  COUNTIF((EXTRACT(MONTH from started_at) BETWEEN 1 AND 12) AND member_casual = 'casual') AS c_total_trips,
  FROM `clean_year_trip_data`
  GROUP BY EXTRACT(month from started_at) 
  ORDER BY MIN(EXTRACT(MONTH from started_at))
)
```
The results of the query showed that both casual and member rides follow the similar trend, where the peak activity falls on summer and winter stands for the least amount of rides with the lowest number of casual rides and their percentage. 
There is a slight difference however. The drop in activity during summer period starts earlier for casual riders than for members.
### Key insight (month)
Both groups show similar trends. The strategy to convert casual members should be put in action during July as it is the peak activity month for this group. The marketing campaign "never-ending summer" with the routes and perks for annual members might incentivise casual riders to convert.
## Riders' activity by part of day
To learn the difference between the daily usage of bikes by both groups I suggest running the following query: 
```
SELECT 
  total_trips,
  total_member_trips,
  total_casual_trips,
  total_morning_trips,
  total_afternoon_trips,
  total_evening_trips,
  total_night_trips,
  (total_morning_trips + total_afternoon_trips + total_evening_trips+total_night_trips),

  ROUND(total_morning_trips/total_trips,2)*100 AS morning_percentage,
  ROUND(total_afternoon_trips/total_trips,2)*100 AS afternoon_percentage,
  ROUND(total_evening_trips/total_trips,2)*100 AS evening_percentage,
  ROUND(total_night_trips/total_trips,2)*100 AS night_percentage,

  ROUND(m_total_morning_trips/total_member_trips,2)*100 AS morning_member_percentage,
  ROUND(m_total_afternoon_trips/total_member_trips,2)*100 AS afternoon_member_percentage,
  ROUND(m_total_evening_trips/total_member_trips,2)*100 AS evening_member_percentage,
  ROUND(m_total_night_trips/total_member_trips,2)*100 AS night_member_percentage,

  ROUND(c_total_morning_trips/total_casual_trips,2)*100 AS morning_casual_percentage,
  ROUND(c_total_afternoon_trips/total_casual_trips,2)*100 AS afternoon_casual_percentage,
  ROUND(c_total_evening_trips/total_casual_trips,2)*100 AS evening_casual_percentage,
  ROUND(c_total_night_trips/total_casual_trips,2)*100 AS night_casual_percentage
FROM 
  (
  SELECT
  COUNT(ride_id) AS total_trips,
  COUNTIF(member_casual = 'member') AS total_member_trips,
  COUNTIF(member_casual = 'casual') AS total_casual_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 05 AND 11) AND (EXTRACT(MINUTE from started_at) BETWEEN 00 AND 59)) AS total_morning_trips,
  COUNTIF(EXTRACT(HOUR from started_at) BETWEEN 12 AND 16) AS total_afternoon_trips,
  COUNTIF(EXTRACT(HOUR from started_at) BETWEEN 17 AND 21) AS total_evening_trips,
  COUNTIF(EXTRACT(HOUR from started_at) BETWEEN 22 AND 23) + COUNTIF(EXTRACT(HOUR from started_at) BETWEEN 00 AND 04) AS total_night_trips,

  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 05 AND 11) AND (EXTRACT(MINUTE from started_at) BETWEEN 00 AND 59) AND member_casual = 'member') AS m_total_morning_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 12 AND 16) AND member_casual = 'member') AS m_total_afternoon_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 17 AND 21) AND member_casual = 'member') AS m_total_evening_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 22 AND 23) AND member_casual = 'member') + COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 00 AND 04) AND member_casual = 'member') AS m_total_night_trips,

  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 05 AND 11) AND (EXTRACT(MINUTE from started_at) BETWEEN 00 AND 59) AND member_casual = 'casual') AS c_total_morning_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 12 AND 16) AND member_casual = 'casual') AS c_total_afternoon_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 17 AND 21) AND member_casual = 'casual') AS c_total_evening_trips,
  COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 22 AND 23) AND member_casual = 'casual') + COUNTIF((EXTRACT(HOUR from started_at) BETWEEN 00 AND 04) AND member_casual = 'casual') AS c_total_night_trips,

  FROM
  `clean_year_trip_data`
  )
  ```
  
  This query gives us explicit information on how annual members and casual riders us Cyclistic's bikes throughout the day. The results of the query showed that annual members use bikes from morning until evening, while casual riders are more likely to take night trips and less active in the morning compared to annual members.
  ### Key insight (part of day) 
  Should develop a marketing strategy during afternoon and evening hours(peak activity time of casual riders) as well as advertise morning rides (morning rides stand for 28 percent of all rides for the casuals)
  
  ## Workdays/weekends 
  To find out whether casual riders are consistently active through the whole week or prefer to use bikes more during weeekends I suggest running this query:
  ```
  SELECT 
 m_total_weekend_trips,
 m_total_workday_trips,
 c_total_workday_trips,
 c_total_weekend_trips,
 m_total_weekend_trips + c_total_weekend_trips AS total_weekend_trips,
 m_total_workday_trips + c_total_workday_trips AS total_workday_trips,
 ROUND(c_total_weekend_trips/(m_total_weekend_trips+c_total_weekend_trips),2)*100 AS casual_weekend_percentage,
 ROUND(m_total_weekend_trips/(m_total_weekend_trips+c_total_weekend_trips),2)*100 AS member_weekend_percentage,
 ROUND(c_total_workday_trips/(m_total_workday_trips+c_total_workday_trips),2)*100 AS casual_workday_percentage,
 ROUND(m_total_workday_trips/(m_total_workday_trips+c_total_workday_trips),2)*100 AS member_workday_percentage
 
  FROM (
  SELECT 
  COUNTIF((EXTRACT(DAYOFWEEK from started_at) = 7 OR (EXTRACT(DAYOFWEEK from started_at) = 1) AND member_casual = 'member')) AS m_total_weekend_trips,
  COUNTIF((EXTRACT(DAYOFWEEK from started_at) BETWEEN 2 AND 6) AND member_casual = 'member') AS m_total_workday_trips,
  COUNTIF((EXTRACT(DAYOFWEEK from started_at) = 7 OR (EXTRACT(DAYOFWEEK from started_at) = 1) AND member_casual = 'casual')) AS c_total_weekend_trips,
  COUNTIF((EXTRACT(DAYOFWEEK from started_at) BETWEEN 2 AND 6) AND member_casual = 'casual') AS c_total_workday_trips,
  
  FROM `cyclists-case-study-378616.trip_data.clear_year_trip_data`
  
)
```
We can see the overall percentages in this query but the key observation here is in the number of casual rides during workdays and during weekends. 

The query showed us that during the working week (5 days) there were 1338943 rides and during weekends (2 days) there were 1203974 rides. It is a major discovery that as almost half of all rides done by casuals falls on weekends. Therefore, weekends are peak activity time for casual riders.

As for annual members the results are differ drastically: workdays - 2497442 rides, weekends - 1238493 rides. Annual members are active throughout the whole week.
  ### Key Insight (workdays/weekends)
  Casual riders are much more likely to use Cyclistic's bikes on weekends while annual members are active all week. Should develop a strategy to raise the activity of casual riders so they would see the potential in using the service during the workdays as well.

## Bike preference
One more query you can run to see whether there is a preference towards a partucular bike type you can run the following query: 
```
SELECT 
  total_member_trips,
  total_casual_trips
  total_member_electric_trips,
  total_casual_electric_trips,
  total_member_classic_trips,
  total_casual_classic_trips,
  ROUND(total_member_electric_trips/total_member_trips,2)*100 AS member_electric_percentage,
  ROUND(total_casual_electric_trips/total_casual_trips,2)*100 AS casual_electric_percentage,
  ROUND(total_member_classic_trips/total_member_trips,2)*100 AS member_classic_percentage,
  ROUND(total_casual_classic_trips/total_casual_trips,2)*100 AS casual_classic_percentage
FROM 
  (
  SELECT
  COUNTIF(member_casual = 'member') AS total_member_trips,
  COUNTIF(member_casual= 'casual') AS total_casual_trips,
  COUNTIF(rideable_type = 'electric_bike' AND member_casual = 'member') AS total_member_electric_trips,
  COUNTIF(rideable_type = 'electric_bike' AND member_casual = 'casual') AS total_casual_electric_trips,
  COUNTIF(rideable_type = 'classic_bike' AND member_casual = 'member') AS total_member_classic_trips,
  COUNTIF(rideable_type = 'classic_bike' AND member_casual = 'casual') AS total_casual_classic_trips
  FROM
  `cyclists-case-study-378616.trip_data.clear_year_trip_data`
  WHERE member_casual NOT IN ('docked_bike')
  )
  ```
  Having no access to the initial number of bikes total and their accessibility makes it difficult however to gain any valuable insight on this topic.
  ### key insight (rideable_type)
  Should investigate in the future whether the rideable type makes a difference in customer's decision making. Requires an additional data on the number of bikes in total and their accessibility 
