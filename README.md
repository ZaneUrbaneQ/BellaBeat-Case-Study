# CASE STUDY: Fitness device usage trends for BellaBeat
##### Author: Zane Urbane
##### Date: June 22, 2024

- ### [Case Scenario](#case-scenario)
- ### [Business Task (Ask)](#cusiness-task-ask)
- ### [About the Data (Prepare)](#about-the-data-prepare)
- ### [Data Preparation (Process)](#data-preparation-process)
- ### [Data Exploration (Analyze)](#data-exploration-analyze)
- ### [Data Visualization (Share)](#data-visualization-share)
- ### [Conclusions and Recommendations (Act)](#conclusions-and-recommendations-act)

## Case scenario
Bellabeat, founded in 2013 by Urška Sršen and Sando Mur, creates health-focused smart products for women. Their product line includes the Bellabeat app, Leaf wellness tracker, Time wellness watch, and Spring hydration bottle. Bellabeat combines artistic design with advanced technology to provide insights into activity, sleep, stress, and reproductive health.

## Business Task (Ask)
Bellabeat aims to analyze smart device usage data to understand how consumers use their products. This analysis will inform strategic marketing decisions aimed at enhancing user engagement, optimizing product offerings, and expanding market reach.
Key stakeholders include Urška Sršen and Sando Mur, co-founders of Bellabeat, as well as the marketing analytics team responsible for analyzing and interpreting consumer data.

## About the Data (Prepare)
Data Source: FitBit Fitness Tracker Data from Mobius: https://www.kaggle.com/arashnic/fitbit

This Kaggle dataset comprises personal fitness tracker data from thirty Fitbit users. These participants consented to provide their personal tracker data, which includes minute-level output for physical activity, heart rate, and sleep monitoring. The dataset features information on daily activity, steps, and heart rate, allowing for an exploration of users' habits. It contains 18 CSV files organized in long format.

ROCCC analysis and dataset limitations:
Reliable: Low - The dataset includes only 30 participants, which is a small sample size and does not accurately represent the broader population of female Fitbit users. This small sample size introduces significant bias and reduces the reliability of any conclusions drawn from the data.
Original: Low — The data being collected from Amazon Mechanical Turk indicates that it is sourced from a third party. 
Comprehensive: Low — The dataset lacks key demographic information such as gender, age, and health conditions. 
Current: Low — The data is approximately 8 years old, having been collected in 2016.
Cited: Low — Data obtained from an unidentified third party (Amazon Mechanical Murk).

## Data preparation (Process)
For data exploration - data uplods, manipulation, preparation, and basic analysis, R and RStudio Cloud were used. All the code can be found in *bellabeat_case_study.Rmd* in this repository.
Two daily ata sets were explored: dailyActivity_merged.csv and sleepDay_merged.csv.
And three hourly data sets were look at: hourlyIntensities_merged.csv, hourlySteps_merged.csv, hourlyCalories_merged.csv.
After preperation files were combined in daily data and hourly data.

All uploded data was checked for NA or duplicate values. And one found and removed.
```
sum(is.na(daily_sleep))
sum(duplicated(daily_sleep))
daily_sleep <- daily_sleep[!duplicated(daily_sleep), ]
```
Daily activities and daily sleep records where explored. There are more participants in daily activity - 33 than in daily sleep records - 24.
``` r
n_distinct(daily_activity$Id)
n_distinct(daily_sleep$Id)
```
Column ActivityDate is recognized as a character (chr) type, so it needs to be changed to a date. And the Id column should be nominal type, as it is used as an index, not a numeric value. These changes had to be made for all the datasets.
``` r
daily_activity$ActivityDate <- as.Date(daily_activity$ActivityDate, format = "%m/%d/%Y")
daily_activity$Id <- as.character(daily_activity$Id)
```
For all datasets a colomn with weekdays was aded to see if it significant dimension.
``` r
daily_activity$Weekday <- weekdays(daily_activity$ActivityDate)
```
Merge daily activities data with sleep data for analysis together. In daily activity data there are 940 rows, but in sleep data only 410 rows. I used left join to keep all data and explore it later.
``` r
comb_daily_data <- left_join(daily_activity, daily_sleep, by= c("Id","ActivityDate" = "SleepDay"))
#Rename ActivityDate column as Date
colnames(comb_daily_data)[colnames(comb_daily_data) == "ActivityDate"] <- "Date"
#In thata set is 2650 NA values. NA is only in merged data in sleep column.
#Replace it with zero for calculations.
comb_daily_data[is.na(comb_daily_data)] <- 0
```

