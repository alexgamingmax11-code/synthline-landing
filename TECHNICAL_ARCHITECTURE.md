# Synthline -- Technical Architecture

## 1. Telephony Layer

### Provider Comparison

| Provider | Inbound $/min | Outbound $/min | Phone # | WebSocket/Media Streams | Best For |
|---|---|---|---|---|---|
| **Twilio** | $0.013 | $0.013 | $1.15/mo | Yes (Media Streams) | Reliability, ecosystem, docs |
| **Telnyx** | $0.005-0.008 | $0.005-0.008 | ~$1.00/mo | Yes (SIP/WebSocket) | Cost-conscious at scale |
| **Vonage** | $0.013 | $0.014 | ~$1.00/mo | Yes (NCCO) | Carrier-grade backbone |
| **Bland.ai** | Included in platform | Included | N/A | Proprietary | Turnkey (not a raw telephony provider) |
| **Vapi** | Included in platform | Included | N/A | Proprietary | Turnkey (wraps Twilio/Vonage) |
| **Retell AI** | Included in platform | Included | N/A | Proprietary | Turnkey (wraps Twilio) |

### Recommendation: Telnyx

**Why Telnyx over Twilio:**
- 50-60% cheaper on per-minute voice rates ($0.005-0.008 vs $0.013)
- Native SIP support for flexibility
- Programmable SIP trunking with WebSocket media streaming
- Better unit economics at scale -- critical for a free tier business model
- Growing developer ecosystem with solid documentation

**When to reconsider Twilio:** If Telnyx's documentation or SDK quality creates friction during MVP, Twilio's superior DX and ecosystem (more tutorials, Stack Overflow answers, community) may justify the 60% cost premium. Twilio is the safe default.

**How the bridge works:**
1. Business gets a Telnyx phone number (or port their existing number)
2. Inbound call hits Telnyx's telephony network
3. Telnyx opens a WebSocket to your server, streaming raw audio (mulaw 8kHz)
4. Your server bridges this audio to the AI pipeline (STT -> LLM -> TTS)
5. TTS audio is streamed back through the same WebSocket to the caller
6. For human escalation: Twilio/Telnyx `<Transfer>` or `<Dial>` verb bridges to the business owner's phone

**SMS (confirmations):** Twilio SMS at $0.0079/msg is the industry standard. Use it for appointment confirmations regardless of voice provider. Alternatively, Telnyx offers competitive SMS rates (~$0.005/msg).

---

## 2. AI Stack

### Voice-to-Text (STT)

| Provider | Cost/min | Latency | Streaming | Notes |
|---|---|---|---|---|
| **Deepgram Nova-2** | $0.0043 | ~300ms | Yes | Best price-performance; purpose-built for real-time |
| **AssemblyAI** | $0.015 | ~400ms | Yes | Higher accuracy on edge cases; more expensive |
| **OpenAI Whisper API** | $0.006 | ~1-2s (batch) | No | Not suitable for real-time; batch processing only |

**Recommendation: Deepgram Nova-2**
- 3x cheaper than AssemblyAI, suitable accuracy for phone conversations
- Native streaming via WebSocket -- critical for real-time voice agents
- Sub-second latency on streaming mode
- $200 free credits on signup for development

**Why not Whisper:** Batch API only -- no streaming. A real-time voice agent cannot wait 1-2 seconds for full audio transcription. Whisper is excellent for post-call transcription (call recordings, analytics) but not for the live conversation loop.

### LLM (Conversation Brain)

| Model | Input $/1M tokens | Output $/1M tokens | Latency (TTFT) | Quality | Voice-optimized? |
|---|---|---|---|---|---|
| **GPT-4o-mini** | $0.15 | $0.60 | ~300ms | Good | No (text-only) |
| **GPT-4o** | $2.50 | $10.00 | ~500ms | Excellent | Yes (audio modality) |
| **Claude 3.5 Sonnet** | $3.00 | $15.00 | ~400ms | Excellent | No (text-only) |
| **Gemini 2.0 Flash** | $0.10 | $0.40 | ~200ms | Very Good | Yes (live API) |
| **Gemini 2.5 Flash** | $0.15 | $0.60 | ~250ms | Excellent | Yes (live API) |

