# Dr. Nobu AI WhatsApp Health Assistant (MVP)

Minimum Viable Product for an AI-powered WhatsApp health assistant with Freemium + Subscription model, OpenAI-powered responses, Paydunya payments, multilingual support (FR/Wolof/EN), emergency reminders, and per-user chat memory.

IMPORTANT MEDICAL DISCLAIMER
- The bot must always identify itself as a virtual assistant and explicitly state it is not a substitute for a real doctor.
- Severe symptoms must trigger guidance to seek urgent care and include local emergency numbers (e.g., Senegal: 15, 18).

Scope Summary (per provided brief)
- Twilio WhatsApp integration
- Freemium: 10 free Qs per user (warn at 8, lock after 10)
- Subscription via Paydunya:
  - Discovery Pack: 7 days, 500 CFA
  - Monthly Pack: 30 days, 2000 CFA
  - Webhook-based payment confirmation and state activation
- AI Health Chat using OpenAI with predefined knowledge base topics (malaria, HIV/AIDS, TB, diabetes, burns, flu, diarrhea, constipation, anxiety/mental health, eye problems, stroke symptoms, headaches, COVID‑19, joint/back pain)
- Emergency reminders for severe symptoms with local numbers
- Chat memory per user for continuity
- Out of scope: Mobile app, WAVE/Stripe/Revolut, referrals/group packs

Tech Stack (proposed)
- Language/Framework: Python 3.10+ with FastAPI
- Web Server: Uvicorn
- Messaging: Twilio WhatsApp Business API (webhook + outbound messages)
- AI: OpenAI API
- Payments: Paydunya REST API + Webhooks
- Database: SQLite for local dev (upgradeable to PostgreSQL in production) via SQLAlchemy
- Background Jobs/Scheduling: Simple in-process scheduler for MVP (upgradeable to Redis/Celery/RQ)
- Config: dotenv (.env) for secrets

Planned Project Structure
- app/
  - main.py                  # FastAPI app factory, router mounting, health checks
  - config.py                # Pydantic settings for env/config
  - deps.py                  # Common dependencies (DB session, settings)
  - i18n/
    - __init__.py
    - templates.py           # Localized message templates (FR/Wolof/EN)
    - languages.py           # Language codes and utilities
  - routers/
    - twilio_webhook.py      # POST /webhooks/twilio (inbound WhatsApp)
    - paydunya_webhook.py    # POST /webhooks/paydunya (payment updates)
    - admin.py               # Minimal status endpoints (optional)
  - services/
    - ai.py                  # OpenAI calls, prompt assembly with KB + disclaimer
    - kb.py                  # Curated knowledge base access
    - emergency.py           # Severe symptom detection and emergency numbers
    - payments.py            # Paydunya invoice creation + verification helpers
    - messaging.py           # Twilio client wrapper to send WhatsApp messages
    - state.py               # User state machine (FREEMIUM/DISCOVERY/PREMIUM)
    - memory.py              # Chat memory storage and retrieval windowing
    - scheduler.py           # Simple scheduled tasks (expiry checks, reminders)
  - db/
    - base.py                # SQLAlchemy Base
    - session.py             # DB engine + session factory
    - models.py              # Users, Payments, Conversations, Messages
    - crud.py                # DB operations
    - migrations/            # (Optional) Alembic migrations
  - schemas/
    - dto.py                 # Pydantic models for inbound/outbound payloads
- tests/
  - test_state.py
  - test_webhooks.py
  - test_emergency.py
- scripts/
  - seed_kb.py               # Seed knowledge base content (optional)
- .env.example
- requirements.txt
- README.md

Environment Variables (.env)
- APP_BASE_URL=https://your-public-host
- ENV=local|staging|prod
- LOG_LEVEL=INFO
- DATABASE_URL=sqlite:///./data.db        # For local dev; use PostgreSQL in production
- OPENAI_API_KEY=sk-...
- TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- TWILIO_WHATSAPP_FROM=whatsapp:+14155238886       # Your Twilio WhatsApp-enabled number
- PAYDUNYA_PUBLIC_KEY=xxxxxxxx
- PAYDUNYA_PRIVATE_KEY=xxxxxxxx
- PAYDUNYA_MASTER_KEY=xxxxxxxx
- PAYDUNYA_BASE_URL=https://paydunya.com/api/v1    # Verify with docs/sandbox URL
- EMERGENCY_NUMBERS_SN=15,18                       # Senegal; can add per-country mapping later

