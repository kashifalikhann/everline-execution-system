# GOVREADY — GSTACK SYSTEM DESIGN
## Complete Architecture for All Layers
## June 27, 2026

---

## THE 5-LAYER ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│ LAYER 5: CLIENT INTERFACE                                       │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│ │ Landing Page │ │  Dashboard   │ │   Mobile App (Phase 3)   │ │
│ │ (Acquisition)│ │  (Operations)│ │   (On-the-go alerts)     │ │
│ └──────────────┘ └──────────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 4: SERVICE DELIVERY                                       │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│ │ Registration │ │ Compliance   │ │   Bid Intelligence       │ │
│ │ Engine       │ │ Tracker      │ │   (Price + Win Score)    │ │
│ └──────────────┘ └──────────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 3: AI + AUTOMATION                                        │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│ │ Form Filler  │ │ Bid Monitor  │ │   Vector Embedding       │ │
│ │ (Claude API) │ │ (n8n + R2)   │ │   (Intent Matching)      │ │
│ └──────────────┘ └──────────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 2: DATA (THE MOAT)                                        │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│ │ Entity DB    │ │ Contractor   │ │   Historical Price       │ │
│ │ (90K+ units) │ │ Profiles     │ │   Intelligence           │ │
│ └──────────────┘ └──────────────┘ └──────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│ LAYER 1: INFRASTRUCTURE                                         │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│ │ Supabase     │ │ Vercel       │ │   Stripe + Resend        │ │
│ │ (Postgres)   │ │ (Hosting)    │ │   (Payments + Email)     │ │
│ └──────────────┘ └──────────────┘ └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## LAYER 1: INFRASTRUCTURE

### Supabase (PostgreSQL)
**Purpose:** Primary database, authentication, file storage, real-time subscriptions