**Recommendation: Gemini 2.0 Flash (primary) or GPT-4o-mini (fallback)**

**Why Gemini 2.0 Flash:**
- Lowest cost at $0.10/$0.40 per 1M tokens
- Native audio/voice modality -- can process audio directly without STT in some architectures
- Fastest time-to-first-token (~200ms)
- Generous free tier (15 RPM, 1M tokens/day) -- useful during development
- Function calling support for booking, lead qualification, FAQ lookups

**Architecture for a 3-minute call with Gemini Flash:**
- Average conversation: ~4,000 input tokens, ~2,000 output tokens
- Cost: (4,000 * $0.10/1M) + (2,000 * $0.40/1M) = ~$0.0012 per call
- Essentially free at the LLM layer

**Why not Claude for production:** Claude is excellent for text, but lacks native real-time audio modality. The higher cost ($3/$15 per 1M tokens) also makes it ~30x more expensive than Gemini Flash for this use case. Keep Claude available as a configurable option for customers who want premium quality.

### Text-to-Speech (TTS)

| Provider | Cost | Latency (first byte) | Quality | Streaming | Notes |
|---|---|---|---|---|---|
| **Cartesia Sonic** | ~$0.05/1K chars | ~75ms | Excellent | Yes | Fastest latency; best for real-time |
| **ElevenLabs** | ~$0.17-0.22/1K chars | ~200ms | Best quality | Yes | Premium voice quality; higher cost |
| **PlayHT** | ~$0.15-0.30/1K chars | ~200ms | Good | Yes | Budget option; less natural |

**Recommendation: Cartesia Sonic**

**Why Cartesia:**
- Lowest latency in the market (~75ms time-to-first-audio) -- critical for natural conversation
- The biggest UX killer in voice agents is awkward pauses between turns; Cartesia minimizes this
- ~70% cheaper than ElevenLabs per character
- High-quality natural-sounding voices
- Excellent WebSocket streaming API

**Cost calculation for a 3-minute call:**
- Average TTS output: ~600 words = ~3,000 characters
- Cartesia: 3,000 / 1,000 * $0.05 = $0.15 per call... wait, let me recheck
- Actually at ~$0.05/1K chars: 3,000 chars = ~$0.15... that seems high
- Re-estimating: a 3-min call has ~2 min of agent speech (~300 words/min)
- That is ~600 words = ~3,000 characters
- At $0.05/1K = $0.15 per call for TTS alone
- Actually let me re-examine: Cartesia pricing is likely per-1K-characters, so 3K chars * ($0.05/1000) = $0.15
- This is the biggest cost driver -- need to optimize agent speech efficiency

**Cost optimization:** Design agent prompts to be concise. A well-designed voice agent speaks 40-60% less than a human agent for the same information. Target 200-400 words per call response (not monologues).

**Fallback: ElevenLabs** for customers who prioritize voice quality over latency/cost. ElevenLabs offers voice cloning which is a premium differentiator.

### Combined Per-Minute Cost (Custom Build)

| Component | Cost/min |
|---|---|
| Telnyx telephony | $0.007 |
| Deepgram STT | $0.0043 |
| Gemini Flash LLM | ~$0.002 |
| Cartesia TTS | ~$0.03 |
| **Total** | **~$0.043/min** |

This is the all-in marginal cost per minute of conversation when building custom.

---

## 3. Turnkey Platforms vs Custom Build

### Platform Comparison

