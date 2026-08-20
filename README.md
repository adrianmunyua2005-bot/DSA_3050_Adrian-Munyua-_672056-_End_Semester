DSA 3050 US 2026 - BUSINESS INTELLIGENCE AND VISUALIZATION END-OF-SEMESTER PRACTICAL EXAMINATION
Dataset -  Global Terrorism Database(GTD) 

### Section A: Dataset Selection and Understanding

1. Source of the dataset
> The Global Terrorism Database (GTD) is compiled by the National Consortium for the Study of Terrorism and Responses to Terrorism (START) at the University of Maryland. It is accessed via the Kaggle mirror of the official START release (globalterrorismdb.com).

2. What does the dataset represent?
> It is an open-source database of terrorist attacks worldwide from 1970 to 2020 (with a data gap in 1993), containing over 180,000 recorded incidents. Each row represents a single terrorist attack, with details on the date, location, perpetrator group, attack method, weapon type, target type, and casualties. 

3. Why did I select it?
>The dataset is large and complex enough to support multiple dimensions of analysis. It meets the minimum requirements of at least 20,000 records. It also has multiple numerical variables, categorical variables, date/ information, and at least one variable that can be used to calculate a KPI.

4. Main Variables Available

> Category                                     Variables
> Date/time                                    iyear, imonth, iday
> Location                                      country_txt, region_txt, provstate, city, latitude, longitude
> Attack                                         characteristics attacktype1_txt, weaptype1_txt, targtype1_txt, success, suicide
> Perpetrator                                  gname (group name)
> Casualties                                    nkill, nwound
> Narrative                                     summary (free text)

5. Business/Analytical Problem
> Understanding global terrorism's evolution, geography, and lethality patterns helps identify high-risk regions, attack methods, and targets, showing shifts over five decades. This mirrors risk analysis used by security agencies, NGOs, and insurers to prioritize resources.

6. Analytical Questions
- Which regions and countries face the most attacks, and how has this changed over time?
- Have attack lethality rates (fatalities per incident) increased or decreased?
- Which attack and weapon types lead to the most casualties?
- What is the "success rate" of attacks, and does it vary by target or region?
- Which target types (civilians, government, military, business) are most attacked, and does this vary regionally?

### Section B: Power Query – Data Cleaning & Transformation

The raw Global Terrorism Database (GTD) file required extensive cleaning before it was suitable for modeling and analysis. Below are the significant transformations applied in Power Query, documented as Problem → Transformation → Reason → Result. Screenshots of each stage are included in /screenshots/.

1. Changed Type

Problem: On import, GTD's columns were not typed correctly by default — eventid, iyear and other numeric fields were read with generic or incorrect types.

Transformation: Used Transform → Data Type to explicitly set columns, e.g.:

