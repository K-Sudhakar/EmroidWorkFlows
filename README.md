# Emroid Workflows

This repository contains the Telegram and n8n orchestration assets for the embroidery job backend.

The integration is intentionally thin:

- Telegram is the user interaction channel.
- n8n handles orchestration, status messages, file handoff, and backend callbacks.
- FastAPI owns SVG validation, job processing, Ink/Stitch or Inkscape execution, output storage, and download authorization.

## Deliverables

- Import-ready workflow: `workflows/telegram-embroidery-job-callback.workflow.json`
- Telegram and n8n setup guide: `docs/telegram-n8n-setup.md`
- FastAPI integration contract: `docs/backend-fastapi-integration.md`
- n8n environment example: `.env.n8n.example`

## Expected Backend Contract

n8n expects FastAPI to expose:

- `POST /jobs` as `multipart/form-data`
- `GET /jobs/{job_id}` as JSON status lookup

`POST /jobs` receives:

- `file`: SVG upload
- `source`: `telegram`
- `telegram_chat_id`
- `telegram_user_id`
- `original_filename`
- `callback_url`

`GET /jobs/{job_id}` returns:

```json
{
  "job_id": "job_123",
  "status": "completed",
  "download_url": "https://example.com/downloads/job_123?token=short-lived-token",
  "file_name": "design.dst",
  "safe_error": null
}
```

See `docs/backend-fastapi-integration.md` for the full contract and callback behavior.

## End-to-End Setup

1. Create a Telegram bot with `@BotFather` and copy the bot token.
2. Start FastAPI and confirm:
   - `POST /jobs` accepts the form-data contract.
   - `GET /jobs/{job_id}` returns canonical job status.
   - the worker calls `callback_url` on job completion or failure.
3. Start n8n with:

```bash
BACKEND_API_BASE_URL=http://host.docker.internal:8000
WEBHOOK_URL=https://your-public-n8n-url.example.com/
```

4. Import `workflows/telegram-embroidery-job-callback.workflow.json`.
5. Select your Telegram credential on every Telegram node.
6. Activate the workflow.
7. In Telegram, send `/start`.
8. Send an SVG document to the bot.
9. Confirm the bot sends:
   - `Your file was received.`
   - `Processing started. Job ID: <job_id>`
   - `Your file is ready. Download it here: <link>` or a safe failure reason.

## Local Development Notes

If n8n is in Docker and FastAPI runs on the host, use:

```bash
BACKEND_API_BASE_URL=http://host.docker.internal:8000
```

Telegram and backend callbacks need public HTTPS endpoints. Use a tunnel such as ngrok or Cloudflare Tunnel for:

- n8n `WEBHOOK_URL`
- backend public download links

Keep internal service URLs and public download URLs separate.

## Operational Notes

- Do not put Telegram tokens in workflow JSON.
- Do not expose filesystem paths in `download_url`.
- Use short-lived signed download URLs in production.
- Keep backend logs detailed and Telegram messages safe.
- Add an n8n error workflow for operational alerts after the base flow is working.