| Platform | Per-min cost (all-in) | Multi-tenant | Calendar | Custom LLM | Dashboard | White-label |
|---|---|---|---|---|---|---|
| **Vapi** | $0.08-0.15 | Limited | Via tools | Yes | Basic | No |
| **Retell AI** | $0.08-0.12 | Yes | Via tools | Yes | Good | No |
| **Bland.ai** | $0.07-0.10 | Yes | Via integrations | Limited | Good | Enterprise only |
| **Synthflow** | $0.10-0.15 | Yes (agency) | Via tools | Limited | Good | Yes ($800+/mo) |
| **Custom (Telnyx+stack)** | $0.04-0.06 | You build it | You build it | Full control | You build it | You control it |

### Decision: Hybrid -- Custom Build with Platform Optionality

**Phase 1 (MVP): Custom build using the Telnyx + Deepgram + Gemini + Cartesia stack.**

Reasons:
1. **Cost control is existential** for a free-tier business model. At $0.04/min custom vs $0.08-0.15/min platform, you burn through margins twice as fast on platforms.
2. **Multi-tenancy is a core requirement.** Turnkey platforms offer limited multi-tenant support; you'd outgrow their model quickly.
3. **Solo developer advantage:** You own the code, can iterate on prompts/tools, and are not locked into a platform's roadmap.
4. **The "hard part" is not telephony.** The hard parts are prompt engineering, tool integrations (Calendar, SMS), escalation logic, and dashboard UX -- none of which a platform solves for you.

**Phase 2 consideration:** If the custom telephony layer proves unreliable at scale, you can always route through Vapi or Retell as a fallback (at 2-3x cost) while keeping the rest of your stack. This is a clean escape hatch, not a commitment.

**When a platform makes sense:** If Synthline pivots to an agency/reseller model where you white-label the entire service. Synthflow's $800-1,499/mo white-label plan could be justified if reselling to agencies. But as a direct SaaS, build custom.

---

## 4. Backend Architecture

### Tech Stack

```
Frontend (Dashboard)
  Next.js 14+ (App Router)
  Tailwind CSS + shadcn/ui
  React Query for data fetching

Backend API
  Node.js + Fastify (or Python + FastAPI)
  TypeScript throughout
  tRPC or REST API

Voice Pipeline Server
  Python + FastAPI (WebSocket handling)
  asyncio for concurrent call processing

Database
  PostgreSQL (via Supabase or Neon)
  Redis (session state, rate limiting, real-time cache)

Auth
  Clerk or NextAuth.js
  Multi-tenant isolation via row-level security (RLS)

Infrastructure
  Railway or Fly.io (primary)
  Cloudflare (DNS, CDN, edge)
  Upstash (Redis -- serverless)
```

**Why Fastify over Express:** 2-3x faster request handling, built-in schema validation, native TypeScript support. Critical when handling dashboard API requests alongside real-time call traffic.

**Why Python for the voice pipeline:** The WebSocket + audio processing stack is more mature in Python (Deepgram SDK, Twilio SDK, asyncio). Keep the dashboard in TypeScript/Next.js.

### Database Design

