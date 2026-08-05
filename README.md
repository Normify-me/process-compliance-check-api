<img width="200"  alt="pb_normify_grey" src="https://github.com/user-attachments/assets/b3777e1c-20a3-42e7-a7ff-c8af17c0ca94" />

<img width="200"  alt="pb_normify_white" src="https://github.com/user-attachments/assets/56a453c7-5417-4e73-bc45-857c1a6c9062" />

# Normify Process Compliance Check API

## Overview

The Process Compliance Check API analyzes business process documentation against one or more Normify standards. It maps processes to norm chapters, identifies coverage gaps, and returns requirement-level findings with evidence, relevance, and recommendations.

## Standards

In addition to the following management standards, any other legal bases or standards (https://normify.me/data/) for which full texts are available may also be used for analysis.

| **Category**                | **Standard**               | **English Title**                                      | **normify_identifier** |
|-----------------------------|----------------------------|--------------------------------------------------------|------------------------|
| **General**                | ISO 9001                   | Quality Management Systems – Requirements              | [DmBUbhYLyK6BDimPv7bHtZ](https://app.normify.me/research/standards/detail/DmBUbhYLyK6BDimPv7bHtZ/)            |
| **Security & IT**           | ISO/IEC 27001              | Information Security Management Systems – Requirements | [oHFBv3JyrjDW5LLbYfnnUo](https://app.normify.me/research/standards/detail/oHFBv3JyrjDW5LLbYfnnUo/)

Analysis is **asynchronous**:

1. `POST` starts the job and returns an `analysis_id` (HTTP 202).
2. Poll `GET` with that `analysis_id`. (optional: Webhook)

The input given by the user is a **full process landscape** (multiple processes).

## Base URLs

`https://app.normify.me/compliance-check/api/check/` |


## Authentication

The use of this API requires premium permissions. Please contact Normify Admins to receive them.

```
Authorization: Bearer YOUR_ACCESS_TOKEN
```

### How to get the bearer token

**POST** `https://app.normify.me/api/token/`

```json
{
  "email": "YOUREMAIL",
  "password": "YOURPASSWORD"
}
```

Use the returned `access` token as the bearer token.

## Endpoints

### 1. Start analysis

**POST** `/compliance-check/api/check/`

Starts an asynchronous compliance analysis. Nothing runs in the HTTP request; work is enqueued on Celery.

#### Request

- **Content-Type**: `multipart/form-data` (recommended when uploading a file) or JSON (text-only)
- **Required**: process input (file and/or text) **and** at least one standard identifier

| Field | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `process_input_file` | file | one of file/text | — | Process documentation upload |
| `process_input_text` | string | one of file/text | — | Optional process description text |
| `normify_identifiers` | string / list | yes | — | One or more standard `normify_identifier` values (comma-separated, JSON array, or repeated form fields) |
| `webhook_url` | string (URL) | no | — | Callback URL notified when the analysis finishes |
| `save_process_file` | bool | no | `false` | If `true`, persist the uploaded file on the analysis; otherwise the file is only passed to the worker |

#### Accepted file types

```
.pdf, .txt, .md, .json, .yml, .yaml, .html, .xml,
.doc, .docx, .rtf, .odt, .ppt, .pptx, .xls, .xlsx, .csv,
.png, .jpg, .jpeg, .gif, .webp, .zip
```

ZIP archives may contain YAML/YML process files that are merged and cleaned for analysis.

#### Example (curl)

```bash
curl -X POST "https://app.normify.me/compliance-check/api/check/" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -F "process_input_file=@/path/to/prozesse.zip" \
  -F "process_input_text=Optional complementary description" \
  -F "normify_identifiers=<NORMIFY_UUID_1>,<NORMIFY_UUID_2>" \
  -F "webhook_url=https://customer.example.com/hooks/normify"
```

#### Success response (`202 Accepted`)

```json
{
  "success": true,
  "data": {
    "analysis_id": "4f18cb4b-fb2b-42e0-87e9-43f10e44eb72",
    "task_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "status": "pending"
  }
}
```

| Field | Description |
| --- | --- |
| `analysis_id` | Stable id used to poll results (`GET`) and included in the webhook payload |
| `task_id` | Task id of the enqueued analysis runner |
| `status` | Initial status (`pending`) |

---

### 2. Poll analysis status / results

**GET** `/compliance-check/api/check/<analysis_id>/`

Returns progress while running, and the full result when finished.

#### Example

```bash
curl -X GET "https://app.normify.me/compliance-check/api/check/4f18cb4b-fb2b-42e0-87e9-43f10e44eb72/" \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN"
```

#### Response while pending / running (`200 OK`)

```json
{
  "success": true,
  "data": {
    "analysis_id": "4f18cb4b-fb2b-42e0-87e9-43f10e44eb72",
    "status": "running",
    "progress": {
      "completed": 2,
      "total": 5,
      "percent": 40.0
    },
    "failure_message": ""
  }
}
```

#### Response when completed (`200 OK`)

```json
{
  "success": true,
  "data": {
    "analysis_id": "4f18cb4b-fb2b-42e0-87e9-43f10e44eb72",
    "status": "completed",
    "progress": {
      "completed": 5,
      "total": 5,
      "percent": 100.0
    },
    "failure_message": "",
    "summary": {
      "overall_coverage": 82.4,
      "matched_requirements": 143,
      "partially_matched": 18,
      "mentioned_requirements": 5,
      "missing_requirements": 31,
      "standards_analyzed": 2
    },
    "result": {
      "standards": [
        {
          "normify_identifier": "…",
          "identifier": "DIN EN ISO 9001",
          "name": "Quality Management Systems – Requirements",
          "status": "completed",
          "coverage": 88.7,
          "process_summary": "Kurzfassung der Prozesslandschaft.",
          "failure_message": "",
          "chapter_mappings": [
            {
              "kind": "mapped",
              "chapter": "4.4",
              "title": "Quality Management System and its Processes",
              "relevance": "high",
              "reason": "…",
              "process_title": "Produktion",
              "process_normify_id": "…",
              "process_external_id": "id-prod",
              "recommended_process_title": ""
            },
            {
              "kind": "uncovered",
              "chapter": "9.2",
              "title": "Internal Audit",
              "relevance": "medium",
              "reason": "Kein Auditprozess vorhanden.",
              "process_title": "",
              "process_normify_id": "",
              "process_external_id": "",
              "recommended_process_title": "Internes Audit"
            }
          ],
          "requirements": [
            {
              "chapter": "4.4",
              "title": "Quality Management System and its Processes",
              "requirement": "…",
              "status": "matched",
              "relevance": "high",
              "reason": "…",
              "process_title": "Produktion",
              "process_normify_id": "…",
              "process_external_id": "id-prod",
              "evidence": [
                {
                  "source": "produktion.yaml",
                  "page": null,
                  "text": "…"
                }
              ],
              "recommendations": []
            }
          ]
        }
      ]
    }
  }
}
```


##### Top-level `data` fields

| Field | Description |
| --- | --- |
| `success` | Always `true` for successful HTTP responses (errors use `success: false`; see [Error handling](#error-handling)). |
| `data.analysis_id` | Stable id of this analysis (same as returned by `POST`). |
| `data.status` | Overall analysis status (`pending`, `running`, `completed`, or `failed`). |
| `data.progress.completed` | Number of finished worker jobs so far. |
| `data.progress.total` | Total worker jobs for this analysis. |
| `data.progress.percent` | Progress percentage (`completed / total × 100`, rounded to one decimal). |
| `data.failure_message` | Empty on success; otherwise a human-readable note about job-level or analysis failures (even when overall status is `completed`). |
| `data.summary` | Aggregate counts and coverage across all standards (only present when status is `completed` or `failed`). |
| `data.result` | Full per-standard result payload (only present when status is `completed` or `failed`). |

##### `summary` fields

| Field | Description |
| --- | --- |
| `overall_coverage` | Weighted requirement coverage across all selected standards (0–100). See coverage formula under [Requirement status values](#requirement-status-values). |
| `matched_requirements` | Count of requirements with status `matched`. |
| `partially_matched` | Count of requirements with status `partial`. |
| `mentioned_requirements` | Count of requirements with status `mentioned`. |
| `missing_requirements` | Count of requirements with status `missing`. |
| `standards_analyzed` | Number of standards included in `result.standards`. |

##### `result.standards[]` fields

| Field | Description |
| --- | --- |
| `normify_identifier` | Normify id of the standard (same values accepted in `normify_identifiers` on create). |
| `identifier` | Short/public identifier of the standard (e.g. `DIN EN ISO 9001`). |
| `name` | Full display name of the standard. |
| `status` | Per-standard analysis status (`pending`, `running`, `completed`, or `failed`). |
| `coverage` | Weighted requirement coverage for this standard only (0–100), or `null` if not yet available. |
| `process_summary` | Short AI summary of how the process landscape relates to this standard. |
| `failure_message` | Empty on success; otherwise the failure reason for this standard’s analysis. |
| `chapter_mappings` | Chapter mappings and uncovered chapters for this standard. |
| `requirements` | Requirement-level findings with evidence and recommendations. |

##### `chapter_mappings[]` fields

| Field | Description |
| --- | --- |
| `kind` | `mapped` if a process covers the chapter; `uncovered` if no suitable process was found. |
| `chapter` | Chapter or clause number in the standard (e.g. `4.4`). |
| `title` | Title of that chapter/clause. |
| `relevance` | Importance of the chapter for the submitted processes: `high`, `medium`, or `low`. |
| `reason` | Explanation of the mapping or why the chapter is uncovered. |
| `process_title` | Title of the matched process (`mapped` only; empty for `uncovered`). |
| `process_normify_id` | Normify-assigned UUID of the matched process (`mapped` only; empty string otherwise). |
| `process_external_id` | Source-system process id when available (e.g. from YAML `_id`; empty if none). |
| `recommended_process_title` | Suggested title for a missing process (`uncovered` only; empty for `mapped`). |

##### `requirements[]` fields

| Field | Description |
| --- | --- |
| `chapter` | Chapter or clause number the requirement belongs to. |
| `title` | Title of that chapter/clause. |
| `requirement` | Text of the requirement being assessed. |
| `status` | How well the process addresses the requirement: `matched`, `partial`, `mentioned`, or `missing`. |
| `relevance` | Importance of the requirement for the submitted processes: `high`, `medium`, or `low`. |
| `reason` | Explanation of the classification. |
| `process_title` | Title of the process used as primary evidence (empty if none). |
| `process_normify_id` | Normify UUID of that process (empty string if none). |
| `process_external_id` | Source-system process id when available (empty if none). |
| `evidence` | List of supporting excerpts from the submitted documentation. |
| `recommendations` | Suggested actions to improve coverage (often empty when `status` is `matched`). |

##### `evidence[]` fields

| Field | Description |
| --- | --- |
| `source` | Origin of the excerpt (e.g. uploaded filename). |
| `page` | Page number when applicable (e.g. PDF); otherwise `null`. |
| `text` | Quoted or summarized evidence text. |


#### Status values

| Status | Meaning |
| --- | --- |
| `pending` | Created, work not finished |
| `running` | In progress |
| `completed` | Finished (may still include partial job failures in `failure_message`) |
| `failed` | Analysis failed |

#### Requirement `status` values

| Status | Score (for coverage) |
| --- | --- |
| `matched` | 1.00 |
| `partial` | 0.50 |
| `mentioned` | 0.25 |
| `missing` | 0.00 |

Relevance weights: `high` = 3, `medium` = 2, `low` = 1.

Coverage =

```
Σ(requirement score × relevance weight) / Σ(relevance weight) × 100
```

---

## Webhooks

If `webhook_url` was provided on create, Normify POSTs to that URL when the analysis first reaches a terminal status (`completed` or `failed`):

```json
{
  "analysis_id": "4f18cb4b-fb2b-42e0-87e9-43f10e44eb72",
  "status": "completed"
}
```

Use `analysis_id` to fetch the full result via `GET /compliance-check/api/check/<analysis_id>/`.

---

## Error handling

### Error response format

```json
{
  "success": false,
  "error": {
    "code": "string",
    "message": "string",
    "details": {}
  }
}
```

### Common error codes

| Code | Status | Meaning |
| --- | --- | --- |
| `MISSING_INPUT` | 400 | Neither `process_input_file` nor `process_input_text` provided |
| `INVALID_FILE_TYPE` | 400 | Uploaded file extension is not allowed |
| `MISSING_STANDARDS` | 400 | No `normify_identifiers` provided |
| `STANDARDS_NOT_FOUND` | 400 | None of the identifiers resolved to a standard |
| `INVALID_WEBHOOK_URL` | 400 | `webhook_url` is not a valid URL |
| `NOT_FOUND` | 404 | Unknown `analysis_id` (or not owned by the caller) |
| — | 401 | Missing/invalid JWT |
| — | 403 | Authenticated but missing `users.compliance_check_api` |

---

## Disclaimer:

This repository and its contents (code, documentation, templates, etc.) may reference, implement, or be inspired by international standards such as ISO 9001, ISO 14001, ISO 27001, and others. However, you are solely responsible for:

Obtaining and using official copies of any referenced standards from the International Organization for Standardization (ISO) or its authorized distributors.
Ensuring that your use of such standards complies with all applicable laws, regulations, and licensing terms set by ISO or the relevant standards body.
Verifying that your intended use (e.g., implementation, certification, or commercial use) is permitted under the licenses you hold for these standards.
We do not provide, distribute, or license any ISO standards or other normative documents. Any references to standards in this repository are for informational and educational purposes only and do not replace the official documents.
