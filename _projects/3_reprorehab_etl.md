---
layout: page
title: An ETL pipeline to track ReproRehab attendance
description: An ETL pipeline that converts messy attendance logs into clean, analysis-ready datasets for tracking participation and engagement.
img: assets/img/reprorehab_icon.png
importance: 1
category: work
---

## Overview

ReproRehab is a 5-year NIH-funded R25 Research Education Program (R25HD105583, 2022-2027). Teaching assistants of the program host 1-hour workblocks every week, and learners are required to attend one of them to work on assignments or projects. Learners are encouraged to host their own workblocks if none of the TA-hosted workblocks work for them.

All workblock hosts are expected to keep records in a shared Google Sheet. This is an example of the sheet:

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/google_sheet.png" title="spreadsheet_snapshot" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A snapshot of the spreadsheet
</div>

## Problem

The following issues need to be addressed before the spreadsheet can be used for analysis:

1. **Inconsistent formatting**: Different hosts used different formats. The column header expects (facilitator, date, time), and the entered values are 'Jin - 10/02/2025, 9-10am PT', 'Amanda - 1/21/26 @10am', 'Duncan - 1.28.26 @ 5p ET', etc. 

2. **Ambiguous timezones**: Entries like 'PT' do not distinguish between PST and PDT.

3. **Name inconsistencies**: Typos, nicknames, and variations (e.g., 'Callen', 'Callen M Maupin', 'Callen Maupin')

4. **No structured schema**: Difficult to aggregate, analyze, or track attendance trends

5. **No existing analysis pipeline**: Data is collected but not used effectively

---

## Solution: Step-by-step

All files can be located in `https://github.com/ohspc89/reprorehab2025/tree/main/contents/TA_Project/Jin`

The workflow of the solution is as follows: **Parse text entries** -> **Automatically fiil in missing information** -> **Human review** -> **Transform data from wide to long format for further enrichment** -> **Human review** -> **Enrich data using metadata**

#### 1) Prerequisites

- Python >= 3.12.3
- RapidFuzz == 3.14.3
- Pandas == 2.3.2
- person_master.csv
- person_alias.csv

*person_master.csv* needs to have five columns:

  - `First_name` (e.g., 'Jinseok')
  - `Last_name`  (e.g., 'Oh')
  - `Timezone`   (e.g., 'America/Los_Angeles')
  - `Pod`        (e.g., 2)
  - `Role`       (e.g., 'TA')

A person's pod assignment and role are easily identified. An individual's **Timezone** needs to be assumed. I personally reviewed ReproRehab's official website, checked a person's affiliation, and searched for the location. The most accurate way to populate this column is to confirm with each individual in the program. For example, some people may work remotely and be located in a different timezone.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/person_master_screenshot.png" title="person_master" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A screenshot of person_master.csv.
</div>

*person_alias.csv* needs to have two columns:

  - `match_key`   (e.g., 'Jin')
  - `match_value` (e.g., 'Jinseok Oh')

If you're preparing this file for the first time, you can leave both columns blank. However, please make sure that this file has column headers.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/person_alias_initial.png" title="person_alias_initial" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A screenshot of person_alias.csv (initial version).
</div>

#### 2) Folder structure

```
data/
  final_output/
  interim/
  reference/
    person_master.csv
    person_alias.csv

src/
  ETL/
    etl_parse_workblocks.py
    autocorrect.py
    etl_enrich_workblocks_pt1.py
    etl_enrich_workblocks_pt2.py
  log/
```

#### 3) Run `src/ETL/etl_parse_workblocks.py`

If you're running this script for the first time, open the script using your editor (e.g., Visual Studio) and go to the end of the script. Please replace the current value of the variable `SOURCE` with the new URL of the Google Sheet.

The URL should start with the string: `'https://docs.google.com/spreadsheets/d/'` This is followed by a long alphanumeric spreadsheet ID (e.g., `'1reeteJsj4_DjMMyLbgeQQ0_HjLEkHDONV0tHII0Tmwl/'`). The trailing portion (e.g., `'edit?gid=0#gid=0'`) of the URL should be replaced with `'export?format=csv&gid=0'`. See the image below:

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/etl_parse_workblocks.png" title="parse_edit" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Where to edit in etl_parse_workblocks.py
</div>

Once you're done editing the file, run it. Please make sure that your working directory is **src/ETL**. From the second time, you do not need to edit the variable.

**Output**

After the code runs, **interim/post_parse** will be created under **/data**. Two files (*cleaned_workblocks.csv*, *workblocks_needing_review.csv*) will be created inside the folder.

*workblocks_needing_review.csv* contains rows that need to be confirmed either because one or more fields were inferred and should be reviewed. At this point, you do not need to do anything.

