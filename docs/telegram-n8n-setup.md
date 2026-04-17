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
5. Open `Normalize Telegram Message` and set:
   - `backendApiBaseUrl` to your FastAPI base URL.
   - `publicN8nBaseUrl` to the public n8n URL that the backend can call.

Recommended local value when n8n is running in Docker and FastAPI is running on the host:

`http://host.docker.internal:8000`

Recommended local value when both run directly on the same host:

`http://127.0.0.1:8000`

The workflow intentionally does not use `$env.BACKEND_API_BASE_URL`, because some n8n deployments block expression access to environment variables.

## n8n Webhook URL

The workflow uses a callback URL from the n8n `Wait For Backend Callback` node. For FastAPI to call it, n8n must know its externally reachable base URL.

The wait node also has a 30-second fallback timeout. If the backend callback is missed, the workflow resumes after 30 seconds and calls `GET /jobs/{job_id}`. This keeps Telegram updates moving even when the callback path is misconfigured.

For hosted n8n, set:

```bash
WEBHOOK_URL=https://n8n.example.com/
```

For local development, expose n8n with a tunnel:

```bash
WEBHOOK_URL=https://your-tunnel.example.com/
```

FastAPI must be able to reach this URL from wherever the backend worker runs.

If the workflow remains stuck on `Wait For Backend Callback`, check:

- FastAPI is storing the exact `callback_url` value received from `POST /jobs`.
- The worker calls `POST {callback_url}` after persisting `completed` or `failed`.
- `WEBHOOK_URL` points to the public n8n URL, not `localhost`.
- Firewalls, tunnels, or reverse proxies allow inbound calls to n8n.
- The n8n execution is still active and has not been pruned.

## Backend API Base URL

Set `backendApiBaseUrl` in the `Normalize Telegram Message` node.

Examples:

`http://host.docker.internal:8000`

`https://api.example.com`

Do not include a trailing slash.

## Public n8n Callback URL

Set `publicN8nBaseUrl` in the `Normalize Telegram Message` node.

Example when n8n is reachable directly on the VPS:

`http://72.60.97.242:5678`

Example when n8n is behind HTTPS:

`https://n8n.example.com`

The workflow uses this value to rewrite `$execution.resumeUrl` before sending `callback_url` to FastAPI. This allows the callback URL to be fixed from the workflow even when n8n's server-level `WEBHOOK_URL` is missing or wrong.

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
