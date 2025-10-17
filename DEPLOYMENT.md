# Deployment and Production Configuration Guide

This document outlines how to deploy the Dr. Nobu AI WhatsApp Health Assistant to production, configure webhooks, and enable hardening features.

1) Prerequisites
- Python: 3.12.x
- Production DB: PostgreSQL (recommended). Set DATABASE_URL accordingly.
- Public HTTPS domain for your API (TLS required).
- Twilio WhatsApp Sender approved and configured.
- PayDunya Business account with production keys enabled.

2) Environment Variables (.env)
Set the following in production:
- APP_BASE_URL=https://your-api.domain.tld
- ENV=prod
- LOG_LEVEL=INFO
- DATABASE_URL=postgresql+psycopg2://user:pass@host:5432/db_name
- OPENAI_API_KEY=sk-...
- TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- TWILIO_WHATSAPP_FROM=whatsapp:+<your_whatsapp_number>
- PAYDUNYA_PUBLIC_KEY=xxxxxxxx
- PAYDUNYA_PRIVATE_KEY=xxxxxxxx
- PAYDUNYA_MASTER_KEY=xxxxxxxx
- PAYDUNYA_TOKEN=xxxxxxxx
- PAYDUNYA_BASE_URL=https://app.paydunya.com/api/v1
- EMERGENCY_NUMBERS_SN=15,18
- ADMIN_API_KEY=generate-a-strong-random-secret

Note: In local development, PAYDUNYA_BASE_URL uses sandbox https://app.paydunya.com/sandbox-api/v1. In production, switch to https://app.paydunya.com/api/v1.

3) Install and Run
- Create venv and install dependencies:
  - python -m venv .venv
  - .venv/Scripts/python -m pip install --upgrade pip
  - .venv/Scripts/python -m pip install -r requirements.txt
- Run server (example uvicorn):
  - .venv/Scripts/python -m uvicorn app.main:app --host 0.0.0.0 --port 8000
- For production, run behind a process manager (e.g., systemd, Supervisor, Docker/K8s). If scaling horizontally, see Scheduler notes below.

4) Webhooks Configuration
- Twilio (WhatsApp) Inbound Webhook:
  - URL (POST): {APP_BASE_URL}/webhooks/twilio
  - Twilio signature validation is enforced automatically when ENV != "local".
    - Ensure your webhook URL matches exactly the URL Twilio posts to (including query string, protocol, and path), as the signature validation uses these.
- PayDunya IPN / Callback:
  - URL (POST): {APP_BASE_URL}/webhooks/paydunya
  - IPN hardening:
    - The service verifies the SHA-512 of your master key against the 'hash' in the callback payload.
    - It also confirms the invoice status via GET {PAYDUNYA_BASE_URL}/checkout-invoice/confirm/{token} before activating the subscription.

5) Production Hardening (already enabled)
- Twilio Signature Validation:
  - Blocks requests with invalid or missing X-Twilio-Signature when ENV != local.
- PayDunya IPN Hardening:
  - Validates SHA-512(master_key) hash.
  - Confirms invoice token status with PayDunya “confirm” endpoint before enabling access.
- Minimal Admin Endpoints (protected by X-Admin-Key):
  - GET /admin/health
  - GET /admin/user?phone=+2217xxxxxxx
  - GET /admin/payment/{token}
  - Include header: X-Admin-Key: {ADMIN_API_KEY}
  - Do not expose this key publicly.

6) Scheduler (Reminders and Expiry)
- APScheduler is started automatically at app startup:
  - Sends due reminders (every 2 minutes).
  - Checks Discovery subscriptions expired and prompts Premium upsell (every 10 minutes).
- If running multiple app instances, ensure only one instance runs the scheduler to avoid duplicate reminders. Options:
  - Use a single “worker” instance for scheduler jobs.
  - Use a distributed job scheduler (e.g., Celery/Redis or APScheduler with database-backed job store).

7) Freemium and Subscriptions
- Freemium:
  - 10 free questions per user; warning at 8; lock at 10 with upsell message.
- Upsell / Payments:
  - User replies "1" => DISCOVERY_7D (7 days, 500 CFA)
  - User replies "2" => MONTHLY_30D (30 days, 2000 CFA)
  - Bot returns a PayDunya checkout link; on completed payment (confirmed), bot activates subscription and notifies with end date.
  - On cancelled payment, a 7-day refusal follow-up reminder is scheduled automatically.

8) Internationalization and Safety
- Bot language is set by user (English, Français, Wolof).
- Responses always include a medical disclaimer.
- Emergency detector adds local emergency numbers for severe symptoms.

9) Testing
- Run unit tests:
  - .venv/Scripts/python -m pytest -q
- Current basic tests:
  - tests/test_state.py (freemium logic and pack selection)
  - tests/test_emergency.py (emergency keyword detection)

10) Observability
- Logs: structured via loguru.
- For production, configure central log collection if desired (e.g., stdout to systemd/journal, or external log shipper).

11) Data and Security Notes
- Store minimal PII. Protect secrets. Use TLS for all endpoints.
- Validate and sanitize inbound text (current flows are restricted to WhatsApp text and known IPN fields).
- Hash verification for PayDunya callbacks included; Twilio signature validation included.
- Update privacy policy and data retention as appropriate.

12) Post-Deployment Checklist
- Switch PayDunya keys to production; set PAYDUNYA_BASE_URL to production.
- Switch Twilio from sandbox to production sender; update inbound webhook URL.
- Verify:
  - /health and /docs are reachable.
  - Twilio inbound to /webhooks/twilio is accepted (signature valid).
  - PayDunya IPN to /webhooks/paydunya is accepted and confirmed.
  - Freemium throttling and upsell messaging triggers as designed.
  - Reminders and expiry jobs are firing (watch logs).
  - Admin endpoints authorized only with X-Admin-Key.
