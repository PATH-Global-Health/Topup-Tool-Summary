# Zambia Rapid Reporting CHW/Facility Top-Up Program: Full Walkthrough

## Key Takeaways

- Health workers are paid airtime **only when they submit a report**; reporting nothing means being paid nothing.
- The **sooner** a report is submitted after its reporting period closes, the **higher** the payment; the longer someone waits, the less they get, down to a small "better late than never" minimum.
- Before anything is paid, every report is checked for a valid/real phone number, a genuine active account, no duplicate or backdated submissions, and complete data.
- Anything that fails those checks isn't silently dropped; it's routed into a "for review" file that goes to program staff every cycle.

This version explains **every step of the pipeline and every function involved**, in plain language first, with the actual code and database queries included underneath for anyone technical who wants to verify. No prior knowledge of R (the programming language used) or SQL (the database query language used) is assumed; each is explained as it comes up. Code and SQL snippets are collapsed by default; click any "Click to see..." line to expand it if you want to check the details yourself. The anti-gaming/anti-theft rules are covered in their own section at the end, after the full pipeline, since most of them are individual steps *inside* that pipeline.

---

## Table of Contents

- [Key Takeaways](#key-takeaways)
- [1. What This Program Is](#1-what-this-program-is)
- [2. The Pipeline: How a Report Turns Into a Payment, Step by Step](#2-the-pipeline-how-a-report-turns-into-a-payment-step-by-step)
  - [Pipeline A: Weekly Rapid Reporting Top-Up](#pipeline-a-weekly-rapid-reporting-top-up)
  - [Pipeline B: Step D Monthly Top-Up](#pipeline-b-step-d-monthly-top-up)
- [3. Anti-Gaming / Anti-Theft Guards](#3-anti-gaming-anti-theft-guards)
  - [Duplicate-Payment Prevention](#duplicate-payment-prevention)
  - [Identity & Eligibility Checks](#identity-eligibility-checks)
  - [Data & Timing Validity Checks](#data-timing-validity-checks)
  - [Oversight](#oversight)
- [4. Appendix: Quick Function Reference](#4-appendix-quick-function-reference)
- [5. Summary](#5-summary)

---

## 1. What This Program Is

Two unattended scripts run automatically on the server and generate a report of the payment amounts owed to Zambian health workers who submitted their malaria surveillance report, **with the amount owed depending on how promptly they reported.** The sooner after their reporting period closes that they submit, the more airtime they're due; the longer they wait, the less they're due (down to a small token minimum); and if they never submit at all, they're due nothing. Amounts are calculated in ZMW (Zambian Kwacha) and also shown converted to USD for reporting. Before anything is included in the report, the data goes through several validity and fraud checks (detailed in Section 3).

| Job | Script | Schedule | Who it pays |
|---|---|---|---|
| Weekly Rapid Reporting Top-Up | `weekly_topup.R` | Every Tuesday, 00:01 | Facility mobile reporters submitting **weekly** malaria case data |
| Step D Monthly Top-Up | `stepD_report.R` | 10th of each month, 04:00 | Facility contacts + "Data CHWs" submitting **monthly** aggregate data |

**How that schedule is actually set:** these are two entries in the server's crontab, the standard Linux job scheduler that runs a command automatically at fixed times without anyone logging in to trigger it:

<details>
<summary>Click to see the actual crontab entries</summary>

```
MAILTO="root"
1 0 * * 2 /usr/bin/Rscript /var/lib/dhis2/dhis/scripts/nmectools/exec/weekly_topup.R
MAILTO="root"
0 4 10 * * /usr/bin/Rscript /var/lib/dhis2/dhis/scripts/nmectools/exec/stepD_report.R
```
</details>

A few things worth noting about these two lines:

- The schedule matches the table above: the weekly job fires every Tuesday at 00:01, and the Step D job fires at 04:00 on the 10th of each month.
- `/usr/bin/Rscript` is just how cron runs an `.R` file automatically, without anyone opening R by hand.
- The install path sits next to the `dish.json` credentials file from Step 1.
- `MAILTO="root"` only routes script *crash* errors to the server admin; it's unrelated to the payment-summary emails the scripts send to program staff.

---

## 2. The Pipeline: How a Report Turns Into a Payment, Step by Step

Both jobs follow the same overall shape: **log in, ask the database a question, clean and check the answer, work out who gets paid what, make report files, then send them out.** Below, each step is broken out for both jobs, with the responsible R function(s), a plain-English explanation, the underlying SQL question where relevant (translated into plain English *and* shown as-is), and the key snippet of code.

### Step 1: Log in to DHIS2
**Function:** `loginToDHIS2()` (in `R/loginToDHIS2.R`)

Before the script can ask DHIS2 for anything, it has to authenticate, the same way you'd log into a website with a username and password, except here the credentials are read from a small configuration file on the server rather than typed by a person. This function is a thin wrapper around a login routine from a separate, shared library (`datimutils`) that the wider DHIS2/PEPFAR-Zambia ecosystem uses.

**Where the credentials actually live:** both cron jobs call this function with the same fixed path:

```r
loginToDHIS2("/var/lib/dhis2/dhis/dish.json")
```

(`exec/weekly_topup.R` line 41, `exec/stepD_report.R` line 12). That file, `dish.json`, is not part of the `nmectools` code or its git repository at all; it's a small JSON text file that lives only on the server's disk, outside any source-controlled folder, so the DHIS2 username/password never appear in the codebase itself.

**How it's read:** `loginToDHIS2()` forwards that file path, along with a `config_path_level` setting (which defaults to `"dhis"`), into `datimutils::loginToDATIM()`. This is the standard configuration convention used across PEPFAR/DATIM's R tooling: the JSON file is expected to have a top-level section named after that level, containing the server's base URL, a username, and a password, roughly:

```json
{
  "dhis": {
    "baseurl": "https://<dhis2-server-url>/",
    "username": "<service-account-username>",
    "password": "<service-account-password>"
  }
}
```

(That's the standard shape this convention expects, not a copy of the real file.) `loginToDATIM()` parses the JSON, pulls out the `"dhis"` section, and uses those three values to authenticate to the DHIS2 server's API, the same login step a person would go through in a browser, just done in code with a stored service account instead of a human typing a password.

**What happens after login:** a successful login produces a "session" object, an internal record holding the authenticated connection (`handle`) and the server's `base_url`, which `loginToDHIS2()` stores in the calling script's environment under the name `d2_default_session`. Every other function used later in the pipeline (`getWeeklyRawData`, `getRapidReportUsers`, `getOrgUnitStructure`, `sendWeeklyReportByEmail`, and so on) takes that same object in as its `d2_session` argument and reuses it for its own DHIS2 API calls, for example `d2_session$base_url` and `d2_session$handle` in Step A1 below. So the login happens exactly once, right at the start of each run, and its result is threaded through everything that follows.

<details>
<summary>Click to see the actual R code</summary>

```r
loginToDHIS2 <- function(config_path = NULL, ...) {
  datimutils::loginToDATIM(config_path = config_path, ...)
}
```
</details>

### Step 2: Set up the report's basic info
**Function:** `createReportInfo(report_date)` (in `R/createReportInfo.R`)

This just creates an empty container to hold everything the rest of the pipeline will build up (the report date being processed, plus a few secret keys read from the server's environment: a currency-conversion API key, and the Telegram bot credentials used later to post the results). Think of it as opening a fresh folder labeled with today's date before starting to fill it with paperwork.

<details>
<summary>Click to see the actual R code</summary>

```r
createReportInfo <- function(report_date) {
  d <- list()
  d$report_date <- as.Date(report_date)
  d$info <- list(
    lucr_api_key = Sys.getenv("lucr_api_key"),
    telegram_token = Sys.getenv("telegram_token"),
    telegram_chat_id = Sys.getenv("telegram_chat_id")
  )
  d
}
```
</details>

---

### Pipeline A: Weekly Rapid Reporting Top-Up

#### Step A1: Ask the database, "who submitted a weekly report recently, and how late was it?"
**Function:** `getWeeklyRawData(report_date, d2_session)` (in `R/prepareWeeklyTopupData.R`)
**SQL behind it:** `inst/sql/topup_raw_data.sql`

In plain English, this database question means: *"Look at the official 'report completed' log for the Weekly Rapid Reporting dataset. Find every facility/week combination where a completion was recorded in the last 7 days, but only if that facility/week hadn't already been marked complete before this most-recent 7-day window. Tell me the facility name and ID, which week it was for, when that week started, how many days ago that was (the 'period age'), and exactly when it was submitted."*

The "only if it hadn't already been marked complete before" part is a deliberate safeguard baked directly into the database question (not just the R code), so that a facility's submission for a given week is only ever picked up **once**, in the week it actually happened, and can't accidentally be re-counted (and re-paid) in a later week's run.

<details>
<summary>Click to see the actual SQL query</summary>

```sql
SELECT ou.name,
ps.iso as week_value,
p.startdate,
'${reportExDate},'::date - p.startdate  as period_age,
y.record_age,
ou.uid as orgunit_uid,
y.submission_date from
( SELECT datasetid, periodid, sourceid ,date as submission_date,
'${reportExDate}'::timestamp  - date as record_age
from
(
SELECT datasetid, periodid, sourceid, MAX(date) as date from (
SELECT a.datasetid, a.periodid,a.sourceid,a.date,b.exists
from private.completedatasetregistration_new_records a
LEFT JOIN (
SELECT  DISTINCT datasetid, periodid,sourceid, 1 as exists
from private.completedatasetregistration_new_records
WHERE ('${reportExDate}'::timestamp, - date) > '7 days'::interval
and date <= '${reportExDate}'::date
) b on a.datasetid = b.datasetid
AND a.periodid = b.periodid
and a.sourceid = b.sourceid
where a.datasetid = 69047
and a.date <= '${reportExDate}'::date
and '${reportExDate}'::timestamp - date  <= '7 days'::interval
) as foo
WHERE exists IS NULL
GROUP BY datasetid, periodid, sourceid
)  as x )  as y
INNER JOIN organisationunit ou on y.sourceid = ou.organisationunitid
INNER JOIN period p on y.periodid = p.periodid
INNER JOIN _periodstructure ps on p.periodid = ps.periodid
WHERE p.startdate < '${reportExDate}'::date
ORDER BY ou.name, ps.iso DESC
```
</details>

This question is sent to DHIS2 as a web request (an "API call"), and DHIS2 sends back the answer as a spreadsheet-style file (CSV) that R then reads and processes:

<details>
<summary>Click to see the actual R code</summary>

```r
url <- paste0(d2_session$base_url,
              "api/sqlViews/PaIP00gzoyN/data.csv?var=reportExDate:", report_date)
topup_data <- url %>%
  httr::GET(httr::timeout(180), handle = d2_session$handle) %>%
  httr::content(., "text") %>%
  readr::read_csv(file = ., show_col_types = FALSE) %>%
  dplyr::rename(orgunit_uid = uid, orgunit_name = name, submitted_by = storedby) %>%
  dplyr::group_by(orgunit_uid, orgunit_name, iso, startdate, submitted_by) %>%
  dplyr::mutate(is_last_submission = submission_date == max(submission_date)) %>%
  dplyr::filter(is_last_submission) %>%
  dplyr::ungroup() %>%
  dplyr::mutate(period_age = as.Date(report_date) - startdate) %>%
  dplyr::mutate(amount = dplyr::case_when(
    period_age <  8                    ~ 0,
    period_age >= 8  & period_age < 15 ~ 5,
    period_age >= 15 & period_age < 22 ~ 3,
    period_age >= 22 & period_age < 29 ~ 2,
    period_age >= 29                   ~ 1))
```
</details>

**What this does in plain terms:** for every facility/week returned, it keeps only the very latest submission if there happened to be more than one on record, works out how many days late the report was ("period age"), and then looks that number up in a simple table to decide the airtime amount:

| Days since the period started | Payment |
|---|---|
| Fewer than 8 days (period not even closed yet) | **0 ZMW** (nothing paid) |
| 8-14 days | **5 ZMW** (the top reward, for reporting right after the period closes) |
| 15-21 days | **3 ZMW** |
| 22-28 days | **2 ZMW** |
| 29+ days | **1 ZMW** (a small "better late than never" minimum) |

#### Step A2: Get the list of eligible reporting users
**Function:** `getRapidReportUsers(d2_session)` (in `R/getRapidReportUsers.R`)

This asks DHIS2 for everyone in the "mobile reporting" user group, along with their registered phone number and which facility they belong to, and separately asks for everyone in a specific "excluded users" group (people staff have manually flagged as ineligible; see Section 3.3). It then works out, from the shape of each phone number, which mobile network it belongs to.

<details>
<summary>Click to see the actual R code</summary>

```r
excluded_users <- datimutils::getUserGroups("KiTZhKdn5VL",
    fields = "users[userCredentials[username]]", d2_session = d2_session) %>%
  purrr::pluck("users") %>% unlist(.)

users <- datimutils:::api_get(
  "users?fields=phoneNumber,organisationUnits[id,name],userCredentials[lastLogin,username,disabled]&filter=userGroups.id:in:[D0WkbASa55p]",
  d2_session = d2_session) %>%
  purrr::pluck("users") %>%
  tidyr::unnest("organisationUnits") %>%
  dplyr::mutate(phoneNumber = gsub("[[:space:]]", "", phoneNumber)) %>%
  dplyr::mutate(phoneNumber = stringr::str_replace(phoneNumber, "^[\\+]?26", "")) %>%
  dplyr::mutate(service_provider = dplyr::case_when(
    stringr::str_detect(phoneNumber, "^(076|096|073)") ~ "MTN",
    stringr::str_detect(phoneNumber, "^(097|077)") ~ "Airtel",
    stringr::str_detect(phoneNumber, "^095") ~ "Zamtel",
    TRUE ~ "Unknown")) %>%
  dplyr::mutate(is_excluded = userCredentials.username %in% excluded_users)
```
</details>

**Plain terms:** it builds a table of "who is allowed to be paid, what's their phone number, which network is it on, and are they on the do-not-pay list."

#### Step A3: Get which donor pays for which facility
**Function:** `getTopupOrgunitGroups(d2_session)` (in `R/getOrgunitGroupSetsMembers.R`)

Asks DHIS2 which funding donor is responsible for each facility (so the final report can be broken out donor-by-donor for accounting purposes; this is a bookkeeping detail, not a fraud control).

#### Step A4: Combine everything and decide who actually gets paid
**Function:** `prepareWeeklyTopupData(report_date, d2_session)` (in `R/prepareWeeklyTopupData.R`; the main "conductor" function that calls all the others in order)

It joins the submission data (Step A1) to the user list (Step A2) and the donor list (Step A3), then applies the eligibility test described fully in Section 3:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::mutate(status_good = service_provider != "Unknown" &
                             !is_excluded &
                             amount != 0) %>%
dplyr::mutate(submitted_by_proxy = facility_username != submitted_by)

d$topup_data   <- topup_data %>% dplyr::filter(status_good)   # gets paid
d$bad_records  <- topup_data %>% dplyr::filter(!status_good)  # does not get paid
```
</details>

Records that pass are grouped by phone number and network, and their amounts across every period reported that week are added up into a final voucher list:

<details>
<summary>Click to see the actual R code</summary>

```r
d$topup_summary <- d$topup_data %>%
  dplyr::group_by(phoneNumber, service_provider, airtime_donor) %>%
  dplyr::summarise(amount = sum(amount)) %>%
  dplyr::mutate(VoucherType = "Direct-Topup") %>%
  dplyr::select(ServiceProvider = service_provider, VoucherType,
                Recipient = phoneNumber, Amount = amount, airtime_donor)
```
</details>

#### Step A5: Build the summary tables (with a currency conversion)
**Function:** `createTopupSummaryTables(d)` (in `R/createTopupSummaryTable.R`)

Adds up the total ZMW being paid out per donor, converts that total into US Dollars using a live currency-conversion service (the `lucr` package with an API key), and produces that summary as a formatted table in three forms: an HTML table (for the email), a PNG image of the table (for Telegram), and a plain-text Markdown table.

<details>
<summary>Click to see the actual R code</summary>

```r
topup_summary_table <- d$topup_summary %>%
  dplyr::group_by(airtime_donor) %>%
  dplyr::summarise(Amount = sum(Amount)) %>%
  dplyr::mutate(USD = lucr::currency_convert(Amount, from = "ZMW", to = "USD",
                                              key = d$info$lucr_api_key)) %>%
  dplyr::bind_rows(data.frame(airtime_donor = "Total",
                               Amount = sum(.$Amount), USD = sum(.$USD)))
```
</details>

#### Step A6: Write out the voucher files and zip them up
**Function:** `writeAttachmentFiles(d)` (in `R/writeAttachmentFiles.R`)

For every donor, it writes an Excel spreadsheet of exactly who should be topped up how much (these are the actual "voucher" files that presumably feed the airtime-purchasing step downstream). It also writes out three "for review" spreadsheets: the rejected/bad records with the reason each was rejected, the full raw data, and a list of users whose phone numbers didn't validate. Everything is then compressed into a single zip file.

<details>
<summary>Click to see the actual R code</summary>

```r
for (i in seq_along(donors)) {
  foo <- dplyr::filter(d$topup_summary, airtime_donor == donors[i])
  openxlsx::write.xlsx(foo, attachment_name.new)   # one Excel file per donor
}
openxlsx::write.xlsx(d$bad_records, bad_records_attachment)
openxlsx::write.xlsx(d$topup_data, topup_data_attachment)
openxlsx::write.xlsx(d$users_bad_phonenumbers, ...)
zip(zipfile = zip_file_name, files = unlist(d$attachments), flags = "-r9XjD")
```
</details>

#### Step A7: Email the results
**Function:** `sendWeeklyReportByEmail(d, d2_session)` (in `R/sendWeeklyReportByEmail.R`)

Looks up who should receive the report (everyone in a specific DHIS2 user group, via `getReportRecipients()` in `R/getReportRecipients.R`, which filters to only well-formed email addresses), then emails them the summary table plus the zip file of vouchers as an attachment.

#### Step A8: Post to Telegram
**Function:** `sendWeeklyTelegramReport(d)` (in `R/sendWeeklyTelegramReport.R`)

Posts the summary table (as an image) and the same zip file to a Telegram chat, via a Telegram "bot" account, for a faster/mobile-friendly heads-up alongside the email.

---

### Pipeline B: Step D Monthly Top-Up

This pipeline pays two separate groups from the same monthly dataset: **facility contacts** (a phone number registered against the facility itself) and **"Data CHWs"** (the individual community health worker user account that actually submitted the report).

#### Step B1: Ask the database, "who completed the monthly Step D report, and was the data actually filled in properly?"
**Function:** `getStepDTopUpData(report_date, d2_session)` (in `R/getTopupData.R`)
**SQL behind it:** `inst/sql/stepd_raw_data.sql`

In plain English: *"For the 'Step D Monthly CHW' dataset, find every facility/month where a completion was recorded in the last month, along with who recorded it. Also tell me the very first time that facility/month was ever completed (for history). Also count how many of the 9 specific mandatory fields actually have a value filled in for that facility/month. Tell me how many days ago it was submitted, and how many days ago the reporting month itself ended."*

<details>
<summary>Click to see the actual SQL query</summary>

```sql
 SELECT ou.uid,
  p.iso,
  p.startdate,
  p.enddate,
  a.date::date as submission_date,
  b.first_date as first_date,
  ceil(extract(epoch from '${reportExDate}'::timestamp - a.date)/86400) as report_age,
  ceil(extract(epoch from '${reportExDate}'::timestamp - p.enddate::date)/86400) as period_age,
  a.storedby,
  COALESCE(c.count,0) as record_count FROM
  (SELECT periodid, sourceid, storedby,MIN(date) as date
    FROM private.completedatasetregistration_new_records
    WHERE datasetid = (SELECT datasetid from dataset where name = 'Step D Monthly CHW')
    and date <='${reportExDate}'::date
    AND age('${reportExDate}'::date,date)<='1 month'::interval
    GROUP BY periodid, sourceid,storedby ) a
  INNER JOIN (
    SELECT periodid,sourceid,min(date)::date as first_date
    from private.completedatasetregistration_new_records
    WHERE datasetid = (SELECT datasetid from dataset where name = 'Step D Monthly CHW')
    GROUP BY periodid,sourceid
  ) b on b.periodid = a.periodid and b.sourceid = a.sourceid
  LEFT JOIN (
      SELECT sourceid,periodid,count(value)
      from datavalue where dataelementid IN (
        SELECT dataelementid from dataelementgroupmembers
        where dataelementgroupid = (
          (SELECT dataelementgroupid from dataelementgroup where uid = 'RLdFtwhThNQ')
        ) )
      AND  age('${reportExDate}',created)<='1 month'::interval
      GROUP BY sourceid,periodid
    ) c on c.periodid = a.periodid and c.sourceid = a.sourceid
    INNER JOIN _periodstructure p on a.periodid = p.periodid
    INNER JOIN organisationunit ou on a.sourceid = ou.organisationunitid
    ORDER BY ou.uid,p.iso
```
</details>

That `record_count` (how many of the mandatory fields were actually filled in) is what later becomes the "was this report actually complete?" check in Section 3.8; it's calculated directly by the database query, then checked in R.

The R code then reads this answer, tags the dates properly, and drops anything with zero recorded values:

<details>
<summary>Click to see the actual R code</summary>

```r
raw_data <- response %>% httr::content(., "text") %>%
  readr::read_csv(file = ., col_types = readr::cols(.default = "c")) %>%
  dplyr::mutate(orgunit_uid = uid, startdate = as.Date(startdate),
                enddate = as.Date(enddate), submission_date = as.Date(submission_date),
                first_date = as.Date(first_date), report_age = as.numeric(report_age),
                period_age = as.numeric(period_age)) %>%
  dplyr::filter(record_count > 0)
```
</details>

#### Step B2: Enrich with org units, Data CHW identities, and donor/training groups
**Functions:**
- `getOrgUnitStructure(d2_session)` (in `R/getOrgUnitStructure.R`, SQL view `CvYLXS2pyQv`): the province/district/facility hierarchy each report belongs to (an "org unit" is DHIS2's term for any place in that hierarchy: a province, a district, a facility, or a specific health post).
- `getDataCHWList(d2_session)` (in `R/getDataCHWs.R`): the list of active Data CHW user accounts, their phone numbers, and their place in the org unit hierarchy (kept only if they're a genuine, currently active, front-line-level account; see Section 3.5).
- `getStep3Orgunits(d2_session)` (in `R/getDataCHWs.R`): the facility's own registered contact person and phone number.
- `getTopupOrgunitGroups(d2_session)` (in `R/getOrgunitGroupSetsMembers.R`): which donor funds each facility, and (if set) who trained that facility's staff.
- `classifyMobileOperator()` (in `R/classifyMobileOperator.R`): the same phone-number-to-network classifier used in the weekly job (see Step A2), run twice here: once on the facility's contact number, once on the Data CHW's number.

All of this is joined onto the raw data, and a few pass/fail flags are computed for every record:

<details>
<summary>Click to see the actual R code</summary>

```r
raw_data <- raw_data %>%
  dplyr::mutate(
    missing_mandatory_des = record_count != 9,
    report_in_future      = period_age < 0,
    not_monthly            = !stringr::str_detect(iso, "^20[12][0-9]{3}"),
    bad_facility_number    = contact_phone_operator == "Unknown",
    bad_chw_number         = data_chw_phone_operator == "Unknown",
    unknown_donor          = airtime_donor == "Unknown",
    qualifies = !(missing_mandatory_des | report_in_future | not_monthly),
    elapsed_months = (lubridate::interval(enddate, as.Date(report_date)) %/% months(1)))
```
</details>

#### Step B3: Work out the two separate payment lists
**Function:** `prepareStepDTopupReport(report_date, d2_session)` (in `R/prepareStepDTopupData.R`; the "conductor" for this pipeline)

**Facility payments:** only for records that `qualifies` and have a valid facility number, paid on a declining scale by how late they were (relative to when the reporting month actually ended):

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::mutate(amount = dplyr::case_when(
  period_age <= 10                     ~ 20,
  period_age > 10 & period_age <= 30   ~ 15,
  period_age > 30 & period_age <= 60   ~ 10,
  period_age > 60                      ~ 5))
```
</details>

| Days since the reporting month ended | Payment |
|---|---|
| Up to 10 days | **20 ZMW** |
| 11-30 days | **15 ZMW** |
| 31-60 days | **10 ZMW** |
| Over 60 days | **5 ZMW** (small minimum for very late catch-up) |

**Data CHW payments:** a flat 4 ZMW per qualifying report they personally submitted that month (so someone covering multiple periods/facilities in one month accumulates 4 ZMW for each):

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::group_by(airtime_donor, data_chw_phone, data_chw_phone_operator) %>%
dplyr::summarise(amount = dplyr::n() * 4, .groups = "drop")
```
</details>

Anyone who fails the number-validity check is instead routed to a rejected list (`facility_bad_numbers`, `data_chw_bad_numbers`) for staff review rather than being paid.

#### Step B4: Build the summary tables (with currency conversion)
**Function:** `createStepDSummaryTables(d)` (in `R/createStepDSummaryTables.R`)

Same idea as Step A5, but produces two separate tables/images, one for facility payments and one for Data CHW payments, each totalled by donor and converted to USD.

#### Step B5: Write out the voucher files and zip them up
**Function:** `prepareStepDAttacments(d)` (in `R/prepareStepDAttachments.R`)

Same idea as Step A6: one Excel voucher file per donor for facility payments, one per donor for Data CHW payments, plus the rejected-number lists and the full raw data, all zipped together.

#### Step B6: Post to Telegram
**Function:** `sendStepDTelegramReport(d)` (in `R/sendStepDTelegramReport.R`)

Posts a text summary (qualifying vs disqualified report counts, bad-number counts, and both payment totals in ZMW and USD) plus both summary table images and the zip file.

<details>
<summary>Click to see the actual R code</summary>

```r
message <- paste0("*StepD Topup Report Summary* \r\n",
  "Qualifying reports: ", sum(d$raw_data$qualifies), "\r\n",
  "Disqualified reports: ", sum(!d$raw_data$qualifies),
  " ( ", round(sum(!d$raw_data$qualifies) / NROW(d$raw_data) * 100), "% ) \r\n",
  "Unknown Facility numbers:", NROW(d$facility_bad_numbers), "\r\n",
  "Unknown Data CHW numbers: ", length(unique(d$data_chw_bad_numbers$storedby)))
```
</details>

#### Step B7: Email the results
**Function:** `sendStepDReportByEmail(d, d2_session)` (in `R/sendWeeklyReportByEmail.R`)

Same recipient list mechanism as Step A7, emailing both summary tables plus the zip of vouchers.

---

## 3. Anti-Gaming / Anti-Theft Guards

Several of the steps above exist specifically because the team observed real attempts to game the system in the past, and tightened the logic in response. This section pulls those guards together in four groups by what kind of cheating each one stops, and points back to where each one lives in the pipeline above.

### Duplicate-Payment Prevention

#### 3.1 Database-level duplicate-payment guard (stops the *same* weekly submission being paid twice)
**Where:** inside the SQL query itself (Step A1, `inst/sql/topup_raw_data.sql`), not in R.

The query is deliberately written to only pick up a facility/week's completion record if there is **no older** completion record for that same facility/week beyond the last 7 days. In plain terms: once a week's report has been "seen" by one weekly run, it cannot be seen (and therefore paid) again by a later run. This is the first and most fundamental protection against double-payment.

#### 3.2 De-duplication of resubmissions within R (stops using repeat submissions to manipulate the timing)
**Where:** Step A1, `R/prepareWeeklyTopupData.R`

Even within one run, if a facility somehow has more than one submission on record for the same week, only the single **latest** one is kept before the payment amount is worked out:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::group_by(orgunit_uid, orgunit_name, iso, startdate, submitted_by) %>%
dplyr::mutate(is_last_submission = submission_date == max(submission_date)) %>%
dplyr::filter(is_last_submission)
```
</details>

### Identity & Eligibility Checks

#### 3.3 Explicit exclusion list / blocklist (lets staff cut off a specific person caught gaming)
**Where:** Step A2, `R/getRapidReportUsers.R`

A dedicated DHIS2 user group (`KiTZhKdn5VL`) acts as a "do not pay" list. Anyone added to it, for example after being caught gaming the system, is flagged `is_excluded = TRUE` and is hard-blocked from payment regardless of anything else about their submission:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::mutate(is_excluded = userCredentials.username %in% excluded_users)
...
dplyr::mutate(status_good = service_provider != "Unknown" & !is_excluded & amount != 0)
```
</details>

This is the manual "off switch" the team can flip for an individual user without touching the general rules.

#### 3.4 Phone number / mobile-network validation (stops fake, mistyped, or non-mobile-money numbers from being paid)
**Where:** Steps A2 and B2, `R/classifyMobileOperator.R`

Every phone number is cleaned up and checked against the real prefix patterns used by Zambia's three mobile networks. Anything that doesn't genuinely match one of those patterns is marked `"Unknown"` and is excluded from payment; it goes into a "bad numbers" file for staff to follow up instead:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::case_when(
  stringr::str_detect(phone, "^(076|096|073)[0-9]{7}$") ~ "MTN",
  stringr::str_detect(phone, "^(097|077)[0-9]{7}$")     ~ "Airtel",
  stringr::str_detect(phone, "^095[0-9]{7}")             ~ "Zamtel",
  TRUE ~ "Unknown")
```
</details>

#### 3.5 Data CHW identity/hierarchy binding (stops payments landing on orphan, duplicate, or admin accounts)
**Where:** Step B2, `R/getDataCHWs.R`

Data CHW payments are only ever linked to user accounts that are (a) currently active (`disabled == FALSE`) and (b) sit at the deepest, front-line level of the organisation hierarchy (their assigned health-post level, not a district/province-level account), keeping only the deepest assignment if a person has more than one:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::filter(disabled == FALSE) %>%
dplyr::mutate(level = stringr::str_count(path, "/")) %>%
dplyr::filter(level >= 4) %>%
dplyr::group_by(data_chw_phone, data_chw_username, data_chw_name) %>%
dplyr::filter(level == max(level))
```
</details>

#### 3.6 Proxy-submission detection (currently a *flag*, not yet an automatic block; a gap worth raising)
**Where:** Step A4, `R/prepareWeeklyTopupData.R`

The weekly job compares the person officially registered as the facility's phone-owning user against whoever's account actually stored/submitted the report. When they don't match, it's recorded as a possible sign of account-sharing or a proxy submission being used to redirect someone else's payment:

<details>
<summary>Click to see the actual R code</summary>

```r
dplyr::mutate(submitted_by_proxy = facility_username != submitted_by)
```
</details>

**Important caveat:** this flag is calculated and kept on the data, but unlike every other guard in this section, it is **not currently wired into the pass/fail eligibility test** (`status_good`). A record flagged this way can still be paid today. This is the clearest candidate for tightening if this payment logic is reused elsewhere.

### Data & Timing Validity Checks

#### 3.7 Zero-rated / future-dated reports (stops claiming a reward before it's due, and stops backdating)
**Where:** Step A1 for the weekly job; Step B2 for Step D.

Weekly: a report for a period that hasn't been open for at least 8 days is explicitly worth **0 ZMW**: you cannot be rewarded for a period that hasn't genuinely closed yet.

Step D goes a step further: if a report's "period age" comes out **negative** (meaning it claims to cover a period that, according to the clock, hasn't ended yet relative to when the report was generated), it's an automatic disqualification, which catches clock manipulation or backdated data entry:

<details>
<summary>Click to see the actual R code</summary>

```r
report_in_future = period_age < 0,
qualifies = !(missing_mandatory_des | report_in_future | not_monthly),
```
</details>

#### 3.8 Report-completeness check (stops an empty/junk report being submitted purely to trigger a payment)
**Where:** Step B1 (counted by the SQL query) and Step B2 (checked in R)

The SQL query itself counts how many of the 9 mandatory data fields actually contain a value for that facility/month. R then requires that count to be exactly 9 before the report "qualifies" for payment; a report where most fields were left blank does not count, even if it was technically marked "complete" in DHIS2:

<details>
<summary>Click to see the actual R code</summary>

```r
missing_mandatory_des = record_count != 9,
```
</details>

#### 3.9 Period-format validation (stops malformed or manipulated period codes slipping through)
**Where:** Step B2, `R/getTopupData.R`

A monthly Step D report's period code must genuinely look like a `YYYYMM` monthly code before it qualifies:

<details>
<summary>Click to see the actual R code</summary>

```r
not_monthly = !stringr::str_detect(iso, "^20[12][0-9]{3}"),
```
</details>

### Oversight

#### 3.10 Human-in-the-loop audit trail (a standing safety net behind the automated rules)
**Where:** Steps A6/B5, and every email/Telegram send

Every single run exports the full set of rejected/flagged records (bad phone numbers, excluded users, disqualified reports, and the complete raw data) as spreadsheet attachments alongside the paid voucher files, sent to program staff every cycle. This means the system isn't a fully "black box" auto-pay process; there is always a paper trail available for a person to review and catch anything the automated rules missed.

---

## 4. Appendix: Quick Function Reference

| Function | File | One-line purpose |
|---|---|---|
| `loginToDHIS2` | `R/loginToDHIS2.R` | Authenticate to the DHIS2 server. |
| `createReportInfo` | `R/createReportInfo.R` | Start a fresh results container for a given report date, loading secret keys. |
| `getWeeklyRawData` | `R/prepareWeeklyTopupData.R` | Pull weekly submission data and compute payment amount by lateness. |
| `prepareWeeklyTopupData` | `R/prepareWeeklyTopupData.R` | Conductor for the whole weekly pipeline. |
| `getRapidReportUsers` | `R/getRapidReportUsers.R` | Pull eligible mobile-reporting users, their phone/network, and exclusion status. |
| `getOrgUnitStructure` | `R/getOrgUnitStructure.R` | Pull the province/district/facility hierarchy. |
| `getAirtimeDonorGroup` / `getTrainedByGroup` / `getTopupOrgunitGroups` | `R/getOrgunitGroupSetsMembers.R` | Pull which donor funds / who trained each facility. |
| `classifyMobileOperator` | `R/classifyMobileOperator.R` | Work out MTN/Airtel/Zamtel/Unknown from a phone number's shape. |
| `createTopupSummaryTables` | `R/createTopupSummaryTable.R` | Build the weekly donor summary table (ZMW + USD, HTML/PNG/Markdown). |
| `writeAttachmentFiles` | `R/writeAttachmentFiles.R` | Write weekly voucher/rejected-record Excel files and zip them. |
| `getReportRecipients` | `R/getReportRecipients.R` | Look up which staff email addresses should receive the report. |
| `report_users` | `R/getUserEmails.R` | Alternate helper to pull valid staff emails from a user group. |
| `sendWeeklyReportByEmail` / `sendStepDReportByEmail` | `R/sendWeeklyReportByEmail.R` | Email the summary + voucher zip to staff. |
| `sendWeeklyTelegramReport` | `R/sendWeeklyTelegramReport.R` | Post the weekly summary + zip to Telegram. |
| `getStepDTopUpData` | `R/getTopupData.R` | Pull monthly Step D submission data and completeness/date checks. |
| `getDataCHWList` / `getStep3Orgunits` | `R/getDataCHWs.R` | Pull active Data CHW accounts and facility contact info. |
| `prepareStepDTopupReport` | `R/prepareStepDTopupData.R` | Conductor for the whole Step D pipeline; builds the facility and Data CHW payment lists. |
| `createStepDSummaryTables` | `R/createStepDSummaryTables.R` | Build the Step D donor summary tables (facility + Data CHW). |
| `prepareStepDAttacments` | `R/prepareStepDAttachments.R` | Write Step D voucher/rejected-record Excel files and zip them. |
| `sendStepDTelegramReport` | `R/sendStepDTelegramReport.R` | Post the Step D summary + zip to Telegram. |
| `parseDHISConf` | `R/parseDHISConfig.R` | Legacy helper to read a `key=value` config file (largely superseded by `datimutils`' own config handling). |

---

## 5. Summary

Health workers are paid airtime **only when they submit a report**, and **more the sooner they submit it** after their reporting period closes; someone who never reports is never paid, and someone who reports very late still gets a small token payment rather than nothing, to keep incentivizing eventual data completeness. Before any payment is calculated, every report passes through a chain of validity and anti-fraud checks: a database-level guard against paying the same submission twice, an application-level de-duplication step, phone-number/network validation, blocking of reports for periods that haven't closed yet or are backdated, a completeness check on the underlying data, valid-period-format checks, and identity/hierarchy checks that tie Data CHW payments to real, currently active front-line accounts. Anything rejected by these checks is routed into a "for review" file rather than silently dropped, giving program staff a standing audit trail every cycle. One known gap: the system currently detects, but does not yet automatically block, a mismatch between the officially registered reporter and whoever actually submitted a report; a candidate rule to harden if this logic is reused elsewhere.
