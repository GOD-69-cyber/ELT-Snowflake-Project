# ELT projekt v Snowflake: Global COVID Strategies

---

## 1 Úvod a popis zdrojových dát

### Téma projektu

Projekt **Global COVID Strategies** analyzuje vývoj pandémie COVID-19 z viacerých pohľadov: epidemiologické metriky, hospitalizácie, očkovanie, mobilitu obyvateľstva a cestovné obmedzenia. Cieľom je poskytnúť komplexný analytický pohľad na dopady pandémie a reakcie štátov.

### Prečo tento dataset

* ide o komplexný, globálny dataset zo Snowflake Marketplace,
* umožňuje kombinovať zdravotné, sociálne a politické aspekty pandémie,
* je vhodný na návrh robustného dimenzionálneho modelu.

### Biznis proces

**Globálne monitorovanie pandémie a hodnotenie efektivity opatrení a očkovania** na úrovni krajín, kontinentov a regiónov.

### Typy údajov

* epidemiologické dáta (prípady, úmrtia, hospitalizácie),
* očkovanie,
* mobilita obyvateľstva,
* cestovné obmedzenia,
* geografické a demografické členenie.

### Zdroj dát (Snowflake Marketplace)

* **Database:** `GLOBAL_COVID_STRATEGIES`
* **Schema:** `INSIGHTS`

### Použité zdrojové tabuľky

| Tabuľka                                    | Popis                                           |
| ------------------------------------------ | ----------------------------------------------- |
| `COVID19_UNIVERSAL_METRICS`                | Základné globálne COVID metriky (cases, deaths) |
| `COVID19_METRICS_BY_COUNTRY`               | Detailné metriky po krajinách                   |
| `COVID19_METRICS_BY_CONTINENT`             | Agregované metriky po kontinentoch              |
| `COVID19_HOSPITAL_METRICS_BY_COUNTRY`      | Hospitalizácie a ICU                            |
| `COVID19_VACCINATION_METRICS_BY_COUNTRY`   | Očkovanie po krajinách                          |
| `COVID19_VACCINATION_METRICS_BY_CONTINENT` | Očkovanie po kontinentoch                       |
| `COVID19_TRAVEL_RESTRICTION_BY_COUNTRY`    | Cestovné obmedzenia                             |
| `COVID19_TRAVEL_RESTRICTION_BY_AIRLINE`    | Obmedzenia v leteckej doprave                   |
| `GOOGLE_GLOBAL_MOBILITY_REPORT`            | Zmeny mobility obyvateľstva                     |
| `VACCINATION_LIST_BY_COUNTRY`              | Typy vakcín dostupné v krajinách                |

 ERD zdrojových dát:

```
![ERD](/img/ERD.png)
```

---

## 2 Návrh dimenzionálneho modelu (Star Schema)

### Faktová tabuľka

### `FACT_COVID_DAILY`
**Účel:** Ukladá denné údaje o COVID-19 v krajinách, vrátane nových prípadov, úmrtí, hospitalizácií, očkovania a agregovaných metrík.  

| Stĺpec           | Typ      | Popis |
|------------------|----------|-------|
| FACT_ID          | INT      | Surrogátový kľúč faktu |
| DATE_KEY         | VARCHAR  | Odkaz na DIM_DATE |
| COUNTRY_KEY      | INT      | Odkaz na DIM_COUNTRY |
| CONTINENT_KEY    | INT      | Odkaz na DIM_CONTINENT |
| VACCINE_KEY      | INT      | Odkaz na DIM_VACCINE |
| NEW_CASES        | INT      | Nové prípady COVID-19 |
| NEW_DEATHS       | INT      | Nové úmrtia |
| HOSPITALIZED     | INT      | Hospitalizovaní pacienti |
| ICU_PATIENTS     | INT      | Pacienti na jednotkách intenzívnej starostlivosti |
| NEW_VACCINATIONS | INT      | Nové dávky očkovania |
| CUMULATIVE_CASES | INT      | Kumulatívny počet prípadov (funkcia okna) |
| AVG_7D_CASES     | FLOAT    | 7-dňový kĺzavý priemer nových prípadov (funkcia okna) |
| PREV_DAY_CASES   | INT      | Počet prípadov z predchádzajúceho dňa (LAG) |
| DAILY_CASE_RANK  | INT      | Radenie krajín podľa nových prípadov za deň (RANK) |

