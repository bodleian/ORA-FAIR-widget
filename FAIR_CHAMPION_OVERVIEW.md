# ORA FAIR Champion Overview

> A narrative guide to how ORA tests datasets against FAIR principles, stores and retrieves results,
> and surfaces them on ORA Public. For code-level implementation details, see
> [FAIR_CHAMPION_INTEGRATION.md](FAIR_CHAMPION_INTEGRATION.md).

## Table of Contents

1. [Overview](#1-overview)
2. [How Tests Are Triggered](#2-how-tests-are-triggered)
3. [Calling the FAIR Champion API](#3-calling-the-fair-champion-api)
4. [Parsing the Response](#4-parsing-the-response)
5. [Saving Results](#5-saving-results)
6. [Retrieving Results](#6-retrieving-results)
7. [Publishing FAIR Scores to ORA Public](#7-publishing-fair-scores-to-ora-public)
8. [Emailing Depositors](#8-emailing-depositors)
9. [Displaying the results (the FAIR score widget)](#9-displaying-the-results-the-fair-score-widget)
10. [Batch Processing](#10-batch-processing)
11. [Configuration Reference](#11-configuration-reference)
12. [UUID Formats](#12-uuid-formats)

---

## 1. Overview

ORA integrates with the external [FAIR Champion](https://champion.fairsharing.org) service to
evaluate published **Dataset** objects against the FAIR data principles: Findable, Accessible,
Interoperable, and Reusable. The integration exists as a pilot project that is designed to scale
to covering approximately 10,000 datasets with monthly testing of each dataset.

The workflow has four distinct stages, each handled by a dedicated component:

1. **Test**: Submit a dataset's public URL to the FAIR Champion API for evaluation.
2. **Store**: Parse the response and append a structured result to a file-based log.
3. **Surface**: At publication time, retrieve the most recent result and write it to the Solr
   index so ORA Public can display it on dataset landing pages.
4. **Notify**: Optionally email a FAIR assessment report to the dataset depositor.

File-based JSONL logging (rather than a database) was chosen deliberately to keep the pilot
simple and maintainable. The system can be extended if the project continues beyond the pilot.

---

## 2. How Tests Are Triggered

Tests enter the system through several paths, each suited to a different operational need:

### Single-Object Testing

When a test needs to run against one specific dataset — for example as part of an automated
pipeline or a manual check — the entry point is `Fair::RunTestAndLog`. This orchestrator
coordinates the API call, response parsing, and log writing in one call. It accepts a dataset
UUID in either internal (`uuid_`) or public (`uuid:`) format.

For cases where the dataset is already published and you also want to update the Solr index and
optionally email the depositor, `FairChampionUpdateService` wraps the full cycle: test, republish,
and notify.

### Batch Testing via Rake Tasks

Two rake tasks handle bulk operations, typically run via cron or manually from the command line:

- **`fair_champion:update_and_report`** — Processes datasets modified since a given date
  (default: last 24 hours). This is the task suited for nightly cron jobs. It queries Solr for
  published datasets with recent modifications and processes each one.

- **`fair_champion:update_all`** — Processes every published dataset regardless of modification
  date. Useful for initial bulk runs or periodic full re-assessment.

Both tasks accept an optional `send_reports` parameter controlling whether depositor emails are
sent. In production, emails are sent by default. In non-production environments (QA, dev, local,
test), emails are suppressed by default to avoid spamming test inboxes — pass `true` explicitly
to enable them for testing purposes.

### During Publication

The most important automatic trigger happens inside `PublishToPublic3`, the service that
handles all ORA publication events. Whenever a dataset is published or republished, this service
retrieves the most recent FAIR result from the log and writes it into the Solr index. This
means FAIR scores are always kept current on ORA Public without any manual intervention — every
time a dataset is republished (including after a fresh FAIR test), the score is refreshed.

---

## 3. Calling the FAIR Champion API

The FAIR Champion API is a single external HTTP endpoint that accepts a POST request with a JSON
body containing two fields:

- **`guid`** — The full public URL of the ORA object being tested. This is constructed from the
  configured `base_public_url` and the dataset's UUID in public format (`uuid:`).
- **`calculation_uri`** — A reference to the FAIR calculation spreadsheet that defines how scores
  are computed. A default Google Sheets URI is provided, but it can be overridden through configuration

The API can take several minutes to respond (up to six minutes in production, but normally ~20 seconds), 
as it performs a comprehensive evaluation of the object's metadata, identifiers, accessibility, and licensing. A
configurable timeout guards against indefinite hangs.

If the request fails due to network issues, timeouts, or HTTP errors, the failure is captured
as a structured error and returned to the caller. The caller (whether `RunTestAndLog`, the
update service, or a rake task) logs the error and continues to the next dataset in batch
scenarios, or returns the error to the caller in single-object scenarios.

For full details on the HTTP request format, headers, and error handling, see the
[Integration Document — API Client](FAIR_CHAMPION_INTEGRATION.md#41-fairapiclient).

---

## 4. Parsing the Response

FAIR Champion returns a JSON object with three top-level fields: `resultset` (a JSON-LD graph
containing assessment metadata), `narratives` (human-readable PASS/FAIL strings), and
`test_results` (individual sub-test outcomes).

`ParseFairChampionResult` transforms this into a clean, structured hash with the following structure, known as the
ORA FAIR abridged result format. This is described in detail in the FAIR Champion integration document.

### 4.1 PASS/FAIL determination

The PASS/FAIL determination is based on a configurable threshold (default: 50%). If a principle
achieves at least 50% compliance across its sub-tests, it is marked PASS. One principle — I2
— is not fully tested by the current FAIR Champion version; it returns an `indeterminate` status 
when absent from the response. An indeterminate result has a null score and is excluded from the 
calculation of aggregate and overall scores.

For the full parsing logic and narrative pattern matching, see the
[Integration Document — Parser](FAIR_CHAMPION_INTEGRATION.md#42-fairparsefairchampionresult).

---

## 5. Saving Results

### The Log File

Results are persisted to a JSONL (JSON Lines) file — one JSON object per line. This format was
chosen because it is append-only, human-readable, grep-friendly, and requires no database
infrastructure.

Each log entry contains the result hash in the abridged format described above in Section 4. 

### Write Safety

Log writes are thread-safe using exclusive file locking (`flock`). This ensures that concurrent
processes (e.g., multiple rake task workers or Sidekiq jobs) can safely append to the same file
without data corruption. The directory is created automatically if it does not exist.

### Log File Locations

| Environment | Path |
|---|---|
| Development | `log/fair_champion_results.jsonl` |
| Test | `log/test_fair_champion_results.jsonl` |
| Production | `/data/log/ORA4/fair_champion/results.jsonl` |

### Log Rotation

In production, log files are rotated by the system logrotate configuration. Rotated files are
compressed (`.gz`) and follow standard naming conventions. The retrieval layer (see next section)
automatically searches rotated files, so historical results are always accessible regardless of
how old they are.

---

## 6. Retrieving Results

`ResultRetriever` reads from the JSONL log files — including rotated and compressed archives — to
find results for a given dataset UUID.

### Most Recent Result

This is the primary retrieval mode, used by `PublishToPublic3` during publication. It scans log
files from newest to oldest (by file modification time) and within each file from last line to
first, returning immediately after the first match. This reverse-scan strategy makes it fast for
the common case where you want the latest result for a dataset that has been tested recently.

### All Historical Results

For reporting or analysis, `all_results` scans every log file (current plus all rotated archives),
collects every matching entry, and sorts them by timestamp with the newest first. This gives a
complete history of every FAIR test ever run against a dataset.

### Search Strategy

Both retrieval modes accept UUIDs in either format (`uuid_` or `uuid:`) and search for both
formats in each log line. This handles edge cases where older log entries might contain either
format, and ensures callers never need to worry about which format to pass.

Compressed `.gz` files are transparently decompressed — the caller does not need to know whether
a result lives in the current log file or a rotated archive.

For the full implementation including file scanning logic and compression handling, see the
[Integration Document — Retriever](FAIR_CHAMPION_INTEGRATION.md#43-fairresultretriever).

---

## 7. Publishing FAIR Scores to ORA Public

This is the critical integration point where FAIR scores become visible to end users on the
public-facing ORA website.

### The Solr Index Field

When a dataset is published or republished, `PublishToPublic3` — the service responsible for all
ORA publication events — checks whether the work type is `Dataset`. If so, it retrieves the most
recent FAIR result from the JSONL log via `ResultRetriever` and writes it into the Solr field
`display__fair_score`. The ORA FAIR abridged JSON result is serialized as a string in this field.

ORA Public (the separate public-facing Hyrax instance) stores this field in Solr and renders it on
dataset landing pages.

### When This Happens

The FAIR score is refreshed in Solr during **every** publication event:

- **Initial publication** — When a dataset first moves to the "published" workflow state.
- **Republishing after a FAIR test** — When `FairChampionUpdateService` runs a new FAIR test on
  an already-published dataset, it explicitly calls `PublishToPublic3` to re-index the Solr
  document with the fresh score.
- **Bulk publication** — Via `BulkPublish3`, which inherits the FAIR score logic from
  `PublishToPublic3`.

This design means the FAIR score on ORA Public is always the most recently computed score for
that dataset, minimising the risk of stale scores provided that tests are and subsequent republication
complete successfully.

### The Update Service Flow

`FairChampionUpdateService` orchestrates the complete update cycle in three steps:

1. **Test** — Calls `RunTestAndLog` which hits the FAIR Champion API, parses the response, and
   appends the result to the JSONL log.
2. **Republish** — If the dataset is already published, calls `PublishToPublic3` which reads the
   newly logged result and writes it to Solr.
3. **Notify** — If reporting is enabled, calls `FairChampionReporterService` to find the
   depositor's email and send them a FAIR assessment report.

---

## 8. Emailing Depositors

### Finding the Depositor

`FairChampionReporterService` locates the depositor by searching the dataset's contributors for
any of these role titles: "Data depositor", "Depositor", or "Data steward". It uses the
contributor's preferred email (falling back to standard contributor email) and display name.

If no contributor with a depositor role has an email address, the email is skipped silently —
the test result is still saved and the Solr index is still updated.

### The Email Content

`FairChampionMailer` sends an HTML email containing:

- Dataset title and public URL
- Copyright date and assessment date
- Overall FAIR score with percentage and status (PASS displayed as "PASS", FAIL displayed as
  "NOT MET")
- A detailed breakdown table for all 19 results, ordered by the FAIR hierarchy (aggregate
  principles highlighted with alternating row styling)
- An explanation of what the FAIR principles mean and a link to ORA's FAIR guidance page

### Environment Safety

In production, emails go to the actual depositor. In non-production environments (QA, dev, local,
test), emails are redirected to `ora-qa@bodleian.ox.ac.uk` and `ora-dev@bodleian.ox.ac.uk` with a
prominent banner noting where the email would have been sent in production. This prevents
accidentally emailing real depositors during testing.

---

## 9. Displaying the results in the FAIR score widget

The most recent FAIR Champion result for a dataset is stored in the ORA Public system in 
the Solr field `display__fair_score`. The ORA Public system will retrieve this field when a
dataset record page is rendered, and display the results in a FAIR score widget.

When a record is viewed in ORA Public, the *Metrics* section contains the FAIR score widget. If a 
user selects the 'Metrics' section, JavaScript code reads the FAIR score extracted from the Solr field
and displays the overall percentage score (the FAIR score). Users are prompted to click for more 
information, and if they do so will see a Nightingale rose chart displaying results against the 
individual principles.

For non-datasets, or if no results are available, the widget displays `N/A` as a no-result value.

The widget is described in more detail in the ORA Public codebase.


---

## 10. Batch Processing

### Nightly Updates

The typical production workflow uses a nightly cron job running `fair_champion:update_and_report`.
This queries Solr for all published datasets modified in the last 24 hours, runs a FAIR test
against each one, logs the results, republishes to update the Solr index, and (in production)
emails each depositor with their updated score.

The 24-hour default window means that datasets which were recently published, updated, or
otherwise modified will get fresh FAIR scores. Datasets that haven't changed are left alone,
avoiding unnecessary API calls.

### Full Re-Assessment

`fair_champion:update_all` processes every published dataset regardless of when it was last
modified. This is useful for initial bulk runs when first deploying the integration, or for
periodic full re-assessment campaigns.

### Solr Query Logic

Both rake tasks query Solr with filter filters for:
- Work type = `Dataset`
- Workflow state = `published`
- (For `update_and_report`) Modification date >= the specified since date

They process results sequentially, reporting progress and success/error counts. Individual
failures (API timeouts, parsing errors, logging failures) do not halt the batch — they are
logged and counted, and processing continues to the next dataset.

---

## 11. Configuration Reference

All settings live in `config/fair_champion.yml`, environment-specific.

| Setting | Description | Default |
|---|---|---|
| `endpoint_url` | FAIR Champion API endpoint (POST) | Required |
| `calculation_uri` | FAIR calculation spreadsheet URI | Uses Google Sheets default |
| `timeout` | HTTP request timeout in seconds | 360 (6 minutes) |
| `log_location` | Path to JSONL log file | Required |
| `base_public_url` | Base URL for public object URLs (no trailing slash) | Required |
| `pass_threshold` | Minimum compliance % for a PASS status | 50 |

The initializer at `config/initializers/fair_champion.rb` loads the `Fair` module at application
boot, making all components available throughout the application.

---

## 12. UUID Formats

ORA uses two UUID formats, and understanding which is used where is essential:

| Context | Format | Example |
|---|---|---|
| ORA4 database (Fedora/Solr) | `uuid_` prefix | `uuid_d5b64ac3-705f-4fc9-a828-652f83e3fac9` |
| Public URLs and FAIR Champion API | `uuid:` prefix | `uuid:d5b64ac3-705f-4fc9-a828-652f83e3fac9` |
| JSONL log `uuid` field | `uuid_` prefix | Always stored internally |
| JSONL log `testedguid` field | `uuid:` in URL | The public URL that was tested |

All services accept **either** format as input and normalise internally. The log always stores
the internal format. The public URL is always constructed using the public format. When
searching logs, both formats are checked to handle any historical edge cases.

For the full UUID conversion functions and normalization logic, see the
[Integration Document — UUID Format Mapping](FAIR_CHAMPION_INTEGRATION.md#3-uuid-format-mapping).

---

**Document Version:** 1.1
**Last Updated:** 2026-07-29
**Status:** Current
**Related:** [FAIR_CHAMPION_INTEGRATION.md](FAIR_CHAMPION_INTEGRATION.md) — code-level implementation reference