**Key Tables:**
```sql
-- Government entities (THE MOAT)
CREATE TABLE entities (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  type TEXT NOT NULL, -- 'federal', 'state', 'county', 'municipality', 'school', 'special'
  state TEXT NOT NULL,
  county TEXT,
  fips_code TEXT,
  website TEXT,
  phone TEXT,
  address TEXT,
  registration_url TEXT,
  registration_type TEXT, -- 'portal', 'email', 'mail', 'cooperative', 'in_person'
  bid_portal_url TEXT,
  bid_portal_type TEXT, -- 'bidnet', 'escnj', 'njstart', 'direct', 'govwin'
  contact_1_name TEXT,
  contact_1_title TEXT,
  contact_1_email TEXT,
  contact_1_phone TEXT,
  contact_2_name TEXT,
  contact_2_title TEXT,
  contact_2_email TEXT,
  contact_2_phone TEXT,
  buying_categories TEXT[], -- ['paving', 'roofing', 'hvac', etc.]
  annual_budget INTEGER, -- in cents, where available
  payment_terms TEXT, -- 'net_30', 'net_60', 'net_90'
  cooperative_memberships TEXT[], -- ['MCCPC', 'ESCNJ', 'Sourcewell']
  prequal_requirements JSONB, -- state-specific pre-qual requirements
  last_verified DATE,
  verification_source TEXT,
  data_quality_score INTEGER DEFAULT 0, -- 0-100
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Contractors (customers)
CREATE TABLE contractors (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  company_name TEXT NOT NULL,
  ein TEXT UNIQUE,
  duns TEXT,
  naics_codes TEXT[], -- ['237310', '238160']
  vertical TEXT NOT NULL, -- 'paving', 'roofing', 'hvac'
  contact_name TEXT NOT NULL,
  contact_email TEXT NOT NULL,
  contact_phone TEXT,
  address TEXT,
  city TEXT,
  state TEXT,
  zip TEXT,
  bonding_capacity INTEGER, -- in cents
  emr_rating TEXT,
  year_established INTEGER,
  employee_count INTEGER,
  annual_revenue INTEGER, -- in cents
  stripe_customer_id TEXT,
  subscription_tier TEXT, -- 'starter', 'growth', 'pro'
  status TEXT DEFAULT 'trial', -- 'trial', 'active', 'paused', 'churned'
  onboarding_completed BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Documents (uploaded once, used everywhere)
CREATE TABLE documents (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  contractor_id UUID REFERENCES contractors(id),
  type TEXT NOT NULL, -- 'brc', 'pwcr', 'mbe', 'insurance', 'w9', 'capabilities', 'bond', 'safety_record'
  file_name TEXT NOT NULL,
  file_url TEXT NOT NULL, -- R2 URL
  file_size INTEGER,
  expires DATE,
  status TEXT DEFAULT 'valid', -- 'valid', 'expiring_soon', 'expired'
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Registrations (contractor × entity)
CREATE TABLE registrations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  contractor_id UUID REFERENCES contractors(id),
  entity_id UUID REFERENCES entities(id),
  status TEXT DEFAULT 'not_started',
    -- 'not_started', 'in_progress', 'submitted', 'approved', 'rejected', 'expired'
  submitted_at TIMESTAMPTZ,
  approved_at TIMESTAMPTZ,
  expires_at TIMESTAMPTZ,
  documents_used UUID[], -- references documents.id
  portal_used TEXT,
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(contractor_id, entity_id)
);

-- Bids (scraped from portals)
CREATE TABLE bids (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  entity_id UUID REFERENCES entities(id),
  title TEXT NOT NULL,
  description TEXT,
  category TEXT,
  posted_at TIMESTAMPTZ,
  due_at TIMESTAMPTZ,
  budget_estimate INTEGER, -- in cents
  url TEXT,
  portal_source TEXT,
  status TEXT DEFAULT 'open', -- 'open', 'closed', 'awarded'
  awarded_to UUID REFERENCES contractors(id),
  awarded_amount INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Historical Prices (THE MOAT)
CREATE TABLE historical_prices (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  entity_id UUID REFERENCES entities(id),
  category TEXT NOT NULL, -- 'paving', 'sealcoating', 'roofing'
  winning_amount INTEGER, -- in cents
  winning_contractor TEXT,
  bid_date DATE,
  quantity NUMERIC, -- sq ft, sq yards, etc.
  unit TEXT, -- 'sq_ft', 'sq_yd', 'linear_ft'
  price_per_unit NUMERIC,
  source_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Compliance Items (cert tracking)
CREATE TABLE compliance_items (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  contractor_id UUID REFERENCES contractors(id),
  type TEXT NOT NULL, -- 'cert', 'insurance', 'license', 'safety'
  name TEXT NOT NULL,
  issuer TEXT,
  number TEXT,
  issued_at DATE,
  expires DATE,
  status TEXT DEFAULT 'valid', -- 'valid', 'expiring_soon', 'expired'
  document_id UUID REFERENCES documents(id),
  reminder_sent_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Invoices
CREATE TABLE invoices (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  contractor_id UUID REFERENCES contractors(id),
  entity_id UUID REFERENCES entities(id),
  amount INTEGER NOT NULL, -- in cents
  description TEXT,
  work_completed_at DATE,
  submitted_at TIMESTAMPTZ,
  paid_at TIMESTAMPTZ,
  status TEXT DEFAULT 'draft',
    -- 'draft', 'submitted', 'approved', 'paid', 'overdue'
  stripe_invoice_id TEXT,
  payment_method TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Email Events (tracking)
CREATE TABLE email_events (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  contractor_id UUID REFERENCES contractors(id),
  email_type TEXT NOT NULL,
  event_type TEXT NOT NULL, -- 'sent', 'delivered', 'opened', 'clicked', 'bounced', 'complained'
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Vercel (Hosting)
**Purpose:** Landing page + dashboard hosting
- Next.js 14 (App Router)
- Edge functions for form handling
- Preview URLs for testing
- Custom domain: govready.app

### Stripe (Payments)
**Purpose:** Subscriptions + one-time payments
- $500 setup fee (one-time)
- $149/$299/$499 monthly subscriptions
- 1% bid bond commission (Phase 3)
- 0.5% success fee (Phase 3)

### Resend (Email)
**Purpose:** Transactional emails
- Lead magnet delivery
- Nurture sequence
- Compliance alerts
- Bid notifications
- Invoice reminders

---

## LAYER 2: DATA (THE MOAT)

### Entity Database Coverage Plan

| Phase | Geography | Entity Count | Source | Method |
|-------|-----------|-------------|--------|--------|
| 1 | NJ Morris County | 79 | Already built | Manual research |
| 1 | NJ Statewide | 1,163 | NJDOE + Census | Download + enrich |
| 2 | PA Statewide | 2,556 | PA Dept of Ed + Census | Download + enrich |
| 2 | NY Statewide | 3,000+ | NYSED + Census | Download + enrich |
| 3 | CT + DE + MD + VA | 8,000+ | State directories | Download + enrich |
| 3-4 | Remaining 44 states | 55,000+ | Census of Governments | Download + enrich |
| **TOTAL** | **All 50 states** | **~70,000** | | |

### Data Quality System
- **Verification frequency:** Weekly (AI agent checks if URLs still work)
- **Contact accuracy:** Monthly (email bounce detection)
- **Portal changes:** Monitored (AI detects form field changes)
- **Data enrichment:** Continuous (every registration teaches the system)

### Historical Price Intelligence
- **Source:** Government meeting minutes, award notices, bid results
- **Coverage target:** 5 years of award data per entity
- **Value:** "The winning bid for sealcoating in Denville was $2.40/sq ft in 2023"
- **Method:** AI extraction from PDF meeting minutes + structured data

---

## LAYER 3: AI + AUTOMATION

### Registration Engine (Form Filler)

**How it works:**
1. AI agent navigates to entity registration portal
2. Identifies form fields using Claude vision
3. Maps fields to contractor data (stored in Supabase)
4. Fills form fields automatically
5. Uploads required documents
6. Submits form
7. Captures confirmation number
8. Updates registration status in database

**Fallback chain:**
1. AI agent fills form → success? Done.
2. AI agent fails → Pakistan ops fills manually → AI learns from correction
3. Portal blocks automation → Email submission fallback
4. Email not accepted → Manual registration (we do it)

**Cost per registration:**
- AI-only: $0.50 (API costs)
- AI + human review: $2.00
- Manual (Pakistan ops): $5.00

### Bid Monitor

**How it works:**
1. n8n cron job runs daily at 6 AM ET
2. Scans all registered entity bid portals
3. Extracts new bids (title, description, due date, budget)
4. Converts bid description to vector embedding
5. Matches against contractor profile (vertical, location, capacity)
6. Calculates win probability score
7. Sends alerts for high-probability bids (score > 70%)
8. Stores all bids in database for historical analysis

**Bid Matching (Vector Search):**
```
Contractor profile: "Paving contractor, NJ, $5M bonding, 20 years experience"
                    ↓
              Vector embedding [0.23, 0.87, -0.12, ...]