---

### Dimenzie
### DIM_DATE (SCD Type 0)
**Účel:** Skladá sa z údajov o dátumoch pre časovú analýzu.  

| Stĺpec     | Typ / Formát | Popis |
|------------|--------------|-------|
| DATE_KEY   | VARCHAR (YYYYMMDD) | Surrogátový kľúč dátumu |
| FULL_DATE  | DATE          | Celý dátum |
| YEAR       | INT           | Rok |
| MONTH      | INT           | Mesiac |
| DAY        | INT           | Deň |
| WEEK       | INT           | Týždeň |
 

---

### DIM_COUNTRY (SCD Type 2)
**Účel:** Skladá sa z údajov o krajinách, podporuje históriu zmien (SCD Type 2).  

| Stĺpec        | Typ       | Popis |
|---------------|-----------|-------|
| COUNTRY_KEY   | INT       | Surrogátový kľúč krajiny |
| COUNTRY_NAME  | VARCHAR   | Názov krajiny |
| ISO_CODE      | VARCHAR   | Medzinárodný kód krajiny |
| POPULATION    | INT       | Počet obyvateľov |
| IS_CURRENT    | BOOLEAN   | Prítomnosť aktuálneho záznamu |

  

---

### DIM_CONTINENT (SCD Type 0)
**Účel:** Skladá sa z údajov o kontinentoch.  

| Stĺpec         | Typ     | Popis |
|----------------|---------|-------|
| CONTINENT_KEY  | INT     | Surrogátový kľúč kontinentu |
| CONTINENT_NAME | VARCHAR | Názov kontinentu |


---

### DIM_VACCINE (SCD Type 1)
**Účel:** Skladá sa z údajov o vakcínach dostupných v rôznych krajinách.  

| Stĺpec       | Typ     | Popis |
|--------------|---------|-------|
| VACCINE_KEY  | INT     | Surrogátový kľúč vakcíny |
| VACCINES     | VARCHAR | Názov vakcíny (Pfizer, Moderna, …) |



 Star Schema diagram:

```
![Star Schema](/img/star.png)
```

---

## 3 ELT proces v Snowflake

###  Extract  staging tabuľky

```sql
CREATE OR REPLACE TABLE STG_COVID_COUNTRY AS
SELECT * FROM GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_METRICS_BY_COUNTRY;

CREATE OR REPLACE TABLE STG_HOSPITAL AS
SELECT * FROM GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_HOSPITAL_METRICS_BY_COUNTRY;

CREATE OR REPLACE TABLE STG_VACCINATION AS
SELECT * FROM GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_VACCINATION_METRICS_BY_COUNTRY;

CREATE OR REPLACE TABLE STG_VACCINATION_LIST AS
SELECT * FROM GLOBAL_COVID_STRATEGIES.INSIGHTS.VACCINATION_LIST_BY_COUNTRY;
```

---
### LOAD

```sql
CREATE OR REPLACE TABLE DIM_DATE AS
SELECT DISTINCT
    TO_VARCHAR(DATE, 'YYYYMMDD') AS DATE_KEY,
    DATE AS FULL_DATE,
    YEAR(DATE) AS YEAR,
    MONTH(DATE) AS MONTH,
    DAY(DATE) AS DAY,
    WEEK(DATE) AS WEEK
FROM STG_COVID_COUNTRY;

CREATE OR REPLACE TABLE DIM_COUNTRY AS
SELECT DISTINCT
    ROW_NUMBER() OVER (ORDER BY COUNTRY) AS COUNTRY_KEY,
    COUNTRY AS COUNTRY_NAME,
    ISO_CODE,
    POPULATION,
    TRUE AS IS_CURRENT
FROM STG_COVID_COUNTRY;

CREATE OR REPLACE TABLE DIM_CONTINENT AS
SELECT DISTINCT
    ROW_NUMBER() OVER (ORDER BY CONTINENT) AS CONTINENT_KEY,
    CONTINENT AS CONTINENT_NAME
FROM STG_COVID_COUNTRY
WHERE CONTINENT IS NOT NULL;

CREATE OR REPLACE TABLE DIM_VACCINE AS
SELECT DISTINCT
    ROW_NUMBER() OVER (ORDER BY VACCINES) AS VACCINE_KEY,
    VACCINES
FROM STG_VACCINATION_LIST;
```

