# ORA FAIR Result Format

> A reference for other repositories or tools that want to consume the FAIR assessment result ORA
> produces for an object. This covers only the **result format** itself — what it looks like, why
> it's shaped this way, and how it's derived. It does not cover the FAIR score widget's rendering
> behaviour or how ORA stores/retrieves results internally; see
> [FAIR_CHAMPION_OVERVIEW.md](FAIR_CHAMPION_OVERVIEW.md) and
> [FAIR_CHAMPION_INTEGRATION.md](FAIR_CHAMPION_INTEGRATION.md) for those.

## Table of Contents

1. [Scope](#1-scope)
2. [Why This Format Exists](#2-why-this-format-exists)
3. [How It's Created](#3-how-its-created)
4. [What It Records](#4-what-it-records)
5. [Stability and Consumer Guidance](#5-stability-and-consumer-guidance)
6. [See Also](#6-see-also)

---

## 1. Scope

When ORA runs a FAIR Champion assessment against one of its objects, it parses the raw response
into a small, stable JSON structure referred to here as the **abridged result**. This is the value
ORA publishes per object once an assessment has been run and parsed (for example, via its
`display__fair_score` Solr field) — it is the format any consumer of ORA's FAIR data should read,
rather than the raw FAIR Champion API response.

## 2. Why This Format Exists

The raw response from the FAIR Champion API is a large JSON-LD/RDF/PROV document — thousands of
lines, with a top-level `resultset.@graph` describing the test run as a graph of `ftr:TestResult`,
`prov:Entity`, and `ftr:TestExecutionActivity` nodes, alongside `test_results`, `tests`,
`conditions`, `guidances`, and `narratives` arrays.

Its field naming is not even stable from one response to the next. For example, the `test_results`
hash keys appear as `F1_Unique` in one real response but as `ft_f1_m_metadata_id_unique` in
another, depending on which version of the test suite produced it. A consumer that depended
directly on that shape would be coupled to an externally-owned, undocumented, and occasionally
inconsistent format.

ORA instead parses the raw response down into a minimal JSON object that records only a pass/fail
status and percentage per FAIR principle, plus a small amount of test provenance. Downstream
consumers then depend on a predictable, ORA-owned contract instead of the raw third-party API
shape. This is formalised in ORA's own cross-partner change-management documentation, which names
this abridged format as the one component of the integration that must remain fully
backwards-compatible — unlike the raw FAIR Champion response format, which is externally owned and
not something ORA can guarantee.

## 3. How It's Created

1. ORA calls the FAIR Champion API, POSTing `{"guid": "<public object URL>", "calculation_uri":
   "<FAIR test configuration spreadsheet URI>"}`, and receives the raw JSON-LD response described
   above.

2. ORA then parses that raw response into the abridged structure:
   - **`testedguid`** is read from the JSON-LD graph's tested-identifier node (the entity whose
     `@id` starts with `urn:ostrails:testedidentifiernode:`), taking its `dct:identifier` value.
   - **`testinfo`** is assembled from other graph entities and the raw `tests` array: the overall
     test-run identifier (from the `urn:fairchampionoutput:*` entity), the FAIR Champion software
     version (from the `urn:fairchampionexecution:*` entity), the endpoint URL (from the first entry
     of the raw `tests` array), and an overall `status` of `"success"`, `"failure"`, or
     `"indeterminate"`, derived by counting pass/fail/indeterminate results across the raw
     `test_results`.
   - **`testresult.results`** — the per-principle scores — are deliberately **not** computed from
     `test_results` or the `conditions` weightings, because (per §2) those keys aren't stable across
     responses. Instead, each principle's result is derived by pattern-matching the raw `narratives`
     array of human-readable strings (e.g. `"PASS: 75% compliance reached for F1."`), since that
     narrative text is the one consistently-shaped artifact across real responses.

     For each FAIR principle, the achieved percentage is: `0%` if every matching narrative for that
     principle is a FAIL, `100%` if every one is a PASS, or the highest PASS percentage reached if
     the narratives are mixed. That percentage is then compared against a configurable pass
     threshold (ORA's current default is 50%) to assign a final `PASS`/`FAIL` status.

     A principle that doesn't appear in the narratives at all is recorded as
     `{"status": null, "result": null}` — except for principles FAIR Champion hasn't implemented
     yet (currently just `I2`), which are instead recorded as
     `{"status": "indeterminate", "result": null}`.

## 4. What It Records

A worked example:

```json
{
  "timestamp": "2026-07-08T14:30:00Z",
  "uuid": "uuid_d5b64ac3-705f-4fc9-a828-652f83e3fac9",
  "testedguid": "https://ora.ox.ac.uk/objects/uuid:d5b64ac3-705f-4fc9-a828-652f83e3fac9",
  "testinfo": {
    "testingendpoint": {
      "url": "https://fair-tests.fairsharing.org/test/ft_f1_m_metadata_id_unique",
      "software": "FAIR Champion",
      "version": "1.1.8"
    },
    "testdate": "2026-07-08T14:25:00+00:00",
    "status": "failure",
    "testguid": "urn:fairchampionoutput:1a968668-fe4c-4596-a3a9-a56f7c6971d6"
  },
  "testresult": {
    "results": {
      "FAIR": {"status": "FAIL", "result": "75%"},
      "F": {"status": "FAIL", "result": "75%"},
      "A": {"status": "PASS", "result": "100%"},
      "I": {"status": "PASS", "result": "100%"},
      "R": {"status": "FAIL", "result": "75%"},
      "F1": {"status": "PASS", "result": "100%"},
      "F2": {"status": "PASS", "result": "100%"},
      "F3": {"status": "PASS", "result": "100%"},
      "F4": {"status": "FAIL", "result": "0%"},
      "A1.1": {"status": "PASS", "result": "100%"},
      "A1.2": {"status": "FAIL", "result": "50%"},
      "A2": {"status": "PASS", "result": "100%"},
      "I1": {"status": "PASS", "result": "100%"},
      "I2": {"status": "indeterminate", "result": null},
      "I3": {"status": "PASS", "result": "100%"},
      "R1.1": {"status": "PASS", "result": "100%"},
      "R1.2": {"status": "FAIL", "result": "50%"},
      "R1.3": {"status": "PASS", "result": "100%"}
    }
  }
}
```

### Field reference

| Field | Description |
|---|---|
| `timestamp` | ISO 8601 UTC timestamp of when ORA parsed this result. |
| `uuid` | The ORA object's **internal** identifier format (`uuid_` prefix). |
| `testedguid` | The **public** URL that was actually submitted for testing (`https://ora.ox.ac.uk/objects/uuid:...`). Note this uses a different UUID format (`uuid:`) from the top-level `uuid` field — the two exist because one identifies the object internally and the other is the public, externally-resolvable URL FAIR Champion tested. |
| `testinfo.testingendpoint.url` | The endpoint URL FAIR Champion tested against. |
| `testinfo.testingendpoint.software` | The name of the testing software (currently always `"FAIR Champion"`). |
| `testinfo.testingendpoint.version` | The FAIR Champion software version that produced the result. |
| `testinfo.testdate` | ISO 8601 timestamp of when the FAIR Champion test run itself executed. |
| `testinfo.status` | Overall run status: `"success"`, `"failure"`, or `"indeterminate"`. |
| `testinfo.testguid` | A URN uniquely identifying this specific FAIR Champion test run. |
| `testresult.results.<principle>` | One entry per FAIR principle (see below), each `{"status": ..., "result": ...}`. |

### Principle codes

`testresult.results` always has one key per FAIR principle:

- **Aggregates**: `FAIR`, `F`, `A`, `I`, `R`
- **Individual principles**: `F1`, `F2`, `F3`, `F4`, `A1.1`, `A1.2`, `A2`, `I1`, `I2`, `I3`, `R1.1`,
  `R1.2`, `R1.3`

Two things to note:
- `R1.4` has been explicitly retired as a scored principle and will never appear.
- `I2` is currently the one principle FAIR Champion hasn't implemented yet. It's included in the
  output for forward-compatibility but will typically report `{"status": "indeterminate", "result":
  null}` rather than a real score until FAIR Champion supports it.

Each principle's value is always exactly `{"status": ..., "result": ...}`, where `status` is one of
`"PASS"`, `"FAIL"`, `"indeterminate"`, or `null`, and `result` is either a `"NN%"` string or `null`.

## 5. Stability and Consumer Guidance

This abridged JSON format is the contract ORA has committed to keeping backwards-compatible — it's
the format any other repository or tool should build against. The raw FAIR Champion API response is
not something ORA owns or guarantees the shape of, and should not be depended on directly.

## 6. See Also

- [FAIR_CHAMPION_OVERVIEW.md](FAIR_CHAMPION_OVERVIEW.md) — narrative guide to the full pipeline,
  including how ORA stores and retrieves results and displays the FAIR score widget.

