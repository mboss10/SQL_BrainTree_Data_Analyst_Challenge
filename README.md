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
I am using my favorite SQL client [DBeaver](https://dbeaver.io). I have created a local SQLite database and imported each .csv file into a table. <br><br>
Below I am listing each question as they were given in the interview challenge, and underneath each I am providing a code block of the SQL queries I have written to solve the question, along with screenshot(s) of the results when relevant.

### Question 1
1.1 Data Integrity Checking & Cleanup <br>

Alphabetically list all of the country codes in the continent_map table that appear more than once. Display any values where country_code is null as country_code = "FOO" and make this row appear first in the list, even though it should alphabetically sort to the middle. 
Provide the results of this query as your answer. <br>

#### SQL code
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


<br>
1.2 <br>
For all countries that have multiple rows in the continent_map table, delete all multiple records leaving only the 1 record per country. The record that you keep should be the first one when sorted by the continent_code alphabetically ascending. Provide the query/ies and explanation of step(s) that you follow to delete these records. <br>

#### SQL code
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

### Question 2
List the countries ranked 10-12 in each continent by the percent of year-over-year growth descending from 2011 to 2012. <br><br>

The percent of growth should be calculated as: ((2012 gdp - 2011 gdp) / 2011 gdp) <br><br>

The list should include the columns: <br><br>
* rank
* continent_name
* country_code
* country_name
* growth_percent

#### SQL code

```
/* I first create a Common Table Expression (CTE) to isolate the GDP growth percent over the year 2011 and 2012 */
with cte_2011_2012_growth AS (
select *, LAG(pc.gdp_per_capita) OVER (partition by pc.country_code ORDER BY pc.year) AS gpp_previous_year,
round((pc.gdp_per_capita - LAG(pc.gdp_per_capita) OVER (partition by pc.country_code ORDER BY pc.year))/(LAG(pc.gdp_per_capita) OVER (partition by pc.country_code ORDER BY pc.year))*100,2) as growth_percent
from per_capita pc
where pc.year in (2011,2012)
)
/* then I create a main CTE to rank the countries in each continent per descending growth percent */
, cte_main AS (
select 
	RANK() OVER (PARTITION BY cm.continent_code ORDER BY cte.growth_percent DESC) as rank,
	co.continent_name,
	cte.country_code,
	ct.country_name,
	cte.growth_percent
FROM 
	cte_2011_2012_growth cte
INNER JOIN 
	continent_map cm on cte.country_code = cm.country_code
INNER JOIN
	continents co on cm.continent_code = co.continent_code
INNER JOIN 
	countries ct on ct.country_code = cm.country_code
	)
/* Finally I select all columns from my main CTE filtering on the ones ranked 10, 11 and 12 */
SELECT * FROM cte_main ctm
where rank in (10,11,12)
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q2-results.png" width="400">

### Question 3
For the year 2012, create a 3 column, 1 row report showing the percent share of gdp_per_capita for the following regions:
* Asia,
* Europe,
* the Rest of the World.

Your result should look something like: <br>

| Asia | Europe | Rest of World |
| ------------- | ------------- | ------------- |
| 25.0%  | 25.0%  | 50.0% |

#### SQL code
```
/* I start by creating a CTE to redefine the continent names using a CASE statement */
WITH cte_redefine_map AS (
SELECT
	c.continent_name,
	CASE c.continent_name 
		WHEN 'Asia' THEN 'Asia'
		WHEN 'Europe' THEN 'Europe'
		ELSE 'Rest of World'
	END as new_continent_name
FROM
	continents c 
),
/* then I am creating my main CTE that reuses the new continent CTE and windows functions to calculate 
  the total by new continent, the total overall and the percentage per nnew contitnent definition */
cte_main AS (SELECT 
	DISTINCT crm.new_continent_name,
	SUM(pc.gdp_per_capita) OVER (PARTITION BY crm.new_continent_name) AS SUM_BY_CONTINENT,
	SUM(pc.gdp_per_capita) OVER () as total,
	round((SUM(pc.gdp_per_capita) OVER (PARTITION BY crm.new_continent_name))/(SUM(pc.gdp_per_capita) OVER ())*100,2) as cont_perc
FROM 
	continent_map cm 
INNER JOIN 
	continents c on c.continent_code = cm.continent_code 
INNER JOIN
	per_capita pc on pc.country_code = cm.country_code 
INNER JOIN 
	cte_redefine_map crm on crm.continent_name = c.continent_name
WHERE pc.year = 2012	
)
/* Lastly I simply select the percentage from the main CTE using a CASE statement to create one column per new continent definition */
SELECT 
	max(CASE cte.new_continent_name WHEN 'Asia' THEN cte.cont_perc END) AS Asia,
	max(CASE cte.new_continent_name WHEN 'Europe' THEN cte.cont_perc END) AS Europe,
	max(CASE cte.new_continent_name	WHEN 'Rest of World' THEN cte.cont_perc END) AS 'Rest of World'
FROM 
	cte_main cte
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q3-results.png" width="400">

### Question 4
4a. What is the count of countries and sum of their related gdp_per_capita values for the year 2007 where the string 'an' (case insensitive) appears anywhere in the country name?
<br><br>
4b. Repeat question 4a, but this time make the query case sensitive.

#### SQL code
```
-- 4a	
ELECT 
	COUNT(c.country_name) as "count of countries",
	ROUND(SUM(pc.gdp_per_capita),2) as "sum of gdp per capita"
FROM
	countries c 
INNER JOIN 
	per_capita pc on c.country_code = pc.country_code 
WHERE 
	UPPER(c.country_name) like '%AN%' AND PC."year" = 2007

	
-- 4b
SELECT 
	COUNT(c.country_name) as "count of countries",
	ROUND(SUM(pc.gdp_per_capita),2) as "sum of gdp per capita"
FROM
	countries c 
INNER JOIN 
	per_capita pc on c.country_code = pc.country_code 
WHERE 
	c.country_name LIKE '%an%' AND PC."year" = 2007
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q4-results.png" width="400">

### Question 5
Find the sum of gpd_per_capita by year and the count of countries for each year that have non-null gdp_per_capita where (i) the year is before 2012 and (ii) the country has a null gdp_per_capita in 2012. <br>
Your result should have the columns:
* year
* country_count
* total 

#### SQL code
```
 /* I am building a CTE that list the details for countries that have a NULL gdp in 2012 */
WITH cte_null_2012_gpd AS(
SELECT 
	pc.country_code,
	year,
	gdp_per_capita 
FROM
	per_capita pc 
WHERE 
	pc.year = 2012 AND pc.gdp_per_capita = ''	
)
/* using windows functions I am calculating the count of countries and their total gdp over each year */
SELECT 
	distinct pc."year",
	count(pc.country_code) OVER (PARTITION BY pc.year) as country_count,
	SUM(pc.gdp_per_capita)  OVER (PARTITION BY pc.year) as total
FROM
	per_capita pc 
INNER JOIN
	cte_null_2012_gpd cte on pc.country_code = cte.country_code
/* the following where clause is to make sure we only consider years prior to 2012 AND we use rows with non null gdp*/
WHERE
	pc.year < 2012 and pc.gdp_per_capita <> ''


/* I am conscious we can use classic aggregate functions with a group by on year instead of window functions, see below*/
WITH cte_null_2012_gpd AS(
SELECT 
	pc.country_code,
	year,
	gdp_per_capita 
FROM
	per_capita pc 
WHERE 
	pc.year = 2012 AND pc.gdp_per_capita = ''	
)
SELECT 
	distinct pc."year",
	count(pc.country_code) as country_count,
	SUM(pc.gdp_per_capita)  as total
FROM
	per_capita pc 
INNER JOIN
	cte_null_2012_gpd cte on pc.country_code = cte.country_code
WHERE
	pc.year < 2012 and pc.gdp_per_capita <> ''	
GROUP BY 
	pc.year
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q5-results.png" width="400">

### Question 6
All in a single query, execute all of the steps below and provide the results as your final answer:
1. create a single list of all per_capita records for year 2009 that includes columns:
   - continent_name
   - country_code
   - country_name
   - gdp_per_capita

2. order this list by:
   - continent_name ascending
   - characters 2 through 4 (inclusive) of the country_name descending
   
3. create a running total of gdp_per_capita by continent_name

4. return only the first record from the ordered list for which each continent's running total of gdp_per_capita meets or exceeds $70,000.00 with the following columns:
   - continent_name
   - country_code
   - country_name
   - gdp_per_capita
   - running_total

#### SQL code
```
 WITH cte_records AS (
SELECT
	c.continent_name,
	c2.country_code,
	c2.country_name,
	pc.gdp_per_capita,
	sum(pc.gdp_per_capita) OVER(PARTITION BY continent_name ORDER BY c.continent_name, SUBSTR(c2.country_name,2,3) DESC) as running_total
FROM 
	continent_map cm 
INNER JOIN
	continents c ON cm.continent_code = c.continent_code 
INNER JOIN 	
	countries c2 ON cm.country_code = c2.country_code 
INNER JOIN 
	per_capita pc on cm.country_code = pc.country_code 
WHERE
	pc."year" = 2009
ORDER BY 
	c.continent_name,
	SUBSTR(c2.country_name,2,3) DESC
)
SELECT 
	continent_name,
	country_code,
	country_name,
	gdp_per_capita,
	ROUND(MIN(running_total),2) as running_total
FROM 
	cte_records
WHERE
	running_total > 70000
group by 
	continent_name
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q6-results NEW.png" width="600">

### Question 7
Find the country with the highest average gdp_per_capita for each continent for all years. Now compare your list to the following data set. Please describe any and all mistakes that you can find with the data set below. Include any code that you use to help detect these mistakes.
<br><br>
| rank | continent_name	| country_code | country_name |	avg_gdp_per_capita |
| --- | --- | --- | --- | --- |
| 1 | Africa | SYC | Seychelles | $11,348.66 |
| 1 | Asia | KWT | Kuwait | $43,192.49 |
| 1 | Europe | MCO | Monaco | $152,936.10 |
| 1 | North America | BMU | Bermuda | $83,788.48 |
| 1 | Oceania | AUS | Australia | $47,070.39 |
| 1 | South America | CHL | Chile | $10,781.71 |

#### SQL code
```
WITH cte_main as (
SELECT 
	RANK () over (PARTITION BY c.continent_name ORDER BY AVG(pc.gdp_per_capita) DESC) AS rank_country,
	c.continent_name,
	pc.country_code,
	c2.country_name,
	ROUND(AVG(pc.gdp_per_capita),2) AS avg_gdp_per_capita
FROM
	per_capita pc 
INNER JOIN 
	continent_map cm on pc.country_code = cm.country_code 
INNER JOIN 
	continents c on cm.continent_code = c.continent_code 
INNER JOIN
	countries c2 on pc.country_code = c2.country_code 
GROUP BY 2,3
)
SELECT 
	*
FROM 
	cte_main
WHERE
	rank_country = 1
```

#### Results
<img src="https://github.com/mboss10/SQL_BrainTree_Data_Analyst_Challenge/blob/main/Q7-results.png" width="600">
