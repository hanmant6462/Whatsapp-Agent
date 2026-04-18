# WhatsApp AI Agent — Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER'S WHATSAPP                              │
│                   (sends/receives messages)                         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  HTTPS (Meta Graph API)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     META BUSINESS PLATFORM                          │
│              WhatsApp Business API / Webhook Relay                  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  POST /api/webhook
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      NEXT.JS APP (Port 3000)                        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    API ROUTES                                │   │
│  │                                                              │   │
│  │  GET  /api/webhook          ── Webhook verification          │   │
│  │  POST /api/webhook          ── Receive & process messages    │   │
│  │  GET  /api/conversations    ── List all conversations        │   │
│  │  PATCH /api/conversations/[id]  ── Toggle agent/human mode  │   │
│  │  GET  /api/conversations/[id]/messages  ── Fetch messages    │   │
│  │  POST /api/conversations/[id]/send  ── Manual send           │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────┐   ┌──────────────┐   ┌────────────────────┐  │
│  │   src/lib/ai.ts  │   │  whatsapp.ts │   │   supabase.ts      │  │
│  │                  │   │              │   │                    │  │
│  │  OpenRouter API  │   │  Meta Graph  │   │  Supabase Client   │  │
│  │  (AI responses)  │   │  API (send)  │   │  (DB + Realtime)   │  │
│  └────────┬─────────┘   └──────┬───────┘   └─────────┬──────────┘  │
│           │                    │                     │             │
└───────────┼────────────────────┼─────────────────────┼─────────────┘
            │                    │                     │
            ▼                    ▼                     ▼
┌───────────────────┐  ┌──────────────────┐  ┌────────────────────────┐
│   OPENROUTER      │  │  META GRAPH API  │  │       SUPABASE         │
│                   │  │                  │  │                        │
│  Model:           │  │  Sends WhatsApp  │  │  PostgreSQL DB         │
│  liquid/lfm-2.5   │  │  reply back to   │  │                        │
│  (or any model)   │  │  the user        │  │  ┌──────────────────┐  │
└───────────────────┘  └──────────────────┘  │  │  conversations   │  │
                                             │  │  ─────────────── │  │
                                             │  │  id (uuid) PK    │  │
                                             │  │  phone (unique)  │  │
                                             │  │  name            │  │
                                             │  │  mode agent|human│  │
                                             │  │  updated_at      │  │
                                             │  │  created_at      │  │
                                             │  └────────┬─────────┘  │
                                             │           │ 1:many     │
                                             │  ┌────────▼─────────┐  │
                                             │  │     messages     │  │
                                             │  │  ─────────────── │  │
                                             │  │  id (uuid) PK    │  │
                                             │  │  conversation_id │  │
                                             │  │  role user|asst  │  │
                                             │  │  content         │  │
                                             │  │  whatsapp_msg_id │  │
                                             │  │  created_at      │  │
                                             │  └──────────────────┘  │
                                             │                        │
                                             │  Realtime Subscription │
                                             │  (pushes to dashboard) │
                                             └────────────────────────┘
                                                        │
                                                        │ Supabase Realtime
                                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     DASHBOARD (Browser)                             │
│                      src/app/page.tsx                               │
│                                                                     │
│  ┌─────────────────────────┐   ┌───────────────────────────────┐   │
│  │       SIDEBAR           │   │         CHAT PANEL            │   │
│  │                         │   │                               │   │
│  │  • Conversation list    │   │  • WhatsApp-style bubbles     │   │
│  │  • Sorted by latest     │   │  • User msgs  (left, grey)    │   │
│  │  • AI / Human badge     │   │  • AI msgs    (right, green)  │   │
│  │  • Real-time updates    │   │  • Timestamps                 │   │
│  │                         │   │  • Mode toggle (AI ↔ Human)   │   │
│  │                         │   │  • Manual send input bar      │   │
│  └─────────────────────────┘   └───────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘


MESSAGE FLOW (Incoming)
═══════════════════════
User types on WhatsApp
  → Meta forwards to POST /api/webhook
  → Conversation found or created in DB
  → User message stored in messages table
  → If mode = "human"  →  stored, dashboard shows it, human replies manually
  → If mode = "agent"  →  fetch last 20 msgs as context
                         → call OpenRouter AI
                         → AI reply sent via Meta Graph API to user's WhatsApp
                         → AI reply stored in messages table
                         → Supabase Realtime notifies dashboard instantly

MESSAGE FLOW (Manual / Human mode)
═══════════════════════════════════
Agent clicks send in dashboard
  → POST /api/conversations/[id]/send
  → Message sent via Meta Graph API to user's WhatsApp
  → Message stored in messages table
  → Dashboard updates via Supabase Realtime


ENVIRONMENT VARIABLES
═════════════════════
WHATSAPP_ACCESS_TOKEN       Meta permanent system user token
WHATSAPP_PHONE_NUMBER_ID    Meta phone number ID
WHATSAPP_VERIFY_TOKEN       Shared secret for webhook verification
OPENROUTER_API_KEY          OpenRouter API key
AI_MODEL                    Model ID (e.g. liquid/lfm-2.5-1.2b-instruct:free)
NEXT_PUBLIC_SUPABASE_URL    Supabase project URL
NEXT_PUBLIC_SUPABASE_ANON_KEY  Supabase anon key (frontend)
SUPABASE_SERVICE_ROLE_KEY   Supabase service role key (backend)
```
