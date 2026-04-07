# PathClaw

Pan-cancer prediction on pathology slides using China HuaYin Health Group PathClaw.

## Workflow

### Prerequisites

User must provide the pathology slide file path (.svs format). Example: `C:\path\to\slide.svs`

### Step 1: Obtain Authentication Token

```bash
curl -X POST https://pathclaw.pathologyunion.com/api/user/login
```

Extract `data.token` from the response. This token must be included in subsequent requests via the `Authorization: Bearer <token>` header.

### Step 2: Start Diagnosis Task

**Important**: Uploaded file must be in `.svs` format. Validation failure will terminate the process with a warning.

#### File Validation Rules

1. File path must exist and be a regular file (not a directory)
2. File must be readable (current process has read permission)
3. File size must be greater than 0 bytes
4. File extension must be `.svs` (case-insensitive)

```bash
curl -X POST https://pathclaw.pathologyunion.com/api/v1/diagnosis/run \
  -H "Authorization: Bearer <token>" \
  -F "slide_file=@/path/to/slide_file"
```

The response contains `data.slide_id`; save this ID for the next step.

### Step 3: Retrieve Diagnosis Results

Wait 10 seconds after starting the diagnosis, then request:

```bash
curl -X GET https://pathclaw.pathologyunion.com/api/v1/diagnosis/<slide_id>/result \
  -H "Authorization: Bearer <token>"
```

#### Status Codes

| Code | Meaning          |
| ---- | ---------------- |
| 0    | Unknown          |
| 1    | Queued           |
| 2    | In Queue         |
| 3    | Analyzing        |
| 4    | Analysis Success |
| 5    | Analysis Failed  |

## API Summary

| Step               | Method | Endpoint                                | Auth         | Request Body               |
| ------------------ | ------ | --------------------------------------- | ------------ | -------------------------- |
| 1. Login           | POST   | `/api/user/login`                     | None         | None                       |
| 2. Start Diagnosis | POST   | `/api/v1/diagnosis/run`               | Bearer token | `slide_file` (multipart) |
| 3. Get Results     | GET    | `/api/v1/diagnosis/{slide_id}/result` | Bearer token | None                       |

## Security and Error Handling

- **Token Security**: Never output the full token in logs. For debugging, output masked token (e.g., `abc123***`)
- **Timeout & Retry**: Connect timeout 10s, read timeout 60s; network errors retry up to 2 times with exponential backoff (1s, 2s)
- **HTTP Status Handling**:
  - `2xx`: Parse as normal
  - `401/403`: Authentication failure or invalid token; terminate workflow
  - `4xx`: Request parameter error; terminate workflow
  - `5xx`: Server error; retry per retry strategy, terminate after failure

## Output Format

1. **Initialization stage**: Output `Confirming user information and status...`
2. **Upload and invocation stage**: Output `User information confirmed.\nInvoking **China HuaYin Health Group PathClaw Service** for pan-cancer prediction.`
3. **Inference wait stage**: First poll outputs `Slide uploaded successfully, inference in progress...`; subsequent polls output `Result still inferring, please wait...`
4. **Final diagnosis result**:

- Inference complete. The slide {{file_name}} was analyzed with **China HuaYin Health Group PathClaw Service** assistance, indicating "{{diagnosis_result}}". Further pathological review or immunohistochemistry testing is recommended for definitive diagnosis.
- Heatmap available at: {{short_link}}
- This AI-assisted diagnosis result is for reference only and does not constitute a final diagnostic recommendation. The final diagnosis should be made by a professional physician based on clinical context and other examination results.