Local Setup (after files are created)
1) Ensure Python 3.10+:
   - python --version

2) Create virtual environment:
   - python -m venv .venv
   - .venv\Scripts\activate  (Windows)
   - source .venv/bin/activate (macOS/Linux)

3) Install dependencies:
   - pip install -r requirements.txt

4) Configure environment:
   - Copy .env.example to .env and fill in values.

5) Run dev server:
   - uvicorn app.main:app --reload

6) Expose public URL for webhooks (for local dev):
   - Use ngrok or similar: ngrok http http://localhost:8000
   - Update Twilio and Paydunya webhooks with your public URL.

Twilio WhatsApp Configuration
- Use Twilio sandbox or your approved WhatsApp sender.
- Set inbound webhook to: POST {APP_BASE_URL}/webhooks/twilio
- We will validate Twilio signatures and send replies via Twilio Messages API.
- Testing: Send a WhatsApp message to the Twilio sandbox; observe logs and bot reply.

Paydunya Integration (MVP)
- Create invoice/checkout for Discovery (7d/500 CFA) and Monthly (30d/2000 CFA).
- Send checkout link to user when Freemium limit reached (or upsell).
- Webhook: POST {APP_BASE_URL}/webhooks/paydunya
  - Verify authenticity (per docs).
  - On PAID: update user subscription state and subscription_ends_at.
  - Idempotent processing (safe for retries).

User State Machine (high level)
- New User → FREEMIUM (counter=0) → Warn at 8th → Lock at 10th → Offer packs
- Payment PAID → DISCOVERY(7d) or PREMIUM(30d) → Unlimited until expiry
- Discovery expiry → Prompt to upgrade to Premium → if declined, polite message + schedule 7-day reminder
- Severe symptoms at any state → append emergency guidance and numbers

AI Response Strategy
- System prompt enforces:
  - Medical disclaimer and safety guidance
  - Language (FR/Wolof/EN) per user preference
  - Use curated KB for supported conditions; avoid diagnosis; provide prevention/next steps
- Include recent chat memory window to maintain context
- Emergency detector augments messages when necessary

Internationalization (i18n)
- All system messages templated in French, Wolof, English:
  - Welcome, disclaimer, language selection
  - Freemium warnings + lock/upsell
  - Payment success/expiry prompts
  - Refusal + reminder scheduling
  - Emergency notices

Testing Plan
- Unit: state transitions, counters, emergency rules, i18n templates
- Integration: Twilio and Paydunya webhooks (signature/verification)
- E2E Path: New user → 10Q lock → payment → unlock → expiry → premium upsell/refusal/reminder

Security & Privacy
- Validate all webhooks (Twilio signatures, Paydunya authenticity)
- Protect secrets (.env, never commit)
- Minimal PII stored, input validation/sanitization
- Data retention policy (documented in README/privacy note)

Deployment (later)
- Deploy FastAPI behind HTTPS
- Apply DB migrations and set env vars
- Configure Twilio and Paydunya to target production URLs
- Monitoring/logging enabled

Roadmap (Implementation Steps)
- [ ] Initialize project with FastAPI skeleton
- [ ] Add config and i18n templates (FR/Wolof/EN)
- [ ] Implement Twilio webhook + language selection
- [ ] Implement user state + freemium counter (+ warn/lock)
- [ ] Build KB and OpenAI integration with disclaimers/guardrails
- [ ] Implement emergency detection (Senegal numbers first)
- [ ] Integrate Paydunya: invoice creation + webhook verification
- [ ] Subscription activation + expirations + state transitions
- [ ] Chat memory persistence and context windowing
- [ ] Reminders (refusal + pre-expiry)
- [ ] Tests (unit/integration/E2E) and logging/admin
- [ ] Deployment and live configuration

Next Steps (what I’ll create next)
- requirements.txt with minimal dependencies
- .env.example with required variables
- app/main.py with FastAPI bootstrap and health check
- routers for /webhooks/twilio and /webhooks/paydunya (stubs)
- i18n templates for welcome/disclaimer and language selection
