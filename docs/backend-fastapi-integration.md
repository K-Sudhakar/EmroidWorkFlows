# FastAPI Integration Contract

This project workspace did not contain the existing FastAPI backend source when the integration was added. Use this contract to align the backend with the n8n workflow in `workflows/telegram-embroidery-job-callback.workflow.json`.

The design keeps n8n as orchestration only. n8n receives Telegram updates, downloads the Telegram file, submits it to FastAPI, waits for a backend callback, checks final status, and sends a Telegram message. FastAPI owns all SVG validation, embroidery processing, Ink/Stitch or Inkscape invocation, output storage, error normalization, and download authorization.

## Required Endpoints

### `POST /jobs`

Accept a Telegram-originated job as `multipart/form-data`.

Form fields:

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `file` | file | yes | SVG uploaded by the Telegram user. n8n sends the downloaded Telegram binary in the `data` binary field. |
| `source` | string | yes | Expected value: `telegram`. |
| `telegram_chat_id` | string | yes | Used for traceability only. Telegram sending remains in n8n. |
| `telegram_user_id` | string | no | Used for audit or quota checks if the backend already has them. |
| `original_filename` | string | no | Original Telegram document name. |
| `callback_url` | string | yes | n8n wait/resume webhook URL. FastAPI calls this when the job reaches a terminal state. |

Expected response:

```json
{
  "job_id": "job_123",
  "status": "queued",
  "download_url": null,
  "file_name": null
}
```

Notes:

- Return quickly after queueing the job. Do not block the HTTP request until embroidery processing completes.
- Keep all file validation and processing in FastAPI.
- Reject invalid files with a safe error body such as `{"safe_error": "Only SVG files are supported."}`.
- Do not expose local filesystem paths in responses.

### `GET /jobs/{job_id}`

Return the current job status.

Expected response while running:

```json
{
  "job_id": "job_123",
  "status": "processing",
  "download_url": null,
  "file_name": null,
  "safe_error": null
}
```

Expected response on success:

```json
{
  "job_id": "job_123",
  "status": "completed",
  "download_url": "https://example.com/downloads/job_123?token=short-lived-token",
  "file_name": "design.dst",
  "safe_error": null
}
```

Expected response on failure:

```json
{
  "job_id": "job_123",
  "status": "failed",
  "download_url": null,
  "file_name": null,
  "safe_error": "The design could not be converted."
}
```

Use only these terminal status values for n8n routing:

- `completed`
- `failed`

## Completion Callback

When a job reaches `completed` or `failed`, FastAPI should call the `callback_url` received in `POST /jobs`.

Request:

```http
POST {callback_url}
Content-Type: application/json
```

Body:

```json
{
  "job_id": "job_123",
  "status": "completed"
}
```

n8n will use this callback only to resume the workflow. It then calls `GET /jobs/{job_id}` to fetch the final canonical status and download URL from FastAPI.

## FastAPI Shape

The exact implementation depends on the existing backend queue, but the endpoint shape should look like this:

```python
from typing import Annotated

from fastapi import APIRouter, BackgroundTasks, File, Form, UploadFile

router = APIRouter()


@router.post("/jobs")
async def create_job(
    background_tasks: BackgroundTasks,
    file: Annotated[UploadFile, File()],
    source: Annotated[str, Form()],
    telegram_chat_id: Annotated[str, Form()],
    callback_url: Annotated[str, Form()],
    telegram_user_id: Annotated[str | None, Form()] = None,
    original_filename: Annotated[str | None, Form()] = None,
):
    # Keep validation and processing inside backend services.
    job = await job_service.enqueue(
        file=file,
        source=source,
        telegram_chat_id=telegram_chat_id,
        telegram_user_id=telegram_user_id,
        original_filename=original_filename,
        callback_url=callback_url,
    )
    return {
        "job_id": job.id,
        "status": job.status,
        "download_url": None,
        "file_name": None,
    }
```

In the worker or job finalizer:

```python
import httpx


async def notify_job_complete(callback_url: str, job_id: str, status: str) -> None:
    async with httpx.AsyncClient(timeout=10) as client:
        await client.post(callback_url, json={"job_id": job_id, "status": status})
```

Call this after the backend has persisted the final job status. If the callback fails, log it with the job ID and retry a small number of times with backoff. Do not mark embroidery processing failed only because Telegram notification failed.

## Download Links

Return a URL in `download_url` that a Telegram user can open directly.

Local development options:

- Run the backend on the host and use a tunnel such as ngrok or Cloudflare Tunnel for the public download base URL.
- If n8n runs in Docker and FastAPI runs on the host, n8n can call `http://host.docker.internal:8000`, but Telegram users still need a public HTTPS download URL.

Production guidance:

- Use HTTPS.
- Prefer short-lived, signed download URLs.
- Do not return filesystem paths, internal container hostnames, or private IP addresses.
- Keep authorization in FastAPI or the storage layer, not in n8n.

## Polling Fallback

The provided workflow is callback-driven. If callback support cannot be added yet, replace `Wait For Backend Callback` with a bounded polling section:

1. Wait 10 to 30 seconds.
2. Call `GET /jobs/{job_id}`.
3. If `completed`, send the completion message.
4. If `failed`, send the failure message.
5. If still running and the attempt count is below the configured limit, loop back to the wait node.
6. On timeout, send a neutral message with the job ID.

Keep the attempt counter and timeout handling in n8n only as orchestration state. Do not copy job validation or conversion rules into n8n.