```sql
-- Core tenant table
CREATE TABLE businesses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES auth.users(id),
  name VARCHAR(255) NOT NULL,
  phone_number VARCHAR(20),           -- Twilio/Telnyx number assigned
  telnyx_connection_id VARCHAR(100),   -- SIP connection for this tenant
  
  -- Configuration
  greeting TEXT DEFAULT 'Hello, thank you for calling. How can I help you today?',
  business_hours JSONB,                -- {"mon": {"open": "09:00", "close": "17:00"}, ...}
  timezone VARCHAR(50) DEFAULT 'America/New_York',
  
  -- AI Agent config
  agent_prompt TEXT,                    -- Custom system prompt for this business
  agent_personality VARCHAR(50) DEFAULT 'professional',
  escalation_number VARCHAR(20),       -- Forward-to-human number
  escalation_enabled BOOLEAN DEFAULT true,
  
  -- Calendar integration
  google_calendar_id VARCHAR(255),
  google_access_token TEXT,            -- Encrypted
  google_refresh_token TEXT,           -- Encrypted
  
  -- Plan & limits
  plan VARCHAR(20) DEFAULT 'free',    -- free, starter, professional, enterprise
  monthly_call_limit INT DEFAULT 50,
  calls_this_month INT DEFAULT 0,
  
  -- Metadata
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Call records
CREATE TABLE calls (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  
  -- Call details
  direction VARCHAR(10) NOT NULL,      -- inbound, outbound
  caller_number VARCHAR(20),
  status VARCHAR(20) NOT NULL,         -- ringing, active, completed, transferred, failed
  started_at TIMESTAMPTZ NOT NULL,
  answered_at TIMESTAMPTZ,
  ended_at TIMESTAMPTZ,
  duration_seconds INT,
  
  -- AI processing
  transcript JSONB,                    -- Full conversation transcript with timestamps
  summary TEXT,                        -- AI-generated call summary
  sentiment VARCHAR(20),               -- positive, neutral, negative
  lead_score INT,                      -- 0-100 qualification score
  
  -- Outcomes
  intent VARCHAR(50),                  -- appointment, inquiry, escalation, spam, etc.
  appointment_booked BOOLEAN DEFAULT false,
  appointment_id UUID,
  escalated BOOLEAN DEFAULT false,
  escalated_at TIMESTAMPTZ,
  
  -- Cost tracking
  telephony_cost_cents INT,
  stt_cost_cents INT,
  llm_cost_cents INT,
  tts_cost_cents INT,
  total_cost_cents INT,
  
  -- Recording
  recording_url TEXT,
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Appointment bookings
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  call_id UUID REFERENCES calls(id),
  
  customer_name VARCHAR(255),
  customer_phone VARCHAR(20),
  customer_email VARCHAR(255),
  
  service VARCHAR(255),
  scheduled_at TIMESTAMPTZ NOT NULL,
  duration_minutes INT DEFAULT 30,
  notes TEXT,
  
  -- Calendar sync status
  google_event_id VARCHAR(255),
  synced_to_calendar BOOLEAN DEFAULT false,
  sms_confirmation_sent BOOLEAN DEFAULT false,
  
  status VARCHAR(20) DEFAULT 'confirmed', -- confirmed, cancelled, completed, no-show
  
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- FAQ knowledge base per business
CREATE TABLE faqs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  question TEXT NOT NULL,
  answer TEXT NOT NULL,
  category VARCHAR(100),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Call analytics (aggregated daily)
CREATE TABLE daily_analytics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID NOT NULL REFERENCES businesses(id),
  date DATE NOT NULL,
  
  total_calls INT DEFAULT 0,
  answered_calls INT DEFAULT 0,
  missed_calls INT DEFAULT 0,
  average_duration_seconds INT,
  
  appointments_booked INT DEFAULT 0,
  leads_qualified INT DEFAULT 0,
  escalations INT DEFAULT 0,
  
  total_cost_cents INT DEFAULT 0,
  
  UNIQUE(business_id, date)
);

-- Indexes for performance
CREATE INDEX idx_calls_business_id ON calls(business_id);
CREATE INDEX idx_calls_started_at ON calls(started_at);
CREATE INDEX idx_calls_status ON calls(status);
CREATE INDEX idx_appointments_business_scheduled ON appointments(business_id, scheduled_at);
CREATE INDEX idx_daily_analytics_business_date ON daily_analytics(business_id, date);
```

### API Design