*cleaned_workblocks.csv* has the following columns:
 
 - `ID_original`       : Original text entry
 - `ID_clean`          : *Normalized* 'ID_original'
 - `host`              : Name of the host, parsed from 'ID_clean'
 - `date_raw`          : Raw date parsed from 'ID_clean'
 - `date_clean`        : Formatted date: %m/%d/%Y
 - `year_inferred`      : Boolean value reporting if year was inferred
 - `time_raw`          : Start time parsed from 'ID_clean'
 - `time_clean`        : 'am/pm' attached time
 - `time_inferred_ampm`: Boolean value reporting if am/pm was inferred
 - `timezone_raw`      : Timezone parsed from 'ID_clean'
 - `tz_full`           : 'timezone_raw' converted to canonical timezones
 - `datetime`          : Local date & time prepared using 'date_clean', 'time_clean', and 'tz_full'
 - `parse_status`      : Boolean value indicating whether the parsed output should be reviewed
 - `review_reason`     : Reasons for reviewing parse_status
 - `Attendee_(%d)`     : Columns labeled for later wide to long transformation

*etl_parse_workblocks.log* is also saved in **/src/log**. The log records the number of rows requiring review. Rows are categorized by specific reasons for review and counted accordingly for user inspection.

#### 4) Run `src/ETL/autocorrect.py`

Missing timezones can be corrected automatically using metadata from *person_master.csv*.

**Output**

After the code runs, *Timezone_corrected.csv* will be saved inside **interim/post_parse**.

*autocorrect.log* is also saved in **/src/log**. The log records the similarity between `host` and `First_name` in *person_master.csv, calculated using the RapidFuzz module. If the calculated score is greater than 75, the missing timezone of the host will be filled with the timezone of the matched candidate.

#### 5) Review and update the corrected fields

Open *Timezone_corrected.csv* and review whether the timezones are correct for all hosts by checking `tz_full` column. Also, you may want to check `host` column to see if there's any missing host name. Both `datetime` and `date_clean` columns need to be reviewed as well. Once the review is done, please save the file as *corrected_workblocks.csv* (this reviewed file is required for the downstream enrichment steps).

#### 6) Run `src/ETL/etl_enrich_pt1.py`

This script transforms *corrected_workblocks.csv* into a long-format attendance table. Each row is associated with a unique `session_id`.

**Output**

After the code runs, *host_ensured_long.csv* will be saved inside **interim/post_enrichment_pt1**. Also, *person_alias.csv* inside **data/reference** is updated.

*host_ensured_long.csv* copies columns: `host`, `datetime`, `date_clean`, `attendee_raw` from *corrected_workblocks.csv*. Additionally, it has the following new columns:
 
 - `session_id`        : Unique workblock id
 - `attendance_source` : 'listed_attendee', 'host_added_missing', 'host_only_session'
 - `is_host`           : Boolean value reporting if the row's `attendee_raw` is the host of the session
 - `host_match_key`    : Key to be matched with `match_value` in *person_alias.csv* using `match_key`
 - `attendee_match_key`: Key to be matched with `match_value` in *person_alias.csv* using `match_key`
 - `time_inferred_ampm`: Boolean value reporting if am/pm was inferred

*etl_enrich_workblocks_pt1.log* is also saved in **/src/log**. The log records the number of rows at each transformation stages of the script.

#### 7) Review *person_alias.csv*

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/img/person_alias_updated.png" title="person_alias_updated" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    A screenshot of person_alias.csv (updated version).
</div>

Fill in missing `match_value` entries and review the correspondence between `match_key` and `match_value`.

#### 8) Run `src/ETL/etl_enrich_pt2.py`

This script adds person-level metadata to *host_ensured_long.csv*.

**Output**

After the code runs, *workblock_attendance_enriched.csv* will be saved inside **data/final_output**.

*workblock_attendance_enriched.csv* copies columns: `session_id`, `date_clean`, `host`, `attendee_raw` from *host_ensured_long.csv*. Additionally, it has the following new columns:
 
 - `workblock_datetime_utc` : Datetime each workblock was hosted in UTC
 - `host_full_name`         : Canonical name of the host
 - `host_timezone`          : Canonical timezone where the host is in
 - `host_local_datetime`    : Local datetime for the host
 - `host_local_date`        : Local date for the host
 - `host_local_hour`        : Local hour for the host
 - `host_local_weekday`     : Weekday of the workblock for the host
 - `attendee_full_name`     : Canonical name of the attendee
 - `attendee_timezone`      : Canonical timezone where the attendee is in
 - `attendee_local_datetime`: Attendee equivalent of `host_local_datetime`
 - `attendee_local_date`    : Attendee equivalent of `host_local_date`
 - `attendee_local_hour`    : Attendee equivalent of `host_local_hour`
 - `attendee_local_weekday` : Attendee equivalent of `host_local_weekday`
 - `attendee_pod`           : Pod assignment for the attendee
 - `attendee_role`          : Role ('TA' vs. 'Learner')
 - `is_host`                : Boolean value indicating whether the attendee row corresponds to the host

*etl_enrich_workblocks_pt2.log* is also saved in **/src/log**. Sometimes hosts put down their names differently as attendees, and that part is corrected ('Host-only sessions misspecified:').

---

## Results / Impact

This pipeline produces:
- a parsed session-level dataset,
- a review queue for ambiguous entries,
- a long-format attendance table,
- and an enriched dataset with canonical names, timezones, and local-time features.

These outputs make it possible to:
- track attendance by learner, TA, and pod,
- analyze participation patterns by local date and time,
- and iteratively improve name normalization through the alias table.

---

## Tech Stack

- **Python** (re, pandas, logging, RapidFuzz)
- **Tabular reference ingestion** from a shared spreadsheet source

---
