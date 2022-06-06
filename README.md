# SQL-Tableau-COVID19-analysis
This project is based on Alex-The-Analyst's SQL and Tableau projects. 
The dashboard largely follows his YouTube tutorial, but is my own creation. 
For the second dashboard, I am creating my own SQL queries (using Google Cloud BigQuery) and Tableau Public. 
This will follow shortly. 

SQL queries:
-- 1. Looking at comparison of total cases vs total deaths
SELECT location, date, total_cases, total_deaths, (total_deaths / total_cases)*100 AS death_percent
FROM covid-data-project-345810.covid.deaths
WHERE location = 'United Kingdom'
ORDER BY 1, 2;
-- Shows likliehood of dying if you contract covid in UK

-- 2. Looking at total cases vs population in the UK
SELECT location, date, population, total_cases,(total_cases / population)*100 AS percentpopulationinfected
FROM covid-data-project-345810.covid.deaths
WHERE location = 'United Kingdom'
ORDER BY 1, 2;

-- 3. Comparison of which countries have the highest death rates per population
SELECT location, population, MAX(CAST(total_deaths AS int)) AS TotalDeathCount 
FROM covid-data-project-345810.covid.deaths
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NOT NULL
GROUP BY location, population
ORDER BY TotalDeathCount DESC;

-- 4. Now looking at a breakdown of death rates grouped by continent. 
SELECT location, MAX(CAST(total_deaths AS int)) AS TotalDeathCount 
FROM covid-data-project-345810.covid.deaths
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NULL
GROUP BY location
ORDER BY TotalDeathCount DESC;
-- Issue - North America doesn't seem to be including Canada when I did 'SELECT continent', but by location and then finding where continent is null did the trick. 
-- This still has earning brackets in it though. These can easily be removed or separated when I export to visualise. 

-- 5. Total death counts grouped and ordered by continent. 
SELECT continent, location, population, MAX(CAST(total_deaths AS int)) AS TotalDeathCount 
FROM covid-data-project-345810.covid.deaths
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NOT NULL
GROUP BY continent, location, population
ORDER BY continent, TotalDeathCount DESC;

-- 6. Comparison of which countries have the highest infection rates per population
SELECT location, population, MAX(CAST(total_cases AS int)) AS HighestInfectionCount, Max((total_cases/population))*100 AS PercentPopulationInfected 
FROM covid-data-project-345810.covid.deaths
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NOT NULL
GROUP BY location, population
ORDER BY PercentPopulationInfected DESC;

-- 7. Global numbers
-- Looking at comparison of total cases vs total deaths per day
SELECT date, SUM(new_cases) AS toal_new_daily_cases, SUM(CAST(new_deaths AS integer)) AS total_new_daily_deaths, SUM(CAST(new_deaths AS integer))/SUM(new_cases)*100 AS death_percent
FROM covid-data-project-345810.covid.deaths
WHERE continent IS NOT NULL 
GROUP BY date
ORDER BY 1, 2;

-- 8. Same but getting an aggregate across the whole of the pandemic. 
SELECT SUM(new_cases) AS toal_new_daily_cases, SUM(CAST(new_deaths AS integer)) AS total_new_daily_deaths, SUM(CAST(new_deaths AS integer))/SUM(new_cases)*100 AS death_percent
FROM covid-data-project-345810.covid.deaths
WHERE continent IS NOT NULL 
ORDER BY 1, 2;

-- 9. Looking at total population vs vaccinations
SELECT DEA.continent, DEA.location, DEA.date, dea.population, VAC.new_vaccinations, 
SUM(CAST(VAC.new_vaccinations AS integer)) OVER(PARTITION BY dea.location ORDER BY dea.location, dea.date) AS new_vac_rolling_total
FROM covid-data-project-345810.covid.deaths as DEA
JOIN covid-data-project-345810.covid.vaccinations02 as VAC
    ON DEA.iso_code = VAC.iso_code
    AND DEA.date = VAC.date
WHERE DEA.continent IS NOT NULL
    AND DEA.date > '2020-12-13'
ORDER BY 1, 2, 3;