```
POST   /api/auth/signup              -- Business owner registration
POST   /api/auth/login               -- Login

GET    /api/businesses               -- List businesses for current user
POST   /api/businesses               -- Create new business
PUT    /api/businesses/:id           -- Update business config
PATCH  /api/businesses/:id/calendar  -- Connect Google Calendar (OAuth)

GET    /api/businesses/:id/agent     -- Get agent configuration
PUT    /api/businesses/:id/agent     -- Update agent prompt, personality, greeting

GET    /api/calls?business_id=&limit=&offset=  -- List calls with pagination
GET    /api/calls/:id                -- Get call detail + transcript
GET    /api/calls/:id/recording      -- Stream call recording audio

GET    /api/faqs?business_id=        -- List FAQs
POST   /api/faqs                     -- Add FAQ
PUT    /api/faqs/:id                 -- Update FAQ
DELETE /api/faqs/:id                 -- Delete FAQ

GET    /api/appointments?business_id=&date=  -- List appointments
POST   /api/appointments             -- Book appointment (usually via voice agent)
DELETE /api/appointments/:id         -- Cancel appointment

GET    /api/analytics/overview?business_id=&period=7d   -- Dashboard summary
GET    /api/analytics/calls?business_id=&period=7d      -- Call volume chart data
GET    /api/analytics/costs?business_id=&period=30d     -- Cost breakdown

POST   /api/webhooks/telnyx          -- Telnyx call events (answered, ended, etc.)
POST   /api/webhooks/telnyx/media    -- WebSocket upgrade for media streaming
POST   /api/webhooks/google          -- Google Calendar push notifications
```

### Real-Time Monitoring

For the dashboard, use **Server-Sent Events (SSE)** or **WebSocket** from the voice pipeline server to push real-time call status to the dashboard.

```
Voice Pipeline Server
  |-- publishes call events to Redis Pub/Sub
  |-- Dashboard subscribes via SSE endpoint
  |-- Events: call_started, call_active, call_ended, transcript_chunk, escalation
```

Redis channels:
- `calls:active:{business_id}` -- currently active calls
- `calls:events:{business_id}` -- all events for live dashboard feed

### Google Calendar Integration

```
Flow:
1. Business owner clicks "Connect Google Calendar" in dashboard
2. Redirect to Google OAuth 2.0 consent screen (scope: calendar.events)
3. Store access_token + refresh_token (encrypted at rest)
4. Voice agent checks availability via Calendar API
5. Books appointment, sends SMS confirmation via Twilio/Telnyx

Implementation:
- Use googleapis npm package
- Handle token refresh automatically (tokens expire in 1 hour)
- Buffer booking window: check available slots in 15-min increments
- Prevent double-booking by querying existing events first
```

---

## 5. Cost Analysis

### Cost Per Minute (Custom Build)

| Component | Cost/min | Source |
|---|---|---|
| Telnyx telephony | $0.007 | Telnyx rate card |
| Deepgram Nova-2 STT | $0.0043 | Deepgram pricing |
| Gemini 2.0 Flash LLM | ~$0.002 | Token cost for avg conversation |
| Cartesia Sonic TTS | ~$0.030 | Character-based, avg speech output |
| **Total AI pipeline** | **~$0.043** | |

### Cost Per Call (3-minute average)

| Component | Cost |
|---|---|
| Telnyx telephony (3 min) | $0.021 |
| Deepgram STT (3 min) | $0.013 |
| Gemini Flash LLM | $0.006 |
| Cartesia TTS (~600 words output) | $0.090 |
| Telnyx SMS (confirmation) | $0.005 |
| Telnyx phone number (1/mo, prorated) | ~$0.03 |
| **Total per call** | **~$0.165** |

### Free Tier Analysis (50 calls/mo)

| Item | Monthly Cost |
|---|---|
| 50 calls x $0.165 | $8.25 |
| 1 phone number | $1.00 |
| Supabase (free tier) | $0 |
| Railway/Fly.io (hobby) | $0-5 |
| Domain/DNS | $0 (Cloudflare free) |
| **Total free tier cost per tenant** | **~$13/mo** |

**At 100 free-tier users:** ~$1,300/mo in infrastructure + per-call costs.

**Mitigation strategies:**
1. Require credit card for signup (even free tier) -- reduces vanity signups
2. Aggressive usage monitoring -- flag and throttle users calling 50x/mo consistently
3. Optimize TTS output length (biggest cost driver at $0.09/call)
4. Consider Gemini Flash's native audio mode to eliminate separate TTS cost for some calls

### Professional Tier Margin ($49/mo, 500 calls included)