## Data Exploration (Analyze)
For begining I tried to understand how Fitbit users used the tracker. Did them used it all day long or only when excersising.There is 1440 min in a day so I will check thas data in time columns sum up or not.
``` r
comb_daily_data <- mutate(comb_daily_data, TotalMinutes = VeryActiveMinutes + FairlyActiveMinutes + LightlyActiveMinutes + SedentaryMinutes + TotalTimeInBed)
count_1440_minutes <- sum(comb_daily_data$TotalMinutes == 1440)
print(count_1440_minutes)
```
604 of 940 records include all day.
``` r
ggplot(comb_daily_data, aes(x = Date, y = Id, fill = TotalMinutes)) +
  geom_tile() +
  scale_fill_gradient(low = "white", high = "blue") +
  labs(title = "Heatmap of Total Minutes by ID and Date",
       x = "Date", y = "ID", fill = "Total Minutes") +
  theme_minimal()
```
![ggplot_1](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/e3b77438-cb74-46ed-bf61-e48fa83589a2)
There are users that have not used Fitbit tracker for all the period. It looks like that some minuts don't go to right date. That could be because the sleep data is daily and could refer to 2 dates.From heat map looks like last date could be incompleat.
Lets look at average minutes per day.
``` r
Id_activity <- comb_daily_data %>%
  group_by(Id) %>%
  summarise(
    total_dates = n_distinct(Date),
    total_minutes = sum(TotalMinutes)
  ) %>%
  mutate(average_total_minutes = total_minutes / total_dates)%>%
  arrange(desc(average_total_minutes))
```
Average minutes for day is from 1322 to 1439 so sleeping time accuracy could be the cases.
And I want to see how looks those records, that don't have daily sleep records.
``` r
comb_data_no_sleep <- comb_daily_data[comb_daily_data$TotalSleepRecords == 0,]
count_1440_minutes_no_sleep <- sum(comb_data_no_sleep$TotalMinutes == 1440)
print(count_1440_minutes_no_sleep)
```
478 from 530 records are 1440 min whitout sleep time.Looks like sleeping time is added to SedentaryMinutes.Maybe users needed to add it manually.
``` r
ggplot(data=comb_data_no_sleep, aes(x=TotalMinutes, y=SedentaryMinutes)) + geom_point()
```
![ggplot_2](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/c16fb60d-aeee-4d30-9906-0daeebdd8426)
For daily activities analysis I will use only data that includes sleeping data. And add colums with active time and laying in bed time.
``` r
daily_analysis_data <- comb_daily_data %>%
  filter(TotalSleepRecords > 0)

daily_analysis_data <- mutate(daily_analysis_data, SomeActivity = VeryActiveMinutes + FairlyActiveMinutes + LightlyActiveMinutes)

daily_analysis_data <- mutate(daily_analysis_data, LayingMinutes = TotalTimeInBed - TotalMinutesAsleep)
```
More active minutes more steps.
``` r
ggplot(data=daily_analysis_data, aes(x=TotalSteps, y=SomeActivity)) + geom_smooth(method = "loess") + 
  labs(title = "Comparison of Steps VS Activity Time", x = "Total Steps", y = "Activity Time")
```
![ggplot_3](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/13f6f9c3-daa1-4973-851b-bd36af1fc0de)

I will look at summary data about how participiants spend they time.
``` r
sums_act <- colSums(daily_analysis_data[, c("VeryActiveMinutes", "FairlyActiveMinutes", "LightlyActiveMinutes", "SedentaryMinutes", "TotalMinutesAsleep", "LayingMinutes")])

total_sum_act <- sum(sums_act)

percentages_sum_act <- (sums_act / total_sum_act) * 100

avg_act <- colMeans(daily_analysis_data[, c("VeryActiveMinutes", "FairlyActiveMinutes", "LightlyActiveMinutes", "SedentaryMinutes", "TotalMinutesAsleep", "LayingMinutes")])

sum_daily_activites <- data.frame(
  Activity = names(sums_act),
  TotalTime = as.numeric(sums_act),
  PercentageOfTotal = percentages_sum_act,
  AvgTime = as.numeric(avg_act)
)
```
Most of the time is spent as sedentary minutes.
``` r
ggplot(data=sum_daily_activites, aes(x=Activity, y=AvgTime))+
  geom_bar(stat="identity")+
  labs(title="Average Time in day for each activity")+
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```
![ggplot_4](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/6ae2814e-5023-4ab9-a1a9-a9e5da88b5af)

And Explore activities by weekdays.
``` r
ggplot(avg_by_weekday_long, aes(x = Weekday, y = Minutes, fill = ActivityType)) +
  geom_bar(stat = "identity") +
  labs(title = "Average Activity Minutes by Weekday",
       x = "Weekday",
       y = "Average Minutes",
       fill = "Activity Type") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```
![ggplot_5](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/5aae9e75-b08f-455e-aacc-42dc3e81ed8b)

Data was prepared for eseier visualization making in Tableau and exported as csv.

## Data Visualization (Share)
### [BellaBeat Data Analysis Dashboard in Tableau](https://public.tableau.com/app/profile/zane.urbane/viz/BellabeatCaseStudy_17190633010800/Bellabeat)
![Bellabeat](https://github.com/ZaneUrbaneQ/BellaBeat-Case-Study/assets/173494641/9eaac7ce-8d20-49c3-912f-f51d3c47b997)


## Conclusions and Recommendations (Act)
Analysis of smart device usage data reveals the following insights:
-  74.3% of awake time users spend in sedentary positions.
-  Very active minutes average only 21-31 minutes daily.
-  The average sleep duration is 7 hours per day.
-  Activity patterns are consistent throughout the weekdays, with Saturday emerging as the most active day and Sunday showing the least activity.
-  Peak steps occur between 17:00 to 19:00 and 12:00 to 14:00..

Recommendations based on the data:
- Devices should minimize manual entries to ensure data accuracy for personalized customer insights.
- Implement reminders for movement, emphasizing precise exercises.
- Include sleep reminders to approach the recommended 8 hours, synchronized with morning alarm times.
- BellaBeat could introduce an educational program coupled with a point-based incentive system.
- Enhance analysis accuracy by expanding participant data, duration, and detailed user information for precise segmentation.