Bid: "Roadway improvement, 5,000 sq ft asphalt overlay, NJ municipality"
                    ↓
              Vector embedding [0.25, 0.82, -0.08, ...]

Similarity score: 0.94 → HIGH MATCH → Alert contractor
```

### Compliance Tracker

**How it works:**
1. Scans all contractor documents for expiration dates
2. Creates compliance items in database
3. Sends alerts at 90, 60, 30, 14, 7, 1 days before expiration
4. Auto-generates renewal documents where possible
5. Tracks renewal completion

**Alert channels:**
- Email (primary)
- Dashboard notification (secondary)
- SMS (Phase 3, for critical items)

---

## LAYER 4: SERVICE DELIVERY

### Registration Service

**White-glove process:**
1. Contractor signs up → uploads 6 documents
2. AI agent registers with 20-50 entities (2-3 days)
3. Pakistan ops quality checks (1 day)
4. Delivered: Registration report with all credentials
5. Contractor reviews → approves → service complete

**Self-serve process (Phase 2):**
1. Contractor signs up → uploads documents
2. AI agent registers automatically
3. Contractor monitors via dashboard
4. Alerts for issues → contractor resolves or requests help

### Pre-Qualification Generator

**How it works:**
1. Contractor selects target state (e.g., NJ DOT)
2. System pulls NJ DOT pre-qual requirements
3. AI generates pre-qual packet using contractor data:
   - Financial statements (from uploaded docs)
   - Work history (from registration data)
   - Safety records (from EMR rating)
   - Equipment list (from contractor profile)
   - Personnel qualifications
4. Pakistan ops reviews packet
5. Delivered: 200-500 page pre-qual PDF, ready to submit

**Value:** Contractors normally spend 40+ hours on pre-qual. We do it in 2 hours.

### Bid Intelligence

**Win Probability Score:**
```
Score = f(
  contractor_bonding_capacity / project_bonding_requirement,
  contractor_experience / project_complexity,
  contractor_location / project_location,
  historical_win_rate,
  competitor_count,
  project_size / contractor_capacity
)

