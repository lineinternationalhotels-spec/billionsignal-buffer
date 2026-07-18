# Dev Brief: Waitlist API Integration — Signal Quality Checker

**From:** Marketing / Product  
**To:** Dev Team  
**Priority:** Medium — current fallback works, this upgrades the integration  

---

## Context

The buffer site at **billionsignal.com** hosts a Signal Quality Checker tool. Users fill in a 13-question form, get an instant structural quality score, then enter their email to unlock the full itemised breakdown.

That email entry = joining the Billionaire Signal waitlist on **billionairesignal.com**.

Right now (before this API exists), the checker client-side reveals the breakdown immediately and opens `https://billionairesignal.com/waitlist?email={email}` in a new tab as a fallback. It works, but the email isn't reliably captured in our system without the user completing the waitlist form themselves.

**What we need:** A proper API endpoint so the email is captured automatically, silently, when the user clicks "Unlock Free →" on the checker — without them needing to do anything on the main site.

---

## What to Build

### Endpoint

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
// 200 OK — subscribed or already on list
{ "success": true }

// 400 Bad Request — invalid email
{ "success": false, "reason": "invalid_email" }

// 409 Conflict — optional, if you want to distinguish new vs existing
{ "success": false, "reason": "already_subscribed" }
```

The checker ignores non-200 responses gracefully — it always reveals the breakdown to the user regardless of API outcome. So a 500 or timeout won't break UX, it'll just silently fall back to the waitlist page redirect.

---

## CORS Headers Required

The request comes from `https://billionsignal.com` (a different domain), so the endpoint **must** include these CORS headers on every response including preflight OPTIONS:

```
Access-Control-Allow-Origin: https://billionsignal.com
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

If you're on Cloudflare (which you are), this can be set as a Transform Rule or via a Worker — no code change needed on the main site.

---

## Email Platform Tagging

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

If the waitlist runs on **Beehiiv, ConvertKit, Mailchimp, or similar**, we can call their subscriber API directly from the buffer site's client-side JS — skipping the need to build a custom endpoint entirely.

Provide us with:
- The platform name
- A **public-only** API key scoped to subscriber creation only (no read, no delete)
- The list/publication ID

We'll wire it in on our end. This is the fastest path if you're already on one of these platforms.

---

## Where to Make the Code Change (Buffer Site)

Once the API is live, find this comment in `checker.html`:

```javascript
// API_ENDPOINT: replace with live endpoint when devs ship it
fetch('https://billionairesignal.com/api/waitlist-subscribe', { ... })
```

This is already the correct endpoint URL — no change needed on our side once you've built it. Just ship the endpoint and it starts working automatically.

---

## Pre-fill Fallback (Currently Active)

Until the API is live, the checker opens this URL in a new tab after email entry:

```
https://billionairesignal.com/waitlist?email={encoded_email}
```

**Request:** Make the waitlist page on billionairesignal.com read the `?email=` query parameter and pre-fill the email input. This makes the fallback seamless — the user lands on the waitlist page with their email already filled in, they just hit Submit.

This is a small frontend change (one line of JS on the waitlist page) and is independent of the API work.

---

## Summary of Requests (Prioritised)

| # | Task | Effort | Impact |
|---|------|--------|--------|
| 1 | Pre-fill waitlist form from `?email=` query param | XS — 1 line JS | Fixes fallback UX immediately |
| 2 | `POST /api/waitlist-subscribe` endpoint with CORS | S–M | Proper cross-site capture |
| 3 | Tag subscribers by `source` + `score_band` | S | Enables segmented sequences |
| 4 | Trigger PDF delivery email on subscribe | M | Completes the promised UX |

---

*Questions: ping the marketing team. The checker tool is live at billionsignal.com/checker.html — test it yourself to understand the UX flow.*