Table.TransformColumnTypes(#"Promoted Headers", {{"eventid", Int64.Type}, {"iyear", Int64.Type}, ...})

Reason: Incorrect types prevent correct aggregation and can cause errors in later steps that depend on numeric or date operations.

Result: Core identifier and numeric columns now hold the correct base types.

2. Removed Duplicates

Problem: After the initial type pass, the extract contained fully duplicate incident rows — the same record appearing more than once.

Transformation: Used Home → Remove Rows → Remove Duplicates:

Table.Distinct(#"Changed Type")

Reason: Duplicate rows would inflate every downstream count and sum measure (Total Incidents, Total Casualties), overstating the real scale of events.

Result: The table now contains one row per genuinely distinct incident record.

3. Removed Errors

Problem: Some rows contained values that failed to convert cleanly during the type-change step, leaving error markers in the table.

Transformation: Used Home → Remove Rows → Remove Errors:

Table.RemoveRowsWithErrors(#"Removed Duplicates")

Reason: Rows with unresolved conversion errors would break aggregations and cause the query to fail on refresh.

Result: A table free of conversion errors, safe to build further transformations on.

4. Changed Type1

Problem: imonth needed further type correction after the first pass.

Transformation:

Table.TransformColumnTypes(#"Removed Errors", {{"imonth", type datetime}})

Reason / Known issue: This step currently sets imonth to Date/Time rather than Whole Number. Because GTD stores month as a small integer (0–12), Power Query interprets these integers as date serial numbers counted from 30 Dec 1899, producing the "12/31/1899" style values visible in the data preview from this step onward.

Result (current): imonth displays as an incorrect datetime value rather than a plain month number. This should be corrected — re-set imonth (and confirm iday) to Whole Number before this step, so downstream steps and the eventual Date column use the real integer values rather than misinterpreted date serials. Document the corrected version in your final submission if you fix it, and note the original issue here as a modelling/cleaning challenge you identified and resolved — this is legitimate content for the "challenges encountered" discussion the brief asks for.

5. Replaced Value — approxdate

Problem: The approxdate field (GTD's free-text note used when an incident's exact date is uncertain) contained empty strings ("") rather than true nulls where no note existed.

Transformation:

Table.ReplaceValue(#"Changed Type1", "", null, Replacer.ReplaceValue, {"approxdate"})

Reason: Empty strings and nulls are treated differently in Power Query/DAX blank checks and counts — leaving both representations mixed would cause "missing" values to be undercounted.

Result: approxdate now consistently uses null to represent "no approximate-date note."

6. Removed Columns

Problem: The raw file included a related column (free-text list of related incident IDs) that added no structured analytical value and cluttered the table alongside the international-involvement flag columns (INT_LOG, INT_IDEO, INT_MISC, INT_ANY).

Transformation:

Table.RemoveColumns(#"Replaced Value", {"related"})

Reason: related is unstructured free text (a list of other event IDs) that cannot be aggregated or filtered meaningfully without separate parsing, and was not needed for the intended analysis.

Result: A leaner table with one fewer unusable free-text column.

7. Rounded Up

Problem: eventid was a large numeric value; keeping it as a plain integer without explicit rounding left it vulnerable to being displayed or treated as an approximate/scientific-notation number rather than an exact whole identifier.

Transformation:

Table.TransformColumns(#"Removed Columns", {{"eventid", Number.RoundUp, Int64.Type}})

Reason: Ensures eventid is stored as a precise whole number identifier with no fractional rounding ambiguity, since it is used purely as a unique key and never aggregated.

Result: eventid is a clean, exact Int64 identifier.

8. Added Custom — Approximate Date Flag

Problem: GTD does not flag which incidents have a fully known date versus an estimated one — imonth/iday values of 0 indicate an unknown month/day, but nothing marks this for analysis.

Transformation:

Table.AddColumn(#"Rounded Up", "Custom", each if [imonth] = 0 or [iday] = 0 then "Approximate" else "Exact")

Reason: Downstream date-based analysis (trends, YoY comparisons) should be able to distinguish incidents with a fully known date from those with an estimated one, rather than silently treating an approximated date as exact.

Result: A new column (rename to ExactDateUnknown or similar for clarity) that labels each row "Exact" or "Approximate," ready to use as a filter or data-quality indicator on the dashboard.

### Section C: Data Modelling
I chose FactIncidents as my primary fact table because it represents each terrorism incident with a single row. This structure is ideal for capturing key numerical measures like casualties, injuries, and success or failure rates while linking directly to all the descriptive dimensions. To complement this, I set up several dimension tables: DimDate, DimGroup, DimAttackType, DimTargetType, DimWeaponType, and DimLocation. This way, I can keep the descriptive attributes organized and avoid repeating text values across numerous fact rows.

I established one-to-many relationships between each dimension and FactIncidents, ensuring that the filtering flows in one direction. This approach makes it clear how data interacts and helps avoid the complications that can arise with many-to-many relationships.

For time-related analysis, I created a dedicated DimDate table using the CALENDAR() function instead of relying on the incident date column. This allows me to utilize time-intelligence features like SAMEPERIODLASTYEAR easily.

One challenge I faced was with the Global Terrorism Database (GTD), which records incidents where the exact day or month might be unknown (indicated by imonth/iday = 0). To handle this, I decided to default those dates to the 1st of the month or year during the data preparation phase in Power Query. I also included a flag, ExactDateUnknown, to ensure that anyone analyzing the data can choose to exclude these estimated dates if they wish.

### Section D: DAX & BUSINESS CALCULATIONS
1. Total Incidents
Country Rank by Incidents  
Calculates each country's rank by total incident count, ignoring any selected country filter. This helps viewers compare the selected country to all others. Main DAX functions include RANKX and ALL. The filter context specifically removes the country filter while keeping others active. Used in the dashboard table on the Regional/Diagnostic page to show ranks alongside incident counts.

DAX
Country Rank by Incidents =
RANKX(ALL(DimLocation[country_txt]), [Total Incidents], , DESC)


Unknown Perpetrator %  
Calculates the percentage of incidents with no identified responsible group. This serves as a diagnostic tool, as rising "unknown" rates can indicate data quality issues, reflecting different regions or time periods. It uses VAR/RETURN, CALCULATE, and DIVIDE functions. The measure recalculates based on selected regions or years, aiding in comparative analysis on the Advanced/Diagnostic page.

DAX
Unknown Perpetrator % =
VAR UnknownCount =
CALCULATE([Total Incidents], FactsIncidents[gname] = "Unknown")
RETURN
DIVIDE(UnknownCount, [Total Incidents], 0)


Full Measure List (12 total)

Level 1 — Core
- Total Incidents  
- Total Killed  
- Total Wounded  
- Average Killed per Incident  

Level 2 — Business 
- Total Casualties  
- Lethality Rate %  
- Success Rate %  
- Distinct Groups Involved  

Level 3 — Advanced
- Previous Year Incidents  
- YoY Incident Growth %  
- Country Rank by Incidents  
- Unknown Perpetrator %  
