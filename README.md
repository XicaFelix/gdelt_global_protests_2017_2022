# GDELT Global Protests Preprocessing Notebook

This Colab notebook performs end-to-end preprocessing of the GDELT Events 2.0 dataset to extract, clean, and enrich global protest events from **2017–2022**. The output is a single CSV file ready for analysis and visualization in Tableau.

---

## 📂 Necessary Files

* **notebooks**

  * `GDLET_protests_2018_2021.ipynb` — the main Colab notebook.
* ** raw data**

  * `protests_1.csv`, `protests_2.csv` — raw CSV exports from BigQuery/Drive.
* **final data**

  * `gdelt_protests_final_2017_2022.csv` — final cleaned dataset.
* **lookups/**

  * [FIPS.country.txt](https://www.gdeltproject.org/data/lookups/FIPS.country.txt) — GDELT FIPS-to-country lookup.

---

## 🔧 Prerequisites

1. **Google Colab** environment
2. Install Python packages (run at top of notebook):

   ```bash
   !pip install pandas numpy spacy pycountry
   !python -m spacy download en_core_web_sm

   import pycountry
   import os
   import pandas as pd
   import numpy as np
   import spacy
   from spacy.matcher import PhraseMatcher
   ```
3. Mount your **Google Drive** to read/write data:

   ```python
   from google.colab import drive
   drive.mount('/content/drive')
   ```

---

## 🔍 Data Source

We use the BigQuery GDELT repository for protest events (EventRootCode starting with **14**). The raw data (2017–2022) can be obtained by querying BigQuery.

Here are the columns we'll use:
| Column             | Description                                              |
| ------------------ | -------------------------------------------------------- |
| **SQLDate**           | Date of event (parsed from `SQLDATE`, format YYYY‑MM‑DD) |
| **Actor1Name**     | Name of the protesting actor                             |
| **Actor2Name**     | Name of the target actor                                 |
| **Latitude**       | `ActionGeo_Lat` – latitude of the protest location       |
| **Longitude**      | `ActionGeo_Long` – longitude of the protest location     |
| **Country\_Code**  | `ActionGeo_CountryCode` – GDELT FIPS country code        |
| **EventRootCode**  | Root event code (14× series indicates protests)          |
| **EventCode**      | Detailed CAMEO event code                                |
| **GoldsteinScale** | Goldstein conflict intensity scale value                 |
| **AvgTone**        | Average tone sentiment score                             |


### Steps to get the data

1. Parse the GDELT “SQLDATE” into a proper DATE type
2. Filter to EventRootCode starting with “14” (all protest events)
3. Restrict to 2017–2022

```SQL
SELECT
  PARSE_DATE('%Y%m%d', CAST(SQLDATE AS STRING))  AS Date,
  Actor1Name,                                           
  Actor2Name,                                           
  ActionGeo_Lat AS Latitude,
  ActionGeo_Long AS Longitude,
  ActionGeo_CountryCode  AS Country_Code,  
  EventRootCode,                                        
  EventCode,                                            
  GoldsteinScale,                                       
  AvgTone                                               
FROM
  `bigquery-public-data.gdelt.events`
WHERE
  EventRootCode LIKE '14%'                              
  AND SQLDATE BETWEEN 20170101 AND 20221231  

```


Then, it should be saved in your Drive. I saved it under:

```
/content/drive/My Drive/gdelt_protests_2018_2021/protests_1.csv
/content/drive/My Drive/gdelt_protests_2018_2021/protests_2.csv
```

---

## 📝 Preprocessing Steps

1. **Load & Combine CSVs**

   * Read both `protests_1.csv` and `protests_2.csv` into pandas DataFrames
   * Concatenate and parse `Date` columns as `datetime`

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

5. **Actor Categorization (NLP + Rules)**

   * Use **spaCy’s** `PhraseMatcher` (LOWER attr) seeded with custom patterns for:

     * Civilians, Government, Political Party, NGO/Advocacy, Corporate, Agriculture, Healthcare, Prison Reform, Media Reform, Religious
   * Load ISO country and nationality lists via **pycountry** into the Government pattern
   * Compile **unique actor names** from `Actor1Name` & `Actor2Name`
   * Classify each name by:

     1. Phrase match → explicit category
     2. NER fallback (`GPE`→Government, `ORG`→NGO, `NORP`→Civilians)
     3. Default to Unknown
   * Vectorize the `name_to_category` mapping
   * Populate `PrimaryActorType`, `SecondaryActorType`, and assign `ProtestMotivation` for nuance categories

6. **Country Name Mapping**

   * Merge `ActionGeo_CountryCode` (GDELT FIPS) against FIPS text file (tab‑delimited, headerless) to get full `Country` names
   * Fill or label any missing codes as `Unknown`

7. **Final DataFrame**

   * Reorder & rename columns for consistency
   * Save final CSV to Drive:

     ```python
      output_path = '/content/drive/My Drive/gdelt_protests_2018_2021/gdelt_protests_final_2017_2022.csv'
      df.to_csv(output_path, index=False)
      print(" Saved to", output_path)
     ```

---

## 🚀 Output

* **`gdelt_protests_clean_2018_2021.csv`** in Drive — \~4.2 million rows, ready for Tableau or further analysis.

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