Score > 80%: "High probability — bid now!"
Score 50-80%: "Medium probability — review carefully"
Score < 50%: "Low probability — skip unless strategic"
```

**Price Recommendation:**
```
Based on historical data for this entity + category + size:
- Average winning price: $2.40/sq ft
- Range: $1.80 - $3.20/sq ft
- Recommended bid: $2.25/sq ft (competitive but profitable)
- Expected margin: 18-22%
```

---

## LAYER 5: CLIENT INTERFACE

### Landing Page (Acquisition)
**Already documented in PRD (13_PRD.md)**

### Dashboard (Operations)
**Sections:**
1. **Overview** — Registration count, active bids, compliance status
2. **Registrations** — Per-entity status with actions
3. **Bids** — New bids, matched bids, bid history
4. **Compliance** — Cert status, expiration alerts, renewal actions
5. **Invoices** — Outstanding, paid, overdue
6. **Documents** — Uploaded docs, expiration dates
7. **Settings** — Profile, billing, notifications

### Mobile App (Phase 3)
**Key features:**
- Push notifications for new bids
- One-click "bid yes" or "bid no"
- Compliance alerts
- Invoice status check
- Quick registration status view

---

## THE INTEGRATION MAP

```
Landing Page (Vercel)
    ↓ Form submit
Supabase (INSERT lead)
    ↓ Trigger
Resend (send lead magnet email)
    ↓ Lead opens email
PostHog (track engagement)
    ↓ Lead clicks Calendly
Calendly (book call)
    ↓ Call webhook
Supabase (UPDATE lead → hot)
    ↓ Call completed
Stripe (charge $500 setup)
    ↓ Payment webhook
Supabase (INSERT contractor, UPDATE lead → customer)
    ↓ Trigger
n8n (start registration workflow)
    ↓ AI agent registers
Supabase (INSERT registrations)
    ↓ Registration complete
Resend (send completion email)
    ↓ Contractor approves
Stripe (charge $199/mo subscription)
    ↓ Monthly
n8n (scan bids, check compliance)
    ↓ New bid found
Resend (send bid alert)
    ↓ Contractor wins bid
