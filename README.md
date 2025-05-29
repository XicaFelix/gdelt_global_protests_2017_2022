# GDELT Global Protests Preprocessing Notebook

This Colab notebook performs end-to-end preprocessing of the GDELT Events 2.0 dataset to extract, clean, and enrich global protest events from **2017–2022**. The output is a single CSV file ready for analysis and visualization in Tableau.

---

## 📂 Necessary Files

* **notebooks**

  * `GDLET_protests_2017_2021.ipynb` — the main Colab notebook.
* ** raw data**

  * `gdelt_2017_2021_raw` — raw export from BigQuery.
* **final data**

  * `gdelt_protests_final_2017_2021.csv` — final cleaned dataset.

---

## 🔧 Prerequisites

1. **Google Colab** environment
2. Install Python packages (run at top of notebook):

   ```bash
    !pip install pycountry rapidfuzz
   
   import pycountry
   import pandas as pd
   import numpy as np
   import spacy
   from pandas_gbq import read_gbq
   from spacy.matcher import PhraseMatcher
   from rapidfuzz import process, fuzz
   ```
3. Read the raw file from **BigQuery** as a pandas data frame:

   ```python
   df = read_gbq(
    "SELECT * FROM `gdelt-protests-2019-2022.gdelt_analysis.gdelt_2017_2021_raw`",
    project_id="gdelt-protests-2019-2022",
    dialect="standard"
   )
   ```

---

## 🔍 Data Source

We use the BigQuery GDELT repository for protest events (EventRootCode starting with **14**). The raw data (2017–2022) can be obtained by querying BigQuery.

Here are the columns we'll use:
| Column             | Description                                              |
| ------------------ | -------------------------------------------------------- |
| **Date**           | Date of event (parsed from `SQLDATE`, format YYYY‑MM‑DD) |
| **Actor1Name**     | Name of the protesting actor                             |
| **Actor1Type1Code**  | Role of protesting actor (LAB - Labor, COP - police)   |
| **Actor2Name**     | Name of the target actor                                 |
| **Actor1Type1Code**  | Role of target actor (LAB - Labor, COP - police)       |
| **ActionGeo_Lat**  | Latitude of the protest location                         |
| **ActionGeo_Long** | Longitude of the protest location                        |
| **ActionGeo_CountryCode**| GDELT FIPS country code                            |
| **EventRootCode**  | Root event code (14× series indicates protests)          |
| **EventCode**      | Detailed CAMEO event code                                |
| **GoldsteinScale** | Goldstein conflict intensity scale value                 |
| **AvgTone**        | Average tone sentiment score                             |
| **NumMentions**    | Number of times protest was mentioned in news media      |



### Steps to get the data

1. Parse the GDELT “SQLDATE” into a proper DATE type
2. Filter to EventRootCode starting with “14” (all protest events)
3. Restrict to 2017–2021

```SQL
WITH events AS (
  SELECT
    PARSE_DATE('%Y%m%d', CAST(SQLDATE AS STRING)) AS Date,
    Actor1Name,
    Actor1Type1Code,
    Actor2Name,
    Actor2Type1Code,
    NumMentions,
    EventRootCode,
    EventCode,
    GoldsteinScale,
    AvgTone,
    ActionGeo_CountryCode,
    ActionGeo_Lat,
    ActionGeo_Long
  FROM
    `gdelt-bq.gdeltv2.events`
  WHERE
    EventRootCode LIKE '14%'  -- Protest events
    AND SQLDATE BETWEEN 20170101 AND 20211231
)

SELECT
  e.Date,
  e.Actor1Name,
  e.Actor1Type1Code,
  e.NumMentions,
  e.Actor2Name,
  e.Actor2Type1Code,
  e.EventRootCode,
  e.EventCode,
  e.GoldsteinScale,
  e.AvgTone,
  e.ActionGeo_CountryCode,
  geo.countryname AS CountryName,
  e.ActionGeo_Lat,
  e.ActionGeo_Long
FROM
  events e
LEFT JOIN
  `gdelt-bq.extra.countrygeolookup` AS geo
ON
  e.ActionGeo_CountryCode = geo.fips; 
```


Then, it should be saved in BigQuery table. I saved it under:

```
 delt-protests-2019-2022.gdelt_analysis.gdelt_2017_2021_raw
```

---

## 📝 Preprocessing Steps