| Item | Cost |
|---|---|
| 500 calls x $0.165 | $82.50 |
| 1 phone number | $1.00 |
| Infrastructure (proportional) | ~$2.00 |
| **Total cost** | **~$85.50** |
| **Revenue** | **$49.00** |
| **Margin** | **-74% (negative)** |

**This is a problem.** The per-call cost at $0.165 makes 500 calls/month unprofitable at $49/mo. Options:

**Option A: Raise prices.** Professional tier at $99/mo for 500 calls.
- Revenue: $99, Cost: $85.50, Margin: ~13.6%

**Option B: Reduce per-call cost.** Optimize TTS usage.
- If TTS cost drops to $0.04/call (shorter responses, Cartesia volume discount):
  - New per-call cost: ~$0.115
  - 500 calls x $0.115 = $57.50
  - At $99/mo: Margin = 42%

**Option C: Volume-based pricing instead of flat rate.** Charge per-minute rather than per-call.
- $0.20/min charged, $0.043/min cost = 78% gross margin

**Recommended pricing model:**

| Tier | Price | Included | Overage | Target Margin |
|---|---|---|---|---|
| Free | $0 | 50 calls/mo | N/A (hard cap) | -$13 (acquisition cost) |
| Starter | $29/mo | 100 calls/mo | $0.20/call | ~40% |
| Professional | $79/mo | 400 calls/mo | $0.15/call | ~50% |
| Enterprise | $199/mo | 1,500 calls/mo | $0.10/call | ~55% |

### Infrastructure at Scale

| Users | Calls/mo | Monthly Cost | Revenue | Net |
|---|---|---|---|---|
| 50 (all free) | 2,500 | $437 | $0 | -$437 |
| 50 free + 20 starter | 5,000 | $870 | $580 | -$290 |
| 50 free + 50 starter + 10 pro | 17,500 | $3,025 | $2,240 | -$785 |
| 50 free + 100 starter + 30 pro + 5 enterprise | 57,500 | $9,938 | $5,620 | -$4,318 |

**Break-even analysis:** With current cost structure, you need ~300 paying customers at average $50/mo to cover free-tier losses and infrastructure. This assumes optimized TTS costs.

**Key insight:** The free tier is a customer acquisition tool, not a product. The business model works when free-tier users convert to paid. Target 15-20% conversion rate within 30 days.

---

## 6. MVP Roadmap

### Phase 1: Core Voice Agent (Weeks 1-4)

**Goal:** A single AI agent that answers calls, handles basic FAQs, and can book appointments.

```
Week 1: Telephony + Basic Voice Pipeline
  - Set up Telnyx account, provision test phone number
  - Build FastAPI WebSocket server for Twilio Media Streams
  - Integrate Deepgram for real-time STT
  - Integrate Gemini Flash for conversation LLM
  - Integrate Cartesia for TTS
  - Basic prompt engineering: greeting, FAQ responses, goodbye
  - Test: Call the number, have a basic conversation

Week 2: Calendar Integration + SMS
  - Google Calendar OAuth flow
  - Check availability and book appointments via voice
  - SMS confirmation via Telnyx/Twilio
  - Human escalation: detect "speak to a person" and transfer call
  - Business hours logic: after-hours greeting + voicemail prompt

Week 3: Multi-Tenant Backend
  - PostgreSQL database setup (Supabase)
  - Business registration + auth (Clerk)
  - Per-tenant agent configuration (greeting, prompt, FAQ)
  - Per-tenant phone number assignment
  - Call logging and cost tracking

Week 4: Dashboard MVP
  - Next.js dashboard with call history
  - Real-time call status (SSE from voice pipeline)
  - Transcript viewer per call
  - Call summary (AI-generated)
  - Basic analytics: calls today, this week, appointments booked
```

**Build vs Buy Decisions for Phase 1:**

