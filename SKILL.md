---
name: PathClaw
description: "Use China Huayin Health Group PathClaw for pan-cancer prediction on pathology slides. The workflow includes: (1) obtain authentication token from login API; (2) start diagnosis task using slide_file; (3) retrieve diagnosis results. Trigger phrases: \"pathology diagnosis\", \"PathClaw\", \"病理切片诊断\", etc. **Note**: User must provide the pathology slide file path (e.g., `C:\\path\\to\\slide.svs`). If not provided, ask the user to supply it."
---

## Overview

Execute a 3-step pathology slide diagnosis workflow against server `http://119.91.47.20:8111`.

## Workflow

### Prerequisites

**User must provide the pathology slide file path** (.svs format). If not provided, ask:

```
Please provide the pathology slide file path, e.g., C:\path\to\slide.svs
```

### Step 1: Obtain Authentication Token

```bash
curl -X POST http://119.91.47.20:8111/api/user/login
```

Extract `data.token` from the response. This token must be included in subsequent requests via the `Authorization: Bearer <token>` header.

### Step 2: Start Diagnosis Task

**Important**: Uploaded file must be in `.svs` format. If validation fails, terminate and warn the user:
`Invalid pathology slide format (should be .svs format)`

#### File Validation Rules (all must pass before upload)

1. File path must exist and be a regular file (not a directory).
2. File must be readable (current process has read permission).
3. File size must be greater than 0 bytes.
4. File extension must be `.svs` (case-insensitive, e.g., `.SVS` is also allowed).
5. Any validation failure must immediately terminate the process; do not call the diagnosis API.

Validation failure example: `C:\Users\HYK\Desktop\SKILL.md` → prompt
`Invalid pathology slide format (should be .svs format)`

```bash
curl -X POST http://119.91.47.20:8111/api/v1/diagnosis/run \
  -H "Authorization: Bearer <token>" \
  -F "slide_file=@/path/to/slide_file"
```

The response contains `data.slide_id`; save this ID for the next step.

### Step 3: Retrieve Diagnosis Results

Wait 10 seconds after starting the diagnosis, then request:

```bash
curl -X GET http://119.91.47.20:8111/api/v1/diagnosis/<slide_id>/result \
  -H "Authorization: Bearer <token>"
```

**ai_diagnosis_status status codes**:

| Code | Meaning       |
| ---- | ------------- |
| 0    | Unknown       |
| 1    | Queued        |
| 2    | In Queue      |
| 3    | Analyzing     |
| 4    | Analysis Success |
| 5    | Analysis Failed |

## Security and Error Handling

1. **Token Security**

   - Never output the full token in logs.
   - For debugging only, output masked token (e.g., first 6 chars + `***`).

2. **Network and Timeout**

   - Every HTTP request must have a timeout (recommended: 10s connect timeout, 60s read timeout).
   - On timeout or network error, retry up to 2 times (exponential backoff: 1s, 2s).
   - After exceeding retry limit, return a clear failure message and stop the workflow.

3. **HTTP Status Code Handling**

   - `2xx`: Parse as normal.
   - `401/403`: Prompt authentication failure or token invalid; terminate workflow.
   - `4xx`: Prompt request parameter error (including file or interface parameter issues); terminate workflow.
   - `5xx`: Prompt server error; retry per retry strategy, terminate after failure.

4. **Result Status Handling**

   - Status `1/2/3`: Continue polling.
   - Status `4`: Output final diagnosis result.
   - Status `5`: Output "Analysis Failed" with error info; terminate workflow.
   - Status `0` or unknown: Treat as exception and terminate workflow.

## API Summary

| Step             | Method | Endpoint                              | Auth          | Request Body              |
| ---------------- | ------ | ------------------------------------- | ------------- | ------------------------- |
| 1. Login         | POST   | `/api/user/login`                   | None          | None                      |
| 2. Start Diagnosis | POST | `/api/v1/diagnosis/run`            | Bearer token  | `slide_file` (multipart) |
| 3. Get Results   | GET    | `/api/v1/diagnosis/{slide_id}/result` | Bearer token | None                      |

## Output Specification

Follow these output specifications during execution:

1. **Initialization stage**: Output `Confirming user information and status...`
2. **Upload and invocation stage**: Output `User information confirmed.\nInvoking **China HuaYin Health Group PathClaw Service** for pan-cancer prediction.`
3. **Inference wait stage**: First poll outputs `Slide uploaded successfully, inference in progress...`
   - For second and subsequent polls, output `Result still inferring, please wait...`
4. **Final diagnosis result analysis**: Plain text + unordered list, template:

- Inference complete. The slide {{file_name}} was analyzed with **China HuaYin Health Group PathClaw Service** assistance, indicating "{{diagnosis_result}}". Further pathological review or immunohistochemistry testing is recommended for definitive diagnosis.
- Heatmap available at: {{short_link}}
- This AI-assisted diagnosis result is for reference only and does not constitute a final diagnostic recommendation. The final diagnosis should be made by a professional physician based on clinical context and other examination results.
