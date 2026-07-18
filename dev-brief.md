# Dev Brief: Waitlist API Integration — Signal Quality Checker

**From:** Marketing / Product
**To:** Dev Team
**Priority:** Medium — current fallback works and is honest about its limits, this upgrades it to real verification

---

## Context

The buffer site at **billionsignal.com** hosts a Signal Quality Checker tool (`checker.html`). Users answer 13 questions, get an instant structural quality score, then have to be on the Billionaire Signal waitlist to see the full itemised breakdown.

Since the buffer site is a static page with no backend, and the waitlist lives on **billionairesignal.com** (a different domain), we currently **cannot verify** whether an email is actually on the waitlist. The UI is honest about this: it offers two paths, and neither claims a fake confirmation.

- **"I'm already on the waitlist"** → user types their email → results unlock immediately, labelled clearly as unverified/self-reported ("this unlocks your results on trust")
- **"I'm new here"** → opens `billionairesignal.com/waitlist` in a new tab (no email asked yet) → user joins there → comes back → types the email they signed up with → we *attempt* a real check (see endpoint below) → if that fails (because the endpoint doesn't exist yet), it falls back to the same honest self-reported unlock

**What we need:** the two endpoints below so both paths become real verification instead of self-report.

---

## Endpoint 1 — Check waitlist membership (used on "I'm new here" return step)

```
GET https://billionairesignal.com/api/waitlist-check?email={email}
```

### Response

```json
// 200 OK
{ "subscribed": true }
// or
{ "subscribed": false }
```

This is already wired into `checker.html` (`unlockAfterJoin()`). Today the fetch fails (endpoint doesn't exist / CORS-blocked) and the code catches that and falls back to an honest "we can't auto-verify yet" message. The moment this endpoint is live with correct CORS, the checker automatically starts giving real confirmations — no further changes needed on our side.

**Response behavior expected by the checker:**
- `subscribed: true` → shows "You're confirmed on the waitlist" and unlocks
- `subscribed: false` → shows "we couldn't find that email yet" with a retry + join link, does **not** unlock
- Any network/CORS/timeout failure → falls back to self-reported unlock (current safe default)

---

## Endpoint 2 — Subscribe directly (optional, nice-to-have)

```
POST https://billionairesignal.com/api/waitlist-subscribe
```

### Request Body (JSON)

```json
{
  "email": "user@example.com",
  "source": "signal-checker",
  "score": 9,
  "score_total": 13,
  "score_band": "Incomplete"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `email` | string | yes | Validate format server-side |
| `source` | string | yes | Always `"signal-checker"` from this tool |
| `score` | integer | no | The user's raw score (0–13) |
| `score_total` | integer | no | Always 13 |
| `score_band` | string | no | One of: `"Well-Structured"`, `"Incomplete"`, `"Major Gaps"` |

### Response

```json
{ "success": true }
```

This isn't wired into the checker's JS yet, since the current flow deliberately sends new users to the *real* waitlist page on billionairesignal.com to sign up themselves (so signup always goes through your actual form/validation/consent flow, not a shadow copy of it). This endpoint would let us skip that redirect entirely and subscribe someone directly from the buffer site — worth doing eventually, but Endpoint 1 (verification) is the priority since it's what makes the current UI honest end-to-end.

---

## CORS Headers Required (both endpoints)

The requests come from `https://billionsignal.com` (a different domain), so both endpoints **must** include these CORS headers on every response including preflight OPTIONS:

```
Access-Control-Allow-Origin: https://billionsignal.com
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

If you're on Cloudflare (which you are), this can be set as a Transform Rule or via a Worker — no code change needed on the main site's existing pages.

---

## Email Platform Tagging (applies to Endpoint 2 if/when built)

When adding to the waitlist, tag/segment the subscriber with:

| Tag | Value |
|-----|-------|
| `source` | `signal-checker` |
| `score_band` | `well-structured` / `incomplete` / `major-gaps` |

This enables targeted follow-up sequences, e.g.:
- **Major Gaps** → "Here's what most people miss when evaluating signals" educational sequence
- **Well-Structured** → "You clearly know what to look for — here's what BS signals do differently" nurture flow
- All → Signal Quality Checklist PDF delivery

---

## Alternative: Use Email Platform API Directly

If the waitlist runs on **Beehiiv, ConvertKit, Mailchimp, or similar**, these platforms typically have a "check subscriber status" lookup endpoint already — we may be able to call that directly instead of building Endpoint 1 from scratch.

Provide us with:
- The platform name
- A **public-only** API key scoped to read-only subscriber lookup (no write/delete)
- The list/publication ID

We'll wire it in on our end. This is the fastest path if you're already on one of these platforms.

---

## Where to Make the Code Change (Buffer Site)

Once Endpoint 1 is live, no change is needed on our side — find this in `checker.html`'s `unlockAfterJoin()` function:

```javascript
fetch(`https://billionairesignal.com/api/waitlist-check?email=${encodeURIComponent(email)}`, ...)
```

It's already pointed at the correct URL and already handles both the success and failure response shapes described above. Ship the endpoint with correct CORS and it starts working immediately.

---

## Summary of Requests (Prioritised)

| # | Task | Effort | Impact |
|---|------|--------|--------|
| 1 | `GET /api/waitlist-check?email=` with CORS | S | Turns the "I'm new here" return step into real verification instead of self-report |
| 2 | `POST /api/waitlist-subscribe` with CORS (optional) | S–M | Lets us skip the redirect to billionairesignal.com entirely, if desired later |
| 3 | Tag subscribers by `source` + `score_band` | S | Enables segmented follow-up sequences |
| 4 | Trigger PDF delivery email on subscribe | M | Completes the promised "Signal Quality Checklist PDF" UX |

---

*Questions: ping the marketing team. The checker tool is live at billionsignal.com/checker — test it yourself to understand the UX flow, including both the "already a member" and "new here" paths.*