---

###  Transform – faktová tabuľka (window functions)

```sql
CREATE OR REPLACE TABLE FACT_COVID_DAILY AS
WITH base_data AS (
    SELECT
        c.COUNTRY,
        c.DATE,
        c.NEW_CASES,
        c.NEW_DEATHS,
        h.HOSP_PATIENTS AS HOSPITALIZED,
        h.ICU_PATIENTS,
        v.NEW_VACCINATIONS,
        c.CONTINENT
    FROM GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_METRICS_BY_COUNTRY c
    LEFT JOIN GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_HOSPITAL_METRICS_BY_COUNTRY h
        ON c.COUNTRY = h.COUNTRY
       AND c.DATE = h.DATE
    LEFT JOIN GLOBAL_COVID_STRATEGIES.INSIGHTS.COVID19_VACCINATION_METRICS_BY_COUNTRY v
        ON c.COUNTRY = v.COUNTRY
       AND c.DATE = v.DATE
    WHERE c.DATE BETWEEN '2021-01-01' AND '2021-12-31'
)
SELECT
    ROW_NUMBER() OVER (ORDER BY COUNTRY, DATE) AS FACT_ID,
    COUNTRY,
    DATE,
    CONTINENT,
    NEW_CASES,
    NEW_DEATHS,
    HOSPITALIZED,
    ICU_PATIENTS,
    NEW_VACCINATIONS,
    SUM(NEW_CASES) OVER (PARTITION BY COUNTRY ORDER BY DATE) AS CUMULATIVE_CASES,
    AVG(NEW_CASES) OVER (
        PARTITION BY COUNTRY 
        ORDER BY DATE 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS AVG_7D_CASES,
    LAG(NEW_CASES) OVER (PARTITION BY COUNTRY ORDER BY DATE) AS PREV_DAY_CASES,
    RANK() OVER (PARTITION BY DATE ORDER BY NEW_CASES DESC) AS DAILY_CASE_RANK
FROM base_data;
```

---

## 4 Vizualizácie
*1.Dynamika nových prípadov podľa krajín
```sql
SELECT
    DATE AS FULL_DATE,
    COUNTRY,
    NEW_CASES
FROM FACT_COVID_DAILY
WHERE COUNTRY IN ('India','Brazil')
ORDER BY DATE, COUNTRY;
```
![1](/img/1.png)

*2.Tempo očkovania v rôznych krajinách
```sql
SELECT
    DATE AS FULL_DATE,
    COUNTRY,
    NEW_VACCINATIONS
FROM FACT_COVID_DAILY
WHERE COUNTRY IN ('India','Brazil')
ORDER BY DATE, COUNTRY;
```
![2](/img/2.png)

*3.Krajiny s najvyšším celkovým počtom infekcií za rok
```sql
SELECT
    COUNTRY,
    MAX(CUMULATIVE_CASES) AS TOTAL_CASES
FROM FACT_COVID_DAILY
GROUP BY COUNTRY
ORDER BY TOTAL_CASES DESC
LIMIT 10;
```
![3](/img/3.png)

*4.Top 10 krajín s najvyšším počtom nových prípadov v konkrétny deň
```sql
SELECT
    DATE AS FULL_DATE,
    COUNTRY,
    DAILY_CASE_RANK
FROM FACT_COVID_DAILY
WHERE DATE = '2021-07-01'
ORDER BY DAILY_CASE_RANK
LIMIT 10;
```
![4](/img/4.png)

*5.Dynamiku úmrtnosti na nové infekcie
```sql
SELECT
    DATE AS FULL_DATE,
    COUNTRY,
    NEW_DEATHS,
    NEW_CASES,
    (NEW_DEATHS::FLOAT / NULLIF(NEW_CASES,0)) AS DEATH_RATE
FROM FACT_COVID_DAILY
WHERE COUNTRY IN ('India','Brazil')
ORDER BY DATE, COUNTRY;
```
![5](/img/5.png)
---

## 5 Štruktúra GitHub repozitára

```
/sql
   dashboard_visualizations.sql
   etl.sql
/img
    1.png
    2.png
    3.png
    4.png
    5.png
    erd.png
    star.png
README.md
```

---

## Autor

**Volodymyr Shevchenko-Shevchuk**
