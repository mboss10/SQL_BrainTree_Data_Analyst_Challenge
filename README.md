# SQL BrainTree Data Analyst Challenge

I am always up for a challenge, especially when it is data related, and even more when it is required to use SQL to go through it!!

## Context

Alexander Connelly, shared in his [Github repository](https://github.com/AlexanderConnelly/BrainTree_SQL_Coding_Challenge_Data_Analyst) what is/used to be a Data Analyst exercise given during interviews at Braintree. <br><br>
It is composed of a series of 7 questions to answer using a dataset with continents, countries and GDP per capita data. Below, I will share my SQL code and comments to answer each question as best as I can.

## Dataset
Let's review each .csv file provided for that challenge.
### countries.csv
#### Definition
The file contains the list of countries: 252 rows, 2 columns `country_code` and `country_name`
#### Preview
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/countries.png" width="400"> 

Full file is available here: [countries.csv](https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/countries.csv)

### continents.csv
#### Definition
The file contains the list of continents: 7 rows, 2 columns `continent_code` and `continent_name`
#### Preview
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/continents.png" width="400">

Full file is available here: [continent.csv](https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/continents.csv)

### per_capita.csv
#### Definition
The file contains the Growth Domestic Product per Capita for each country for each year from 2004 to 2012: 2079 rows, 3 columns `country_code` , `year` and `gdp_per_capita`
#### Preview
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/per_capita.png" width="400">

Full file is available here: [per_capita.csv](https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/per_capita.csv)

### continent_map.csv
#### Definition
The file contains the a mapping of countries and their respective continent: 251 rows, 2 columns `country_code` and `continent_code`
#### Preview
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/continent_map.png" width="400">

Full file is available here: [continent_map.csv](https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/continent_map.csv)

## The challenge
I am using my favorite SQL client [DBeaver](https://dbeaver.io). I have created a local SQLite database and imported each .csv file into a table.

### Question 1
1.1 Data Integrity Checking & Cleanup <br>

Alphabetically list all of the country codes in the continent_map table that appear more than once. Display any values where country_code is null as country_code = "FOO" and make this row appear first in the list, even though it should alphabetically sort to the middle. 
Provide the results of this query as your answer. <br>
```
SELECT 
	COALESCE(cm.country_code, 'FOO') as country_code 
FROM 
	continent_map cm
GROUP BY 1 
HAVING COUNT(1)>1
ORDER BY cm.country_code 
```
<br>
1.2 <br>
For all countries that have multiple rows in the continent_map table, delete all multiple records leaving only the 1 record per country. The record that you keep should be the first one when sorted by the continent_code alphabetically ascending. Provide the query/ies and explanation of step(s) that you follow to delete these records.

```
-- Create a temporary table with a new column ID as a row_number on the table after order by contry_code, continent_code

CREATE TABLE t1 AS
	SELECT 
		row_number() over (order by country_code, continent_code asc) as 'ID',
		country_code,
		continent_code
	FROM 
		continent_map 
 
CREATE TABLE t2 AS Select MIN(ID) as ID from t1 group by country_code 
 
/*Delete the rows that dont have a min ID number after group by country_code*/
Delete From t1 where ID NOT IN(select ID from t2) ;

/*Reset continent_map table*/
Delete From continent_map;

/*Refill continent_map from temp_table*/
insert into continent_map
  select country_code, continent_code from t1;
 
 /*drop temporary tables*/
 DROP TABLE t1;
 DROP TABLE t2;
```