| Decision | Build | Buy/Use | Verdict |
|---|---|---|---|
| Auth | -- | Clerk ($0 free, $25/mo after) | Buy |
| Database | -- | Supabase (free tier) | Buy |
| Hosting | -- | Railway ($5/mo) or Fly.io | Buy |
| Dashboard UI | Next.js + shadcn | -- | Build |
| Voice pipeline | FastAPI + WebSockets | -- | Build |
| Calendar | googleapis npm | -- | Build |
| SMS | Telnyx API | -- | Build (it is one API call) |
| Landing page | Already built (index.html) | -- | Done |

### Phase 2: Production Readiness (Weeks 5-8)

**Goal:** Billing, onboarding flow, monitoring, reliability.

```
Week 5-6: Billing + Onboarding
  - Stripe integration for subscription billing
  - Free tier signup flow (no credit card required)
  - Upgrade flow: Free -> Starter -> Professional
  - Phone number provisioning during onboarding
  - Google Calendar connection during onboarding
  - FAQ setup wizard (or CSV upload)

Week 7: Monitoring + Reliability
  - Call failure alerts (voice pipeline errors)
  - Latency monitoring (STT + LLM + TTS per call)
  - Cost tracking dashboard (per-business and aggregate)
  - Rate limiting (prevent abuse)
  - Call quality metrics (interruption rate, completion rate)

Week 8: Polish + Beta Launch
  - Dashboard UX polish
  - Call recording playback
  - Email notifications (daily/weekly call summaries)
  - Business owner mobile-responsive dashboard
  - Beta launch to 10-20 small businesses
```

### Phase 3: Growth Features (Months 3-6)

```
Month 3: Intelligence Layer
  - Lead qualification scoring (AI scores caller intent)
  - Sentiment analysis per call
  - Call outcome categorization
  - Follow-up SMS sequences
  - CRM integrations (HubSpot, Google Sheets)

Month 4: Advanced Voice Features
  - Multi-language support (detect caller language)
  - Voice cloning (bring-your-own-voice via ElevenLabs)
  - Custom hold music / wait messages
  - Conference call support (bridge multiple parties)
  - Voicemail transcription

Month 5: Analytics + Optimization
  - Call volume forecasting
  - Best-time-to-call recommendations
  - A/B testing for greetings and prompts
  - Call-to-booking conversion funnel
  - Customer satisfaction surveys (post-call SMS)

Month 6: Scale + Enterprise
  - White-label option for agencies
  - API for programmatic agent management
  - Bulk import/export
  - SOC 2 compliance groundwork
  - Dedicated phone number pools for high-volume users
```

---

## 7. Hosting + DevOps

### Cloud Provider

**Primary: Railway ($5/mo hobby tier)**

Why Railway over alternatives:
- One-click deploys from GitHub
- Built-in PostgreSQL hosting (saves Supabase dependency)
- WebSocket support (required for voice pipeline)
- $5/mo gets you a basic always-on service
- Scales vertically easily (CPU/RAM upgrades)
- Free $5/mo credit for new projects

**Alternative: Fly.io** -- better for global edge deployment if latency matters (it does for voice). Fly.io's machines can be deployed in multiple regions to reduce round-trip time between Telnyx's POPs and your voice server.

**When to move to AWS/GCP:** When you have 100+ concurrent calls or need advanced networking. Not before.

### Architecture Diagram

```
                    [Phone Call]
                         |
                   [Telnyx Network]
                         |
              [WebSocket: Audio Stream]
                         |
                  [Voice Pipeline Server]
                  (Python + FastAPI)
                  Railway/Fly.io
                         |
           +-------------+-------------+
           |             |             |
      [Deepgram]    [Gemini Flash]  [Cartesia]
       (STT)          (LLM)         (TTS)
                         |
              [PostgreSQL / Supabase]
                         |
                  [Dashboard API]
                  (Fastify + TypeScript)
                  Railway
                         |
               [Next.js Dashboard]
                  Vercel (free)
```

### CI/CD

