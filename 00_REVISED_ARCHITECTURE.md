# EVERLINE REGISTRATION SYSTEM — REVISED ARCHITECTURE
## What exists, what's missing, how they connect

---

## THE DATA WE HAVE:

| Source | File | What it contains | Status |
|--------|------|------------------|--------|
| Phase 1 calls | `Downloads/Target_Towns_Contact_List - Sheet1.csv` | 103 call records with caller comments, dispositions, emails found, bid portals identified | DONE — data exists, follow-ups NOT actioned |
| Morris Co schools | `Everline-Leads/everline_morris_county_p1.csv` | 40 school districts with BA emails + phones from NJDOE | DONE |
| Morris Co munis | MIT Media Lab dataset | 39 municipality names only | ENRICHED by me — now has QPA names/emails |
| Full NJ schools | `NJ_Combined.csv` (598 schools) | All NJ school districts with BA emails | DONE |
| Full NJ munis | `everline_master_leads.csv` (565 munis) | All NJ municipalities, minimal contacts | NEEDS ENRICHMENT |
| PA entities | `PA_Combined.csv` (2,556) | All PA municipalities | FUTURE PHASE |

---

## WHAT I BUILT WRONG:

1. **MASTER_TRACKER.csv** — I created this from scratch with 79 Morris County entities. But it doesn't connect to the Phase 1 call sheet. If someone already called Parsippany and got a name/email, that intel should be IN the tracker, not lost in a separate CSV.

2. **Ignored caller comments** — The Phase 1 sheet has 41 actionable follow-ups (emails to send, callbacks to make, contacts to reach). These are WARMER than any cold lead. They should be the FIRST actions, buried in a separate CSV.

3. **No single source of truth** — Data is scattered across 5+ files. The caller needs ONE place to see everything: what was called, what was found, what to do next.

---

## THE FIX (What I'm building now):

### 1. UNIFIED MASTER SHEET
Replace MASTER_TRACKER.csv with a single sheet that combines:
- Phase 1 caller intelligence (comments, emails found, dispositions)
- Morris County enriched data
- All 79 Morris entities in ONE place
- Clear "NEXT ACTION" for each entity based on what's already been done

### 2. PHASE 1 FOLLOW-UP FIRST
Before any new outreach, action the 41 follow-ups from Phase 1:
- 28 entities need emails sent (caller found the email, nobody sent it)
- 5 entities need callbacks
- 8 entities need the email Again

### 3. ONE TRACKER TO RULE THEM ALL
Single CSV with columns:
- Entity Name
- Entity Type (School/Municipal)
- County
- Phase 1 Status (what caller found)
- Phase 1 Caller Notes (verbatim)
- Email Found (from caller or enrichment)
- Phone Found (from caller or enrichment)
- Bid Portal Found (from caller notes)
- Next Action (specific: "Email JPapetti@ElizabethNJ.org" not just "send email")
- Lane (A/B/C)
- Status (Not Started / In Progress / Registration Confirmed / Dead End)
- Follow-Up Date
- Registration Confirmed Date

---

## EXECUTION ORDER (Revised):

### WEEK 1 PHASE 0 — Mine Phase 1 for wins (Day 1, 2 hours):
1. Extract every "send email" from Phase 1 → send those emails TODAY
2. Extract every "decision maker reached" → send follow-up email TODAY
3. Extract every "needs follow-up" → call back TODAY
4. Every entity that said "send email" + has a known bid portal → register on portal + send email

### WEEK 1 — Morris County (Days 1-5):
Same as before but using unified tracker

### WEEK 2-3 — Essex + Union Counties (Phase 1 was already warm):
Same process but now following up on partially leads AND hitting new entities

### WEEK 4+ — Remaining 18 NJ counties:
Scale with the same system