1. **Load BigQuery table**

   * Read `gdelt_2017_2021_raw` into a pandas DataFrame
   * Parse `Date` columns as `datetime`

2. **Replace Missing Actor Names**

``` python
df['Actor1Name'] = df['Actor1Name'].fillna('Unknown Actor 1')
```
3. **Deduplicate & Validate Coordinates**

   * Drop exact duplicates on `[Date, Latitude, Longitude, EventRootCode, Actor1Name, Actor2Name]`
   * Remove rows with invalid lat/lon (outside −90–90, −180–180)

4. **Cleanup & Numeric Rounding**

   * Drop rows missing critical columns (`AvgTone`, `GoldsteinScale`)
   * Round `AvgTone` and `GoldsteinScale` to 2 decimals
   * Drop event root code, as it is the same for all rows:
     ```python
      df = df.drop(columns=['EventRootCode'])
     ```

6. **Create a column to track if protest was during Covid-Era**
   ```python
    df['COVID_Era'] = np.where(df['Date'] < '2020-03-01', 'Pre-COVID','COVID-Era')
   ```

7. **Add Protest Motivations**
   Use `EventCode` to infer protest motivations:
   ```python
    # Track motivations of the protest using the Event Code

   # Convert EventCode to string if it's numeric
   df['EventCode'] = df['EventCode'].astype(str)
   
   # Define conditions and corresponding motivations
   conditions = [
       df['EventCode'] == '141',
       df['EventCode'] == '142',
       df['EventCode'] == '143',
       df['EventCode'] == '144',
       df['EventCode'] == '145'
   ]
   
   motivations = [
       'Policy Change',
       'Anti-Government',
       'Anti-Business',
       'Group Rights',
       'Anti-Discrimination'
   ]
   
   # Default fallback if no match
   df['ProtestMotivation'] = np.select(conditions, motivations,
                                       default='General Protest')

   ```

8. **Convert ActorTypeCodes** into Human Readable Names
   Use the [GDELT CAMEO Actor Type Lookup table](https://www.gdeltproject.org/data/lookups/CAMEO.type.txt) to join the actor names into the dataset using their CAMEO codes
   
9.  **Actor Categorization (NLP + Rules)**

   * Generate a list of top raw actor names by actor type (ex. Agriculture (Type): farmer, farmhand)
    * Remove locations from the top actors list 
   * Use rapidfuzz's fuzzy matching to match all actors with similar names to our top actors to the corresponding type
   * Use `pycountry` to match all countries and subdivisions to the government category
   * 
   *  Use **spaCy’s** `PhraseMatcher` (LOWER attr) seeded with custom patterns for:

     * Civilians, Government, Political Party, NGO/Advocacy, Corporate, Agriculture, Healthcare, Prison Reform, Media Reform, Religious
   * Classify each name by:
     1. Phrase match → explicit category
     2. NER fallback (`GPE`→Government, `ORG`→NGO, `NORP`→Civilians)
     3. Default to Unknown
   * Vectorize the `name_to_category` mapping
   * Populate `PrimaryActorType`, `SecondaryActorType`, and assign `ProtestMotivation` for nuance categories

10. **Final DataFrame**

   * Reorder & rename columns for consistency
   * Save final CSV to Drive (neccessary for Tableau Public) and BigQuery(optional):

     ```python
      output_path = /content/drive/My Drive/gdelt_protests_2017_2021/gdelt_protests_final_2017_2021.csv
      df.to_csv(output_path, index=False)
      print(" Saved to", output_path)
     ```

---

## 🚀 Output

* **`gdelt_protests_final_2017_2021.csv`** in Drive — \~4.2 million rows, ready for Tableau or further analysis.

---

## 🔗 Next Steps

1. **Load into Tableau**: Connect via **Google Drive** → Text File. Refresh to pull the cleaned data.
2. **Build Visualizations**: Follow the dashboard plan to create trend lines, maps, heatmaps, and actor flows.
3. **Publish** your workbook or story to Tableau Public (or use Desktop/Server for larger extracts).

---

## 📚 References

* GDELT Event Database: [https://www.gdeltproject.org/](https://www.gdeltproject.org/)
* spaCy PhraseMatcher & NER: [https://spacy.io/usage/](https://spacy.io/usage/)
* pycountry for ISO codes: [https://pypi.org/project/pycountry/](https://pypi.org/project/pycountry/)
* Tableau Public: [https://public.tableau.com/](https://public.tableau.com/)

---