-- 10. Looking at total population vs vaccinations
SELECT DEA.continent, DEA.location, DEA.date, dea.population, VAC.new_vaccinations, 
SUM(CAST(VAC.new_vaccinations AS integer)) OVER(PARTITION BY dea.location ORDER BY dea.location, dea.date) AS new_vac_rolling_total
FROM covid-data-project-345810.covid.deaths as DEA
JOIN covid-data-project-345810.covid.vaccinations02 as VAC
    ON DEA.iso_code = VAC.iso_code
    AND DEA.date = VAC.date
WHERE DEA.continent IS NOT NULL
    AND DEA.date > '2020-12-13'
ORDER BY 1, 2, 3;

WITH popvsvac AS
(
    SELECT DEA.continent, DEA.location, DEA.date, dea.population, VAC.new_vaccinations, 
SUM(CAST(VAC.new_vaccinations AS integer)) OVER(PARTITION BY dea.location ORDER BY dea.location, dea.date) AS rolling_people_vaccinated
FROM covid-data-project-345810.covid.deaths as DEA
JOIN covid-data-project-345810.covid.vaccinations02 as VAC
    ON DEA.iso_code = VAC.iso_code
    AND DEA.date = VAC.date
WHERE DEA.continent IS NOT NULL
    AND DEA.date > '2020-12-13'
)

SELECT *, (rolling_people_vaccinated/population)*100 AS rolling_percentage_vaccinated
FROM popvsvac;

-- 11. Looking at median ages, grouped against TotalDeathCount
WITH MedianAgeTable AS
(
SELECT continent, location, population, MAX(CAST(total_deaths AS int)) AS TotalDeathCount, VAC.median_age, 
    CASE 
        WHEN VAC.median_age < 25 THEN 'Low median age'
        WHEN VAC.median_age < 35 THEN 'Middle median age'
        WHEN VAC.median_age < 45 THEN 'High median age'
        END AS Median_Age_Groupings
FROM covid-data-project-345810.covid.deaths AS DEA
JOIN covid-data-project-345810.covid.vaccinations02 AS VAC
    ON DEA.iso_code = VAC.iso_code
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NOT NULL AND median_age IS NOT NULL
GROUP BY continent, location, population, median_age
--ORDER BY median_age, TotalDeathCount DESC
)

SELECT location, TotalDeathCount, Median_Age_Groupings
FROM MedianAgeTable 
WHERE Median_Age_Groupings IS NOT NULL AND TotalDeathCount IS NOT NULL
GROUP BY Median_Age_Groupings, TotalDeathCount, location
ORDER BY Median_Age_Groupings ASC, TotalDeathCount;

-- 12. My intent is to group the % over 65s into buckets for a comparison. 
WITH OlderAgeTable AS
(
SELECT continent, location, population, MAX(CAST(total_deaths AS int)) AS TotalDeathCount, VAC.aged_65_older, 
    CASE 
        WHEN VAC.aged_65_older < 10 THEN 'Under 10% 65 or over'
        WHEN VAC.aged_65_older < 15 THEN 'Under 15% 65 or over'
        WHEN VAC.aged_65_older < 20 THEN 'Under 20% 65 or over'
        WHEN VAC.aged_65_older < 25 THEN 'Under 25% 65 or over'
        WHEN VAC.aged_65_older < 30 THEN 'Under 30% 65 or over'
        WHEN VAC.aged_65_older < 35 THEN 'Under 35% 65 or over'
        END AS Older_Age_Groupings
FROM covid-data-project-345810.covid.deaths AS DEA
JOIN covid-data-project-345810.covid.vaccinations02 AS VAC
    ON DEA.iso_code = VAC.iso_code
-- Where statement required: there are continental groupings in the data, and this removes them so we don't have 'world' or 'asia' in the results. 
WHERE continent IS NOT NULL AND aged_65_older IS NOT NULL
GROUP BY continent, location, population, aged_65_older
ORDER BY aged_65_older, TotalDeathCount DESC
)

SELECT AVG(TotalDeathCount) AS AvgTotalDeathCount, Older_Age_Groupings
FROM OlderAgeTable 
WHERE Older_Age_Groupings IS NOT NULL AND TotalDeathCount IS NOT NULL
GROUP BY Older_Age_Groupings
ORDER BY Older_Age_Groupings;

-- 13. Correlation chart: 

SELECT location, aged_65_older, aged_70_over, MAX(CAST(total_deaths AS int)) AS TotalDeathCount, total_cases,(total_cases / population)*100 AS percentpopulationinfected, 
