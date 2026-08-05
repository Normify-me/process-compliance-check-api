<img width="200"  alt="pb_normify_grey" src="https://github.com/user-attachments/assets/b3777e1c-20a3-42e7-a7ff-c8af17c0ca94" />

<img width="200"  alt="pb_normify_white" src="https://github.com/user-attachments/assets/56a453c7-5417-4e73-bc45-857c1a6c9062" />

# Normify Process Compliance Check API

## Overview

The Process Compliance Check API analyzes business process documentation against one or more Normify standards. It maps processes to norm chapters, identifies coverage gaps, and returns requirement-level findings with evidence, relevance, and recommendations.

## Standards

In addition to the following management standards, any other legal bases or standards (https://normify.me/data/) for which full texts are available may also be used for analysis.

| **Category**                | **Standard**               | **English Title**                                      | **normify_identifier** |
|-----------------------------|----------------------------|--------------------------------------------------------|------------------------|
| **Standard**                | ISO 9001                   | Quality Management Systems – Requirements              | [DmBUbhYLyK6BDimPv7bHtZ](https://app.normify.me/research/standards/detail/DmBUbhYLyK6BDimPv7bHtZ/){:target="_blank"}             |
| **Security & IT**           | ISO/IEC 27001              | Information Security Management Systems – Requirements | [oHFBv3JyrjDW5LLbYfnnUo](https://app.normify.me/research/standards/detail/oHFBv3JyrjDW5LLbYfnnUo/){:target="_blank"}

Analysis is **asynchronous**:

1. `POST` starts the job and returns an `analysis_id` (HTTP 202).
2. Poll `GET` with that `analysis_id`. (optional: Webhook)

The input given by the user is a **full process landscape** (multiple processes).

## Base URLs

| Environment | Base |
| --- | --- |
| Production | `https://app.normify.me/compliance-check/api/check/` |
| Local | `http://localhost:8000/compliance-check/api/check/` |

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