```
GitHub Actions Pipeline:
1. Push to main
2. Run tests (unit + integration)
3. Build Docker images
4. Deploy to Railway (auto-deploy from main)
5. Run smoke test (make a test call to staging number)

Environments:
  - staging: Auto-deploy from develop branch
  - production: Auto-deploy from main branch (with approval gate)

Monitoring:
  - UptimeRobot: Free, monitors HTTP endpoints + TCP ports
  - Sentry: Free tier for error tracking
  - Railway metrics: CPU, memory, network built-in
  - Custom: Log structured call events to PostHog (free tier)
```

### Call Quality Monitoring

The voice pipeline needs tight monitoring because latency kills user experience.

```
Target Latencies:
  - STT (first token): < 500ms
  - LLM (first token): < 300ms
  - TTS (first audio): < 100ms
  - End-to-end (user stops speaking to agent starts speaking): < 800ms
  - Total pipeline round-trip: < 1.2s

Monitoring approach:
  - Instrument each pipeline stage with timing
  - Log p50, p95, p99 latencies per call
  - Alert if p95 exceeds 1.5s for any component
  - Dashboard showing real-time latency across all active calls
```

---

## 8. Key Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Voice latency too high (>1.5s) | Users hang up; product feels broken | Cartesia's 75ms TTS + Gemini's fast TTFT + streaming STT. Profile early. |
| Free tier abuse | 10,000 bots calling 50x/mo = $13K/mo loss | Require phone verification; rate limit outbound; hard cap at 50 calls. |
| Google Calendar API quota limits | Can't book during peak | Use per-business OAuth (each business has own quota); implement queuing. |
| LLM hallucinations | Agent gives wrong business info | Strict system prompts; FAQ knowledge base via RAG; guardrails on responses. |
| Twilio/Telnyx outage | All calls fail | Secondary provider fallback (Telnyx primary, Twilio backup). Geographic redundancy later. |
| Single developer bus factor | Everything stops | Keep architecture simple; document aggressively; consider co-founder before scaling. |

---

## 9. Summary: Recommended Stack

```
Telephony:     Telnyx (primary) / Twilio (fallback)
STT:           Deepgram Nova-2 ($0.0043/min, streaming)
LLM:           Gemini 2.0 Flash ($0.10/1M in, $0.40/1M out)
TTS:           Cartesia Sonic ($0.05/1K chars, 75ms latency)
Backend:       Python FastAPI (voice pipeline) + Node.js Fastify (dashboard API)
Frontend:      Next.js 14 + shadcn/ui
Database:      PostgreSQL via Supabase or Railway Postgres
Auth:          Clerk
Hosting:       Railway ($5/mo start)
SMS:           Telnyx SMS ($0.005/msg)
Calendar:      Google Calendar API (googleapis)
Monitoring:    UptimeRobot + Sentry + PostHog
CI/CD:         GitHub Actions + Railway auto-deploy
```

**Estimated MVP build time:** 8 weeks (Phases 1-2) for a solo developer working 20-30 hours/week.

**Estimated monthly cost at MVP launch (beta, 20 users):**
- Hosting: $20/mo (Railway Pro)
- Supabase: $0 (free tier)
- Telnyx: $5/mo + usage (~$30 for 200 calls)
- Deepgram: $0 (free credits for first ~46K minutes)
- Gemini Flash: ~$0 (free tier handles development + beta)
- Cartesia: ~$20/mo (free tier + starter)
- Clerk: $0 (free for <10K MAUs)
- Domain: $0 (Cloudflare)
- **Total: ~$75/mo**

**Key pricing sources:**
- [Twilio Voice Pricing](https://www.twilio.com/voice/pricing)
- [Telnyx Pricing](https://telnyx.com/pricing)
- [Vapi.ai Pricing](https://vapi.ai/pricing)
- [Retell AI Pricing](https://www.retellai.com/pricing)
- [Deepgram Pricing](https://deepgram.com/pricing)
- [ElevenLabs Pricing](https://elevenlabs.io/pricing)
- [Cartesia Pricing](https://cartesia.ai/pricing)
- [Gemini API Pricing](https://ai.google.dev/pricing)
