<img width="200"  alt="pb_normify_grey" src="https://github.com/user-attachments/assets/b3777e1c-20a3-42e7-a7ff-c8af17c0ca94" />

<img width="200"  alt="pb_normify_white" src="https://github.com/user-attachments/assets/56a453c7-5417-4e73-bc45-857c1a6c9062" />

# Normify Process Compliance Check API

## Overview

The API allows users to upload process diagrams or descriptions in any format.

Based on common management system standards and, where applicable, existing data in the legal registry (https://github.com/Normify-me/legal-discover-api), sections or chapters of individual standards or laws are assigned to the process, and, where appropriate, suggestions for improvement and information on overall coverage are provided.

## Standards

In addition to the following management standards, any other legal bases or standards (https://normify.me/data/) for which full texts are available may also be used for analysis.

| **Category**                | **Standard**               | **English Title**                                      |
|-----------------------------|----------------------------|--------------------------------------------------------|
| **Standard**                | ISO 9001                   | Quality Management Systems – Requirements              |
|                             | ISO 14001                  | Environmental Management Systems – Requirements        |
|                             | ISO 45001                  | Occupational Health and Safety Management Systems      |
| **Security & IT**           | ISO/IEC 27001              | Information Security Management Systems – Requirements |
|                             | ISO/IEC 27701              | Privacy Information Management Systems – Requirements |
| **Energy & Resources**      | ISO 50001                  | Energy Management Systems – Requirements              |
|                             | ISO 46001                  | Water Efficiency Management Systems – Requirements    |
| **Risk, Compliance, Resilience** | ISO 31000          | Risk Management – Guidelines                            |
|                             | ISO 37301                  | Compliance Management Systems – Requirements           |
|                             | ISO 22301                  | Business Continuity Management Systems – Requirements  |
| **Operations & Organization** | ISO 41001              | Facility Management – Management Systems – Requirements|
|                             | ISO 55001                  | Asset Management – Management Systems – Requirements    |
|                             | ISO 30401                  | Knowledge Management Systems – Requirements            |
| **Security / Supply Chain** | ISO 28000                  | Supply Chain Security Management Systems – Requirements|

---

## Base URL

```
https://app.normify.me/research/api/discover/
```

## Headers

```
Content-Type: "application/json"
```

### Authentication

The API uses API keys for authentication. Include your API key in the header of each request:

```
Authorization: Bearer YOUR_API_KEY
```

#### How to get the bearer token

Send a post request to:

**Endpoint** `https://app.normify.me/api/token/`

**Content-Type** `application/json`

**Body (raw json)**
```json
{
    "email": "YOUREMAIL",
    "password": "YOURPASSWORD"
}
```

This returns a JSON with a refresh token and an access token. Use the access token for authorization as the bearer token. It has a limited lifetime so it should be fetched once you start a new API request process.

## Endpoints

### Query Laws and Standards

**POST** `'https://app.normify.me/research/api/discover/`

Checks the current version dates and changes for a law or standard.

#### Request Body

```json
{
  "process_input": "data",
  "normify_identifier": ["string"],
  "kataster": "string"
 }
```

**Parameters:**
- `process_input`: Process Documentation
  - Pictures: PNG, JPEG
  - Documents: PDF, DOCX, PPTX,
  - Structure document: XML, ...
- `normify_identifier`: IDs of the standards you want to check (optional)
- `kataster`: IDs of the kataster whose underlying standards you would like to use to check the process

If a search input is provided, the API returns a list of relavent law/standards. 

#### Response

```json
{
  "overall_coverage": "boolean",

  "standard": [
        {
            "identifier": "string"
            "name": "string"
            "coverage": "decimal"
            "recomendations": "text"
        }
   ]
}
```

**Response Fields:**
- `overall_coverage`: Total coverage across all selected standards, in percent
- `identifier`: Short title of the standard
- `name`: Complete title of the standard
- `coverage`: coverage of a standasrd in percent
- `recomandations`: Suggestions for improvement to increase coverage with respect to a standard


## Examples

Further examples can be found in the [Example folder](./Examples/).

### Example 1: Query with ID

**Request:**
```
curl --location 'https://app.normify.me/research/api/discover/' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer ey************' \
--data '{
"search_input": "Mercedes Benz"
}'

```

## Error Handling

### HTTP Status Codes

- `200 OK`: Request successful
- `400 Bad Request`: Invalid request (missing parameters, invalid date format)
- `401 Unauthorized`: Invalid or missing API key
- `404 Not Found`: Law or standard not found
- `500 Internal Server Error`: Server error

### Error Response Format

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
### Common Error Codes

- `MISSING_PARAMETERS`: At least one parameter is required
- `INVALID_DATE_FORMAT`: Invalid date format
- `STANDARD_NOT_FOUND`: Law or standard not found

## Disclaimer:

This repository and its contents (code, documentation, templates, etc.) may reference, implement, or be inspired by international standards such as ISO 9001, ISO 14001, ISO 27001, and others. However, you are solely responsible for:

Obtaining and using official copies of any referenced standards from the International Organization for Standardization (ISO) or its authorized distributors.
Ensuring that your use of such standards complies with all applicable laws, regulations, and licensing terms set by ISO or the relevant standards body.
Verifying that your intended use (e.g., implementation, certification, or commercial use) is permitted under the licenses you hold for these standards.
We do not provide, distribute, or license any ISO standards or other normative documents. Any references to standards in this repository are for informational and educational purposes only and do not replace the official documents.