Stripe (charge 0.5% success fee)
```

---

## THE ERROR HANDLING SYSTEM

| Error | Detection | Fallback | Alert |
|-------|-----------|----------|-------|
| Form submit fails | Client-side validation | Retry 3x, show error | Log to Supabase |
| Email bounces | Resend webhook | Flag contact as invalid | Slack to Kashif |
| AI form fill fails | Agent self-report | Queue for Pakistan ops | Dashboard alert |
| Portal blocks bot | CAPTCHA detected | Email fallback or manual | Slack to Kashif |
| Stripe payment fails | Stripe webhook | Retry in 3 days | Email to contractor |
| Calendly no-show | No calendar event after 24h | Reschedule email | Dashboard alert |
| Compliance expires | Daily cron scan | Send urgent alert | Email + dashboard |
| Data quality drops | Weekly verification scan | Flag for research | Slack to Kashif |
| Contractor churns | Cancellation webhook | Exit survey + save data | Slack to Kashif |

---

## THE MONITORING DASHBOARD (Internal)

**Real-time metrics:**
- New leads today
- Registrations completed today
- Revenue today
- Active bids matching contractor profiles
- Compliance items expiring in 30 days
- Contractor health score (engagement + payment)

**Weekly reports:**
- Lead → customer conversion rate
- Registration success rate
- Revenue growth (week over week)
- Churn rate
- Data quality score

---

## THE SCALING PLAN

### When to add Pakistan ops:
- **Trigger:** 5+ paying customers OR registrations taking >10 hrs/week of your time
- **Hire:** 2 registration specialists via LinkedIn Pakistan or Upwork
- **Cost:** $600/month each ($1,200 total)
- **Training:** 1 week using your existing process docs

### When to add fractional CTO:
- **Trigger:** System reliability issues OR need for complex features
- **Hire:** Senior Pakistani developer, part-time (20 hrs/week)
- **Cost:** $1,500-2,500/month
- **Responsibility:** Security, backups, performance, code quality

### When to add US customer success:
- **Trigger:** 50+ customers OR support taking >10 hrs/week
- **Hire:** Part-time US-based contractor
- **Cost:** $2,000-3,000/month
- **Responsibility:** Onboarding calls, support emails, churn prevention

---

## THE COMPLETE FILE STRUCTURE (When You Say "Code")

```
govready/
├── app/
│   ├── (marketing)/
│   │   ├── page.tsx              # Landing page
│   │   ├── layout.tsx            # Marketing layout
│   │   └── pricing/
│   │       └── page.tsx          # Pricing section
│   ├── (dashboard)/
│   │   ├── dashboard/
│   │   │   ├── page.tsx          # Main dashboard
│   │   │   ├── registrations/
│   │   │   │   └── page.tsx      # Registration list
│   │   │   ├── bids/
│   │   │   │   └── page.tsx      # Bid feed
│   │   │   ├── compliance/
│   │   │   │   └── page.tsx      # Compliance tracker
│   │   │   └── invoices/
│   │   │       └── page.tsx      # Invoice list
│   │   ├── layout.tsx            # Dashboard layout
│   │   └── settings/
│   │       └── page.tsx          # Settings
│   ├── api/
│   │   ├── auth/[...nextauth]/
│   │   │   └── route.ts          # NextAuth
│   │   ├── leads/
│   │   │   └── route.ts          # Lead capture
│   │   ├── stripe/
│   │   │   ├── checkout/
│   │   │   │   └── route.ts      # Stripe checkout
│   │   │   └── webhook/
│   │   │       └── route.ts      # Stripe webhook
│   │   ├── calendars/
│   │   │   └── webhook/
│   │   │       └── route.ts      # Calendly webhook
│   │   └── entities/
│   │       └── route.ts          # Entity search
│   ├── layout.tsx                # Root layout
│   └── not-found.tsx             # 404
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # Browser client
│   │   ├── server.ts             # Server client
│   │   └── admin.ts              # Service role client
│   ├── resend/
│   │   └── client.ts             # Email client
│   ├── stripe/
│   │   └── client.ts             # Stripe client
│   ├── ai/
│   │   ├── form-filler.ts        # Registration AI
│   │   ├── bid-monitor.ts        # Bid scanning AI
│   │   └── embeddings.ts         # Vector matching
│   └── utils/
│       ├── auth.ts               # Auth helpers
│       └── db.ts                 # DB helpers
├── components/
│   ├── ui/                       # shadcn/ui components
│   ├── forms/                    # Form components
│   ├── dashboard/                # Dashboard components
│   └── marketing/                # Marketing components
├── supabase/
│   └── migrations/               # SQL migrations
├── scripts/
│   ├── seed.ts                   # Seed data
│   ├── scrape-entities.ts       # Entity scraper
│   └── scrape-bids.ts            # Bid scraper
├── public/
│   └── pdf/
│       └── nj-entity-list.pdf    # Lead magnet
├── .env.local                     # Environment variables
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

## THE GSTACK WORKFLOW FOR BUILDING THIS

When you say "code," here's the exact workflow:

1. **`/office-hours`** → Already done (we have the PRD)
2. **`/plan-ceo-review`** → Already done (we have the business plan)
3. **`/plan-eng-review`** → Already done (we have this system design)
4. **Exit plan mode** → Claude Code writes the files
5. **`/review`** → Pre-landing PR, auto-fix issues
6. **`/qa`** → Real browser testing on staging
7. **`/ship`** → Tests, review, push, open PR

**The entire build should take 1 Claude Code session (~2-4 hours) for the MVP (landing page + Supabase + email capture + Stripe).**

---

*System design by OWL (Hermes Agent) | June 27, 2026*
*Ready for gstack execution when you say "code"*
