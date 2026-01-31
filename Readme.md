# Covid Data Project

## Overview

This project analyzes COVID-19 case, death, and vaccination data stored in the `PortfolioProject` database. The SQL queries in this repository extract, calculate, and aggregate information to answer questions such as: Which countries have the highest infection rates? What is the death rate among confirmed cases? How many people have been vaccinated (rolling totals) relative to population? The results are suitable for reporting, visualization, and further analysis.

---

## Data Sources / Tables

* `PortfolioProject..CovidDeaths`

  * Contains per-location, per-date COVID case and death metrics and population metadata.
  * Columns referenced in queries: `location`, `date`, `total_cases`, `new_cases`, `total_deaths`, `new_deaths`, `population`, `continent`.

* `PortfolioProject..CovidVaccinations`

  * Contains vaccination events per location and date.
  * Columns referenced in queries: `location`, `date`, `new_vaccinations`.

> Note: Adjust column names and types if your source tables differ. Several queries cast `new_deaths` and `new_vaccinations` to integer/numeric to ensure aggregation correctness.

---

## Files / SQL Queries Included

The project contains a set of SQL queries (see SQL file or SQL snippets in the repository). Below is a short summary of the intent behind the main queries:

1. `SELECT * FROM PortfolioProject..CovidDeaths ORDER BY 3,4`

   * Quick dump of the `CovidDeaths` table ordered by the 3rd and 4th columns (review indices before heavy use).

2. Select specific fields:
   `SELECT location, date, total_cases, new_cases, total_deaths, population FROM PortfolioProject..CovidDeaths ORDER BY 1,2`

   * Base dataset for many of the subsequent calculations.

3. Death percentage among confirmed cases (example for Egypt):
   `SELECT location, date, total_cases, total_deaths, (total_deaths/total_cases)*100 AS DeathPercentage FROM PortfolioProject..CovidDeaths WHERE location LIKE 'egypt' ORDER BY 1,2`

   * Be careful with division by zero when `total_cases = 0`.

4. Infected percentage of the population:
   `SELECT location, date, population, total_cases, (total_cases/population)*100 AS InfectedPercentage FROM PortfolioProject..CovidDeaths ORDER BY 1,2`

   * This shows the share of the population that was confirmed infected (cumulative).

5. Countries with highest infection rate (aggregate example):

   ```sql
   SELECT location, population, MAX(total_cases) AS HighestInfectionCount,
          MAX((total_cases/population))*100 AS InfectedPercentage
   FROM PortfolioProject..CovidDeaths
   GROUP BY location, population
   ORDER BY 4 DESC;
   ```

6. Highest death counts per location/continent (examples using `MAX` and `CAST`):

   * Location-level: `SELECT location, MAX(CAST(total_deaths AS int)) AS TotalDeathCount FROM PortfolioProject..CovidDeaths GROUP BY location ORDER BY TotalDeathCount DESC`.
   * Continent-level: `SELECT continent, MAX(CAST(total_deaths AS int)) AS TotalDeathCount FROM PortfolioProject..CovidDeaths WHERE continent IS NOT NULL GROUP BY continent ORDER BY TotalDeathCount DESC`.

7. Global aggregated numbers (sums and global death percentage):

   ```sql
   SELECT SUM(new_cases) AS total_cases,
          SUM(CAST(new_deaths AS int)) AS total_deaths,
          SUM(CAST(new_deaths AS int)) / SUM(new_cases) * 100 AS DeathPercentage
   FROM PortfolioProject..CovidDeaths;
   ```

   * Ensure `new_cases` sum is not zero before dividing.

8. Vaccination rolling totals and percent of population vaccinated (window function / partition):

   * Join deaths and vaccinations on `location` and `date` and compute a running (cumulative) vaccinated count per location with `SUM(...) OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date)`.
   * Example uses a CTE `PopvsVac` to compute `(RollingPeopleVaccinated / Population) * 100`.

9. Temporary table approach and persistent view:

   * Temporary table `#PercentPopVaccinated` is created to store the rolling vaccinated counts and computed percentages for later convenience.
   * A view `PercentPopulationVaccinated` is created to persist the rolling cumulative vaccination calculations for reporting/visualization.

---

## How to run

1. Ensure you have read access to `PortfolioProject..CovidDeaths` and `PortfolioProject..CovidVaccinations` and that the SQL user has privileges to create temporary tables and views if needed.

2. Run queries in a safe order:

   * Use the `SELECT` queries that do not create objects first to validate column names and data types.
   * Then run the temp table creation and insertion.
   * Finally, create the `PercentPopulationVaccinated` view if you want a reusable object for dashboards.

3. Suggested environment: SQL Server Management Studio, Azure Data Studio, or any T-SQL-capable client connected to the database.

---

## Implementation notes & best practices

* **Nulls & zeroes**: Always guard against division by zero and `NULL` values in `population`, `total_cases`, or aggregated sums. Use `NULLIF(population,0)` or `CASE WHEN population > 0 THEN ... END`.
* **Type casting**: Some numeric fields may be strings in the source (e.g., `new_vaccinations`, `new_deaths`). Cast or `CONVERT` them before summing.
* **Performance**: Window functions over large time-series tables can be expensive. Ensure proper indexes on `(location, date)` and consider pre-aggregating if the dataset is huge.
* **Data freshness**: If your data pipeline appends daily data, schedule view refreshes or rely on the underlying table updates. If using materialized snapshots, document how often they refresh.

---

## Example enhancements / visualizations

* Use the `PercentPopulationVaccinated` view as a source for:

  * Time-series charts of percent vaccinated by country.
  * Choropleth maps showing infected percentage or death percentage by country.
  * Leaderboards for top 10 countries by infection percentage or death-per-case ratio.

* Export the aggregated results to CSV for use in Power BI, Tableau, or Python/R notebooks.

---

## Troubleshooting

* If queries fail due to `conversion` or `overflow` errors, inspect the offending rows with `TRY_CAST` or `ISNUMERIC()` and clean the data.
* If the join between `CovidDeaths` and `CovidVaccinations` returns fewer rows than expected, verify date formats and that location naming is consistent (consider trimming/normalizing names).

---

## License & Attribution

If you share derived visualizations or reports, attribute the original data source (the owner of the `PortfolioProject` dataset). Choose an open license for your analysis artifacts if you plan to publish.

---

## Contact

For questions about the SQL queries or to request additional computed metrics (e.g., case fatality rate by age group, weekly rolling averages), open an issue or contact the data owner/maintainer.

---

*Generated README for the Covid Data Project.*
