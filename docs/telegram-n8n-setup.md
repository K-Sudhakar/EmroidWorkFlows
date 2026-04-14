# Telegram and n8n Setup Guide

This guide configures Telegram as the user channel, n8n as the orchestration layer, and FastAPI as the processing backend.

## Architecture

Flow:

1. User sends `/start`, `/help`, or an SVG document to the Telegram bot.
2. n8n receives the message with `Telegram Trigger`.
3. n8n downloads the Telegram file into binary field `data`.
4. n8n sends the file to FastAPI with `POST /jobs`.
5. FastAPI queues and processes the job.
6. FastAPI calls the n8n callback URL when the job reaches `completed` or `failed`.
7. n8n calls `GET /jobs/{job_id}` to fetch the final canonical status.
8. n8n sends a Telegram completion or failure message.

n8n does not run Ink/Stitch, Inkscape, or any embroidery conversion command.

## Telegram Bot Token

1. Open Telegram and message `@BotFather`.
2. Run `/newbot`.
3. Choose a bot name and username.
4. Copy the bot token.
5. In n8n, create a Telegram credential:
   - Credential type: `Telegram API`
   - Access token: the token from BotFather

Do not commit the Telegram token to the repository.

## n8n Import

1. Open n8n.
2. Go to `Workflows`.
3. Import `workflows/telegram-embroidery-job-callback.workflow.json`.
4. Open these Telegram nodes and select the Telegram credential:
   - `Telegram Trigger`
   - `Send Help Message`
   - `Send Unsupported Message`
   - `Send File Received`
   - `Send Processing Started`
   - `Send Job Create Failure`
   - `Send Completion Message`
   - `Send Failure Message`
   - `Send Nonterminal Status`
5. Set the backend base URL through the n8n environment variable `BACKEND_API_BASE_URL`.

Recommended local value when n8n is running in Docker and FastAPI is running on the host:

```bash
BACKEND_API_BASE_URL=http://host.docker.internal:8000
```

Recommended local value when both run directly on the same host:

```bash
BACKEND_API_BASE_URL=http://127.0.0.1:8000
```

## n8n Webhook URL

The workflow uses a callback URL from the n8n `Wait For Backend Callback` node. For FastAPI to call it, n8n must know its externally reachable base URL.

For hosted n8n, set:

```bash
WEBHOOK_URL=https://n8n.example.com/
```

For local development, expose n8n with a tunnel:

```bash
WEBHOOK_URL=https://your-tunnel.example.com/
```

FastAPI must be able to reach this URL from wherever the backend worker runs.

## Backend API Base URL

Set `BACKEND_API_BASE_URL` in the n8n runtime, not inside the workflow JSON.

Examples:

```bash
BACKEND_API_BASE_URL=http://host.docker.internal:8000
BACKEND_API_BASE_URL=https://api.example.com
```

Do not include a trailing slash.

## FastAPI Callback Requirement

The workflow sends `callback_url` as a form field in `POST /jobs`. The backend must call that URL when the job reaches a terminal state:

```json
{
  "job_id": "job_123",
  "status": "completed"
}
```

n8n then calls `GET /jobs/{job_id}` and sends the user either:

- `Your file is ready. Download it here: <link>`
- `Processing failed. Reason: <safe reason>`

## Download URL Exposure

The `download_url` returned by FastAPI must be reachable by the Telegram user.

Local development:

- Use an HTTPS tunnel to the backend for downloads.
- Set the backend public download base URL to the tunnel URL.
- Keep the internal n8n-to-backend API URL separate from public download URLs.

Production:

- Use HTTPS.
- Prefer signed, short-lived URLs.
- Do not expose local paths or internal service names.

## Error Handling

The workflow has separate failure paths:

- Unsupported Telegram message: asks the user to send an SVG.
- `POST /jobs` failure: sends `Processing failed. Reason: ...`.
- Backend terminal failure: sends the backend `safe_error`.
- Nonterminal callback/status mismatch: sends the job ID and current status.

Backend errors should be normalized to safe user-facing messages. Keep stack traces and command output in backend logs only.

## Retry and Timeout Guidance

Recommended backend behavior:

- Retry the n8n callback 3 times with short exponential backoff.
- Log callback failures with `job_id`, terminal status, and HTTP response status.
- Keep job status durable so a missed callback can be recovered manually.

Recommended n8n behavior:

- Keep execution data enabled while testing.
- Add an error workflow for operational alerts.
- Avoid adding file validation or conversion retries in n8n; those belong in FastAPI or the job worker.
