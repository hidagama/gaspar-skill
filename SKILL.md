# Gaspar API Skill

Operational runbook for working with [Gaspar](https://gaspar.hidagama.com), an email-campaign platform with a REST API at `https://api.hidagama.com/api/gaspar/`. **Read this before creating any campaign.** Every section maps to a failure mode hit in production — the rules exist because someone skipped them.

> If you only have 60 seconds: read the [Pre-flight checklist](#3-pre-flight-checklist-before-any-campaign-fires) and follow it religiously.

---

## Table of contents

1. [API quickstart — auth, base URL, key concepts](#1-api-quickstart)
2. [Time conversion — the #1 mistake](#2-time-conversion--the-1-mistake)
3. [Pre-flight checklist (before any campaign fires)](#3-pre-flight-checklist-before-any-campaign-fires)
4. [Full campaign lifecycle](#4-full-campaign-lifecycle)
5. [Campaign body shape (POST /campaigns)](#5-campaign-body-shape)
6. [HTML email design — table-based, mobile-safe, brand-perfect](#6-html-email-design)
7. [Subject lines — craft + limits](#7-subject-lines)
8. [Recipient management — manual, CSV, Sheets, audiences](#8-recipient-management)
9. [Template syntax (merge fields, fallbacks, special variables)](#9-template-syntax)
10. [The watchdog — and how to unpause without re-pause](#10-the-watchdog)
11. [Click-tracking + UTM attribution](#11-click-tracking)
12. [Frequency rules + sequence staggering](#12-frequency-rules)
13. [Quick-send vs full campaign vs test-send](#13-quick-send-vs-full-campaign-vs-test-send)
14. [Sequences (multi-step follow-ups)](#14-sequences)
15. [Forms (lead capture)](#15-forms)
16. [Suppressions + replies + inbox](#16-suppressions-replies-inbox)
17. [Analytics — opens, clicks, GA4, deliverability](#17-analytics)
18. [Complete API endpoint reference](#18-complete-endpoint-reference)
19. [MCP tools mapping](#19-mcp-tools-mapping)
20. [Common campaign patterns (cold outreach, T-1 reminder, recovery)](#20-common-patterns)
21. [Five mistakes worth memorizing](#21-five-mistakes)
22. [When to invoke this skill](#22-when-to-invoke)

---

## 1. API quickstart

**Base URL:** `https://api.hidagama.com/api/gaspar/`
**App URL (UI only — do not call):** `https://gaspar.hidagama.com/app`
**Auth:** Bearer token in `Authorization` header.

```bash
export GASPAR_API_KEY=gsk_...    # mint from gaspar.hidagama.com → Settings → API
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/auth/check
```

**Key shape:** Gaspar API keys start with `gsk_`. They're scoped — recommended scopes for AI-assistant integrations:
- `campaigns:read` — list campaigns, recipients, stats
- `campaigns:write` — create + edit drafts (no send)
- `campaigns:launch` — actually fire campaigns (leave OFF when an AI controls the key)

**Never** commit the key. **Never** print it in chat output. Cache in `GASPAR_API_KEY` env var, macOS Keychain, 1Password CLI, GitHub Actions secret — never inline.

**First call to confirm everything works:**
```bash
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/auth/check | jq
```
Returns `{ ok: true, user_id, via, scopes, plan, active, ... }`. If `ok: false` or 401, the key is wrong or expired.

**Wrong URLs that return "Not found":**
- `gaspar.hidagama.com/api/...` — that's the app frontend
- `api.hidagama.com/api/gaspar/v1/...` — no version prefix
- `api.hidagama.com/gaspar/...` — missing `/api/` segment

---

## 2. Time conversion — the #1 mistake

Gaspar stores `scheduled_start_at` as **UTC milliseconds since epoch**. Operators always think in local time. Confusing the two ships campaigns 3 hours late.

| Operator says | What you compute | Sanity check |
|---|---|---|
| "Send Friday at 6pm IL" | IL is UTC+3 in summer → UTC 15:00 Fri → ms epoch | `date -u -r $((TS_MS/1000))` should print Fri 15:00 UTC |
| "Send Monday morning Chicago time" | Chicago is UTC-5 in summer → UTC ~13:00 Mon → ms | Don't confuse operator-locale vs recipient-locale |
| "Send tomorrow at 9am" | **ALWAYS ASK: 9am whose time?** | Never guess |

**Always echo both timezones before save:**

> "Friday 18:00 IL = Friday 15:00 UTC = epoch ms 1780521060000. Confirm?"

Helpers:
```bash
TZ=Asia/Jerusalem date -d "2026-06-08 09:00"    # local → readable
TZ=UTC date -d "@1780521060"                     # epoch → UTC readable
TZ=America/Chicago date -d "@1780521060"         # epoch → recipient-local

# Compute ms epoch from local time:
TS_MS=$(TZ=UTC date -d "2026-06-08 13:30" +%s%3N)
echo "$TS_MS"
```

---

## 3. Pre-flight checklist (before any campaign fires)

Run every box. Skipping any has cost real campaigns.

- [ ] **1. Audience scope confirmed in writing** — "all ~N non-suppressed contacts from list X" not "all contacts"
- [ ] **2. Throttle rate sane for sender** — Gmail OAuth caps at ~30/hr safely. Verified custom domains allow more. Verify the connected account at `GET /accounts`
- [ ] **3. `scheduled_start_at` echoed in BOTH operator-local AND UTC** before save
- [ ] **4. Subject ≤60 chars** — Gmail mobile truncates at ~60. Run `echo -n "subject" | wc -c`
- [ ] **5. Pre-header text set** — the hidden text after the subject in inbox preview. Use a `<div style="display:none;max-height:0;overflow:hidden;...">` block as the first body element. Without it, inbox shows the first line of code or a leftover word.
- [ ] **6. Body uses `{{firstName | fallback}}` syntax** — see section 9
- [ ] **7. Unsubscribe link present in footer**: `https://api.hidagama.com/api/gaspar/u?e={{email}}&c={{campaign_id}}`. Required by CAN-SPAM + GDPR.
- [ ] **8. CTA link includes UTM**: `?utm_source=gaspar&utm_medium=email&utm_campaign=<your_slug>` — critical, see section 11.
- [ ] **9. Test-send to yourself** via `POST /test-send`. Inspect rendered output. Note: test-send skips merge-field substitution — verify the production template, not the test render.
- [ ] **10. Audience size ≥ 25** if you want the auto-generated PDF completion report.
- [ ] **11. Frequency check** — has this list received an email in the past 48h? Section 12.
- [ ] **12. Render check at mobile width** — open HTML in a browser at 375px. If it horizontally scrolls or text overflows, the email is broken on mobile (the majority of opens).

If you cannot tick every box, do not press Schedule.

---

## 4. Full campaign lifecycle

```
1. Mint API key                       → POST /keys (or Settings UI)
2. Connect a sender account           → POST /oauth/start  (or /outlook/start, /custom-domains)
3. Verify auth                        → GET  /auth/check
4. (Optional) Add recipients to list  → POST /audiences, /sheets-import, etc.
5. Design HTML body                   → Local — see section 6
6. Create draft campaign              → POST /campaigns (status: 'draft')
7. Test-send to yourself              → POST /test-send
8. Schedule the real send             → PATCH /campaigns/{id}/schedule
9. Monitor in-flight                  → GET  /campaigns/{id}  (poll stats)
10. Pause if needed                   → POST /campaigns/{id}/pause
11. Investigate bounces / replies     → GET  /campaigns/{id}/recipients, /replies
12. Resume after addressing issues    → POST /campaigns/{id}/resume   (see section 10)
13. Completion → auto PDF report     → arrives at reply_to address (≥25 recipients)
14. Pull analytics                    → GET  /campaigns/{id}/stats, /ga/metrics/{id}
```

Every step is an API call. Every step has gotchas. Sections 5-17 cover each in depth.

---

## 5. Campaign body shape

```json
POST /api/gaspar/campaigns
Content-Type: application/json
Authorization: Bearer $GASPAR_API_KEY

{
  "name": "Q4 Trade Show — T-1 reminder",
  "subject_template": "One last note before the show opens",
  "body_html_template": "<!doctype html><html>...</html>",
  "body_text_template": "Plain-text fallback for non-HTML clients",
  "from_name": "Your Brand",
  "reply_to": "you@yourdomain.com",
  "gaspar_account_id": "<from-/api/gaspar/accounts>",
  "throttle_per_hour": 30,
  "jitter_seconds": 0,
  "scheduled_start_at": 1780713600000,
  "track_opens": 1,
  "track_clicks": 1,
  "campaign_type": "broadcast",
  "recipients": [
    {
      "email": "recipient@example.com",
      "company": "Example Co",
      "firstName": "First",
      "lastName": "Last",
      "country": "US",
      "website": "https://example.com"
    }
  ]
}
```

**Field reference:**

| Field | Type | Notes |
|---|---|---|
| `name` | string | Internal label, shown in dashboard. Not seen by recipients. |
| `subject_template` | string | Goes in the email subject. ≤60 chars. Supports `{{merge_fields}}`. |
| `body_html_template` | string | Full HTML email. See section 6 for design rules. |
| `body_text_template` | string | Plain-text fallback. Optional but recommended. Some spam filters penalize HTML-only emails. |
| `from_name` | string | The display name shown in recipient's inbox ("Your Brand <you@example.com>"). |
| `reply_to` | string | Where replies go. Often differs from sender. |
| `gaspar_account_id` | string | Pre-connected sender account. Get from `GET /accounts`. |
| `throttle_per_hour` | number | Sends per hour. Default 30. Gmail OAuth: keep ≤30 unless you've verified higher. Custom domains: higher allowed. |
| `jitter_seconds` | number | Random delay (0-N seconds) added to each send to avoid looking robotic. Default 0. |
| `scheduled_start_at` | number (ms) | UTC milliseconds. See section 2. |
| `track_opens` | 0\|1 | Inserts 1px tracking pixel. |
| `track_clicks` | 0\|1 | Should wrap CTA links — see section 11 for the gotcha. |
| `campaign_type` | string | `broadcast` (one-shot to list), `sequence` (multi-step). |
| `recipients` | array | Flat objects. See section 8 for the schema. |
| `attachment_name` | string | Optional. PDF/image attached to every send. |
| `attachment_mime` | string | E.g. `application/pdf`. |
| `attachment_b64` | string | Base64-encoded attachment body. |
| `ab_enabled` | 0\|1 | Enable A/B subject testing. |
| `ab_subject_b` | string | Variant B subject when A/B is on. |
| `ab_body_html_b` | string | Variant B HTML body. |
| `ab_pick_after_mins` | number | After N minutes, auto-pick the winner and send the rest of the list with it. |

**Returns:**
```json
{
  "ok": true,
  "campaign": {
    "id": "uuid",
    "status": "draft",
    "total_recipients": 280,
    ...
  }
}
```

The campaign starts in `status: 'draft'`. Set `scheduled_start_at` to launch. Status flow: `draft → scheduled → running → completed` (or `paused` mid-flight).

---

## 6. HTML email design

Email rendering is a 1998-era nightmare. Gmail strips `<style>`, Outlook ignores flex/grid, iOS auto-styles, Yahoo inlines remote CSS. Follow these rules and you'll render correctly across 95%+ of clients.

### 6.1 Hard rules

1. **Inline CSS only.** Gmail strips `<style>` blocks. Every CSS rule must be on the element via `style="..."`.
2. **Table-based layout.** Outlook ignores `display: flex` and `display: grid`. Use `<table role="presentation">` with `cellpadding=0` `cellspacing=0` `border=0`.
3. **Max-width 600px.** Industry standard for desktop. Mobile clients shrink to viewport width.
4. **No external CSS.** No `<link rel="stylesheet">`. No web fonts via `@import`. System fonts only.
5. **Image-light.** Some clients block remote images by default. The email must work text-only. Don't embed the entire design in one image.
6. **Pre-header text** in a hidden div at the top of body — controls inbox preview text.
7. **Mobile-safe touch targets.** CTA buttons ≥44px tall.
8. **`x-apple-disable-message-reformatting`** meta in head — prevents iOS auto-styling.

### 6.2 Skeleton (copy-paste this)

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <meta name="x-apple-disable-message-reformatting">
  <title>Your campaign subject</title>
  <!--[if mso]>
  <style type="text/css">table {border-collapse:collapse;}</style>
  <![endif]-->
</head>
<body style="margin:0;padding:0;background:#F4F4F5;font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Inter,Helvetica,Arial,sans-serif;">

  <!-- Pre-header (hidden, shows in inbox preview) -->
  <div style="display:none;max-height:0;overflow:hidden;color:#F4F4F5;">
    Your inbox-preview line — make it count.
  </div>

  <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="100%" style="background:#F4F4F5;">
    <tr><td align="center" style="padding:24px 12px;">

      <!-- 600px container -->
      <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="600" style="max-width:600px;width:100%;">

        <!-- Brand header band (optional) -->
        <tr><td style="background:#0D0D0D;border-radius:8px 8px 0 0;padding:28px 32px;text-align:center;">
          <div style="font-size:22px;font-weight:800;letter-spacing:0.42em;color:#F2F2F2;line-height:1;">YOUR BRAND</div>
        </td></tr>

        <!-- Accent rule -->
        <tr><td style="background:#C5A059;height:3px;line-height:3px;font-size:0;">&nbsp;</td></tr>

        <!-- Body card -->
        <tr><td style="background:#FFFFFF;padding:40px 32px;">
          <p style="margin:0 0 16px;font-size:10px;font-weight:700;letter-spacing:0.28em;color:#C5A059;text-transform:uppercase;">
            Eyebrow / context
          </p>
          <h1 style="margin:0 0 24px;font-size:32px;line-height:1.15;color:#0D0D0D;font-weight:800;letter-spacing:-0.02em;">
            Your headline — 8 words max.
          </h1>
          <p style="margin:0 0 18px;font-size:16px;line-height:1.6;color:#27272A;">
            Hi {{firstName | there}},
          </p>
          <p style="margin:0 0 32px;font-size:16px;line-height:1.6;color:#27272A;">
            Body copy. Two or three short paragraphs. One idea per paragraph.
            Verb-first sentences. Active voice.
          </p>

          <!-- CTA button — pill style -->
          <table role="presentation" cellpadding="0" cellspacing="0" border="0" style="margin:0 auto 24px;">
            <tr><td align="center" style="border-radius:999px;background:#00FF94;">
              <a href="https://example.com/?utm_source=gaspar&amp;utm_medium=email&amp;utm_campaign=q4_t1"
                 style="display:inline-block;padding:18px 36px;font-size:16px;font-weight:700;line-height:1;color:#0D0D0D;text-decoration:none;border-radius:999px;">
                Your CTA &nbsp;→
              </a>
            </td></tr>
          </table>

          <p style="margin:0;font-size:13px;line-height:1.5;color:#71717A;text-align:center;">
            Trust micro-line under the button (optional).
          </p>
        </td></tr>

        <!-- Sign-off -->
        <tr><td style="background:#FFFFFF;padding:32px 32px 24px;">
          <p style="margin:0;font-size:16px;line-height:1.6;color:#0D0D0D;font-weight:700;">— Your Brand</p>
        </td></tr>

        <!-- Footer -->
        <tr><td style="background:#FAFAFA;border-radius:0 0 8px 8px;border-top:1px solid #E4E4E7;padding:24px 32px;">
          <p style="margin:0 0 6px;font-size:12px;line-height:1.6;color:#71717A;">
            <strong>Your Company</strong><br>
            Your address
          </p>
          <p style="margin:0;font-size:12px;line-height:1.6;color:#A1A1AA;">
            <a href="https://api.hidagama.com/api/gaspar/u?e={{email}}&amp;c={{campaign_id}}" style="color:#A1A1AA;text-decoration:underline;">Unsubscribe</a>
            &nbsp;·&nbsp;
            <a href="https://yourdomain.com" style="color:#A1A1AA;text-decoration:underline;">yourdomain.com</a>
          </p>
        </td></tr>

      </table>
    </td></tr>
  </table>
</body>
</html>
```

### 6.3 Common bugs and how to avoid them

| Bug | Cause | Fix |
|---|---|---|
| Outlook ignores rounded corners on the CTA | Outlook uses MSO HTML, doesn't support `border-radius` | Either accept square buttons in Outlook, OR use MSO conditional VML — see Email Geeks resources |
| CTA disappears in Outlook dark mode | Outlook dark mode inverts colors | Set explicit `background-color` on the button table cell, not just the `<a>` |
| Mobile clients shrink text below 16px | iOS Safari auto-shrinks | Set explicit font-size on every text element. Don't rely on inheritance. |
| Hidden pre-header text appears on iOS | iOS auto-respects `display: none` but preview text shows | Use the trick `<div style="display:none;max-height:0;overflow:hidden;font-size:1px;line-height:1px;color:#F4F4F5;">...</div>` |
| Gmail dark mode inverts background | Gmail auto-applies CSS filter | Use `background-color` (not `background-image`) on every wrapper; test in Gmail dark mode |
| Yahoo strips `<style>` blocks but keeps inline | Different ESPs different rules | Always inline CSS, never depend on `<style>` |
| Images blocked by default | Most ESPs block images until user clicks "show images" | Ensure email body works text-only. Don't put critical CTA in an image. |

### 6.4 Use the AI helper (optional)

Gaspar has built-in endpoints that wrap an LLM to help with email design:

- `POST /api/gaspar/generate-email` — generate a full HTML email from a brief
- `POST /api/gaspar/analyze-email` — score the HTML across deliverability, mobile fit, spam-flag risk
- `POST /api/gaspar/check-links` — verify every link in the body returns a valid URL
- `POST /api/gaspar/translate-email` — translate body to another language while preserving HTML structure

Example:
```bash
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{ "html": "<html>...</html>" }' \
  https://api.hidagama.com/api/gaspar/analyze-email | jq
```

Returns scoring + actionable recommendations (e.g. "subject too long", "missing alt text", "image-to-text ratio high — likely to trigger spam filters").

---

## 7. Subject lines

### 7.1 Hard rules

- **≤60 chars** for Gmail mobile preview (truncates at ~60).
- **Sentence case** — not Title Case ("Your inbox preview" not "Your Inbox Preview"). Title Case reads spammy in 2026.
- **Front-load the value** — first 30 chars are guaranteed visible.
- **No ALL CAPS** — spam-filter penalty + reads aggressive.
- **No `!!!`** — one `!` max. Zero is better.
- **Avoid spam-trigger words** — "FREE", "guarantee", "urgent", "act now", "$$$"
- **No emoji in B2B** — fine for B2C, distracting for B2B. Test before assuming.
- **Match the body promise** — bait-and-switch subjects spike unsubscribe rates.

### 7.2 Personalization

Subjects can use `{{merge_fields}}`. Best practice: use them when they add value, not as a vanity tactic.

| Subject | Why |
|---|---|
| ✅ "{{firstName}}, your Q4 sourcing list is ready" | Personalization adds context |
| ⚠️ "{{firstName}}, check this out" | Personalization feels desperate when content is generic |
| ✅ "Tomorrow at NeoCon — 90 seconds to set up" | No personalization, but specific + valuable |
| ❌ "{{firstName}}!!! URGENT — open NOW" | Multiple anti-patterns at once |

### 7.3 A/B testing

Gaspar supports built-in A/B testing on subject lines:

```json
{
  "ab_enabled": 1,
  "subject_template": "Variant A subject",
  "ab_subject_b": "Variant B subject",
  "ab_pick_after_mins": 60
}
```

Sends to a 10% sample with both subjects, waits 60 min, picks the winner by open rate, sends the rest of the list with it. Don't A/B everything — pick high-volume campaigns (≥1000 recipients) where signal beats noise.

---

## 8. Recipient management

### 8.1 Inline recipients (small lists)

For lists ≤ ~500, pass `recipients` array inline in `POST /campaigns`. See section 5.

### 8.2 Audiences (saved reusable lists)

For lists you'll send to repeatedly:

```bash
# Create an audience
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{ "name": "Trade Show 2026 prospects", "description": "..." }' \
  https://api.hidagama.com/api/gaspar/audiences

# List audiences
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/audiences
```

Then reference by `audience_id` instead of inline `recipients`.

### 8.3 Bulk import from Google Sheets

The fastest way to add 10-10,000 recipients:

```bash
# Step 1: read sheet metadata (validates URL + shows columns)
curl -sS -X GET -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/sheets-meta?url=https://docs.google.com/spreadsheets/d/SHEET_ID/edit"

# Step 2: import
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "url": "https://docs.google.com/spreadsheets/d/SHEET_ID/edit",
    "audience_id": "<from-step-1>",
    "column_map": {
      "email": "Email",
      "firstName": "First Name",
      "lastName": "Last Name",
      "company": "Company"
    }
  }' \
  https://api.hidagama.com/api/gaspar/sheets-import
```

The sheet must be shared "Anyone with the link can view" OR connected via OAuth.

### 8.4 Suppressions (block specific addresses)

Suppressed = blocked from ever receiving any of your campaigns. Use for hard bounces, manual blocks, GDPR deletion requests.

```bash
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{ "email": "bouncer@example.com", "reason": "hard_bounce" }' \
  https://api.hidagama.com/api/gaspar/suppressions
```

### 8.5 Contacts (your overall contact book)

`GET /contacts` returns every email address you've ever sent to across all campaigns. Useful for de-duping new lists.

### 8.6 Enrichment

Some campaigns enrich recipient data (company, country, website) via the Gaspar enrichment service:

```bash
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{ "emails": ["a@x.com", "b@y.com"] }' \
  https://api.hidagama.com/api/gaspar/enrich
```

Returns enriched fields you can merge into your recipient objects before sending.

---

## 9. Template syntax

| What works | What does NOT work |
|---|---|
| `{{firstName}}` | `{{ firstName }}` — whitespace breaks substitution |
| `{{firstName \| there}}` — pipe + fallback | `{{firstName ?? 'there'}}` — JS syntax |
|  | `{{firstName, 'there'}}` — Mustache syntax |
| `{{email}}` — special, recipient's own email at send time | `{{recipient.email}}` — no nested paths |
| `{{campaign_id}}` — special, the campaign's UUID | `{{campaign.id}}` — no nested paths |

**Test-sends show placeholders, not merged data.** `POST /test-send` renders each `{{field}}` as a highlighted `[field]` chip (and `{{email}}` as your own test address), so you can see at a glance which fields will merge. It does not pull a real recipient's row. To see genuinely merged output, send to a one-recipient list containing your own address, or use `/quick-send`.

**No first-class unsubscribe variable.** Hand-build the URL: `https://api.hidagama.com/api/gaspar/u?e={{email}}&c={{campaign_id}}`. There is no `{{unsubscribe_url}}`.

**`List-Unsubscribe` is set for you.** Every send carries `List-Unsubscribe` plus `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, so Gmail and Yahoo render their native one-click unsubscribe control. You do not need to add it. A visible unsubscribe link in the body is still good practice on top of it.

---

## 10. The watchdog — and how to unpause without re-pause

Gaspar runs a server-side deliverability watchdog that automatically pauses a campaign when its recent bounce rate crosses a safety threshold. This protects the sending domain's reputation, which outlives any single campaign. The threshold and window are deliberately not published.

**The trap:** unpausing without resolving the bounced addresses puts the same dead rows back in the queue, and the campaign pauses again. Symptom — it briefly resumes, fires a few more, stops.

**Safe unpause workflow:**

1. Identify bounced recipients: `GET /campaigns/{id}/recipients?status=bounced`
2. **Do not delete them** — they are referenced by the events table, so the delete fails. Their history is also what stops the same dead addresses being re-imported later.
3. Transition them to `suppressed`, which blocks them from this and every future send.
4. Resume: `POST /campaigns/{id}/resume`
5. Give it a few minutes and confirm it is still running.

**`bounced` vs `suppressed`:**

- `bounced` = the address failed on a recent send
- `suppressed` = blocked from receiving any further campaign from you

If it pauses again after a clean suppression pass, the list itself is the problem. Verify it before sending (section 3) rather than resuming repeatedly — repeated sends into a bad list is how a sending domain gets burned.

---

## 11. Click-tracking

Every `<a href>` in your HTML is rewritten to route through Gaspar's click tracker before redirecting to the real destination, so CTA clicks are recorded — not just the unsubscribe link. Your UTM parameters survive the redirect.

Reported click counts are **human clicks only**. Corporate security gateways and link scanners fetch every URL in an email before the recipient ever sees it; that traffic is classified separately and excluded. Your numbers will read lower than a raw hit count, and truer.

**Still put UTMs on every CTA.** Click tracking tells you a click happened; analytics tells you what happened next.

1. **Every CTA gets UTM**: `?utm_source=gaspar&utm_medium=email&utm_campaign=<your_slug>`
2. **GA4 → Acquisition → Traffic Acquisition** filtered by `Session source = gaspar` shows real arrivals.
3. **Realtime view** during a fresh fire shows clicks within seconds.

Gaspar also has a built-in GA4 connector:
```bash
# Connect GA4
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/ga/oauth/start

# After OAuth, list properties
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/ga/properties

# Pick property to track
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{ "property_id": "..." }' \
  https://api.hidagama.com/api/gaspar/ga/properties/select

# Pull per-campaign GA4 metrics (sessions, conversions)
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/ga/metrics/$CAMPAIGN_ID"
```

---

## 12. Frequency rules

Industry baselines for B2B:
- **Healthy cadence:** 1-2 sends per week to the same list
- **Re-engagement OK:** up to 3 sends in a campaign window (kickoff + 2 reminders)
- **Fatigue zone:** 4+ sends in 7 days — unsubscribe rate jumps, open rate degrades, spam complaints rise

**Before scheduling a new send:**
1. For each recipient, find most recent `sent_at` across all campaigns to that address.
2. If < 48h ago, the new send should either segment to non-receivers OR carry an explicit acknowledgement ("just making sure this reached you").
3. If you've already sent ≥3 emails to the list in 7 days, **default to NOT sending** — confirm with the campaign owner. Deliverability cost > marginal reach.

### Sequence staggering

When running a sequence (kickoff + reminder 1 + reminder 2 + post-event):

1. **Distinct `scheduled_start_at` per campaign.** Never leave two at the same timestamp.
2. **If you ever unpause multiple campaigns at once, stagger them ≥1 hour.** Otherwise all resume simultaneously and recipients see 3 emails arrive together.
3. **Before resuming, verify `scheduled_start_at` is FUTURE.** If past, campaign fires immediately on resume.
4. **After unpause, watch it for a few minutes** before walking away — confirm it is still running.

---

## 13. Quick-send vs full campaign vs test-send

Gaspar exposes three send modes:

| Endpoint | Use case | Constraints |
|---|---|---|
| `POST /campaigns` | Full multi-recipient campaign (broadcast or sequence) | Required for >1 recipient, scheduling, analytics |
| `POST /quick-send` | One-off ad-hoc send to a small list with no campaign tracking | Easier for sales-style 1-to-1 follow-ups |
| `POST /test-send` | Verify rendering — send to yourself only | Skips merge-field substitution; no campaign record |

**Use `quick-send` for:** ad-hoc sales follow-ups where you don't need open/click tracking
**Use `campaigns` for:** anything you want analytics on, anything with > ~20 recipients
**Use `test-send` for:** every pre-flight check (see section 3, step 9)

---

## 14. Sequences (multi-step follow-ups)

A sequence is a multi-step campaign where step N fires based on engagement (or lack of) with step N-1.

```bash
# Create a sequence
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Q4 prospect sequence",
    "steps": [
      { "delay_days": 0, "subject_template": "...", "body_html_template": "...", "trigger": "always" },
      { "delay_days": 3, "subject_template": "...", "body_html_template": "...", "trigger": "no_open" },
      { "delay_days": 7, "subject_template": "...", "body_html_template": "...", "trigger": "no_reply" }
    ]
  }' \
  https://api.hidagama.com/api/gaspar/sequences
```

Trigger types:
- `always` — fires for everyone after `delay_days`
- `no_open` — fires only if the previous step wasn't opened
- `no_click` — fires only if the previous step wasn't clicked
- `no_reply` — fires only if the recipient hasn't replied
- `opened_but_no_click` — engagement signal but no conversion

**Stop conditions:** recipient is automatically removed from the sequence if they reply, click a specific "stop" link, or unsubscribe.

---

## 15. Forms (lead capture)

Gaspar has a built-in form builder for capturing leads (typically embedded on a landing page):

```bash
# Create a form
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Trade Show 2026 sign-up",
    "audience_id": "<auto-add-submitters-to-this-audience>",
    "redirect_url": "https://yourdomain.com/thank-you",
    "fields": [
      { "name": "email", "type": "email", "required": true },
      { "name": "firstName", "type": "text", "required": true },
      { "name": "company", "type": "text", "required": false }
    ]
  }' \
  https://api.hidagama.com/api/gaspar/forms
```

Returns an embeddable URL or HTML snippet. Every submission appends to the linked audience automatically.

---

## 16. Suppressions, replies, inbox

### Replies (`/replies`)

When a recipient replies to a campaign, Gaspar captures it (if you've connected the inbox via OAuth). Read with:

```bash
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/replies?campaign_id=$ID&limit=50"
```

Replies are conversion-worthy — vasco/the campaign owner should reply to them manually.

### Inbound webhook (`/inbound`)

External services (e.g., Mailgun, SES inbound) can POST reply notifications to `/api/gaspar/inbound`. Configure on your ESP side.

### Suppressions (already covered in section 8.4)

Hard-bounced or unsubscribed addresses get auto-added to your suppression list. You can also manually add via `POST /suppressions`.

---

## 17. Analytics

### Per-campaign stats

`GET /campaigns/{id}` returns the campaign object PLUS a `stats` block:

```json
{
  "campaign": { ... },
  "stats": {
    "total": 280,
    "sent": 236,
    "pending": 11,
    "failed": 0,
    "bounced": 15,
    "suppressed": 46,
    "human_opens": 47,
    "clicks_total": 0,
    "clicks_unique": 0,
    "replies": 9,
    "mpp_prefetches": 0,
    "gmail_renders": 54,
    "scanner_hits": 31
  }
}
```

**Note on opens:** Gaspar splits `human_opens` (real humans) from `mpp_prefetches` (Apple Mail Privacy Protection bot prefetches) + `gmail_renders` (Gmail image proxy) + `scanner_hits` (security scanner bots). The headline number is `human_opens`. Don't trust raw `opens` if you see one — it's likely inflated by bots.

### Deliverability dashboard

```bash
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/deliverability
```

Returns bounce rate, spam-complaint rate, unsubscribe rate trending across all campaigns. Use for sender-reputation monitoring.

### Smart send time

```bash
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/smart-send-time?audience_id=$ID"
```

Returns the engagement-optimal send time for a given audience, based on their historical open patterns.

### Trade-show ROI

If you're using Gaspar with the DaGaMa trade-show ecosystem, `GET /shows` + `GET /show-roi` correlate campaigns with show-attendee captures.

---

## 18. Complete endpoint reference

| Endpoint | Method | Purpose |
|---|---|---|
| `/auth/check` | GET | Verify auth + show scopes |
| `/keys` | GET, POST | List + mint API keys |
| `/accounts` | GET | List connected sender accounts |
| `/oauth/start`, `/oauth/callback` | POST, GET | Gmail OAuth |
| `/outlook/start`, `/outlook/callback` | POST, GET | Outlook OAuth |
| `/outlook-accounts` | GET | List Outlook accounts |
| `/custom-domains` | GET, POST | Custom-domain senders (BYO domain) |
| `/campaigns` | GET, POST | List + create campaigns |
| `/campaigns/{id}` | GET, PATCH, DELETE | Read, update, delete a campaign |
| `/campaigns/{id}/schedule` | PATCH | Reschedule safely |
| `/campaigns/{id}/pause` | POST | Pause a running campaign |
| `/campaigns/{id}/resume` | POST | Resume a paused campaign |
| `/campaigns/{id}/recipients` | GET | Per-recipient state + events |
| `/quick-send` | POST | Ad-hoc one-off send |
| `/test-send` | POST | Send a test to yourself |
| `/generate-email` | POST | LLM-assisted email generation |
| `/analyze-email` | POST | LLM scoring + deliverability check |
| `/check-links` | POST | Verify every link in body |
| `/translate-email` | POST | Translate body preserving HTML |
| `/audiences` | GET, POST | Saved audience lists |
| `/contacts` | GET | Overall contact book |
| `/sheets-meta` | GET | Inspect a Google Sheet's columns |
| `/sheets-import` | POST | Import recipients from Sheets |
| `/enrich` | POST | Enrich emails with company/country |
| `/suppressions` | POST | Block an address |
| `/sequences` | GET, POST | Multi-step sequences |
| `/forms` | GET, POST | Lead-capture forms |
| `/replies` | GET | Captured replies |
| `/inbound` | POST | Webhook for external ESPs |
| `/signature` | GET, POST | Sender email signature |
| `/upload-image` | POST | Upload to Gaspar's image CDN |
| `/deliverability` | GET | Cross-campaign deliverability metrics |
| `/smart-send-time` | GET | Engagement-optimal send time |
| `/show-roi`, `/shows` | GET | Trade-show ROI (DaGaMa ecosystem) |
| `/ga/oauth/start`, `/ga/properties`, `/ga/metrics/{id}` | POST, GET | GA4 connector |
| `/meta/connect`, `/meta/callback`, `/meta/ad-accounts` | POST, GET | Meta retargeting |
| `/stripe/checkout`, `/stripe/portal`, `/stripe/status` | POST, GET | Gaspar billing |
| `/usage`, `/plan` | GET | Current plan + usage stats |

---

## 19. MCP tools mapping

The [`gaspar-mcp`](https://github.com/hidagama/gaspar-mcp) Model Context Protocol server wraps these endpoints as 13 callable tools. When installed in Claude Code (or any MCP-compatible assistant), agents can call them directly via natural-language prompts.

| MCP Tool | Wraps endpoint | Use case |
|---|---|---|
| `auth_check` | `GET /auth/check` | First call to verify connection |
| `list_accounts` | `GET /accounts` | Discover sender accounts |
| `list_campaigns` | `GET /campaigns` | See what's been sent |
| `get_campaign` | `GET /campaigns/{id}` | Full config + stats |
| `create_campaign` | `POST /campaigns` | Draft a new send |
| `add_recipients` | (inline or `/audiences`) | Build the audience |
| `preview_campaign` | `POST /test-send` | Render the merged email |
| `schedule_campaign` | `PATCH /campaigns/{id}/schedule` | Set the fire time |
| `launch_campaign` | (status change) | Fire it (requires `campaigns:launch` scope) |
| `pause_campaign` | `POST /campaigns/{id}/pause` | Emergency stop |
| `resume_campaign` | `POST /campaigns/{id}/resume` | Restart after pause |
| `get_stats` | (stats from `/campaigns/{id}`) | Pull headline metrics |
| `list_recipients` | `GET /campaigns/{id}/recipients` | Per-recipient drill-down |

Install the MCP server: see [gaspar.hidagama.com/mcp-setup](https://gaspar.hidagama.com/mcp-setup).

---

## 20. Common campaign patterns

### 20.1 Cold outreach (kickoff)

```
Audience: 200-2000 cold contacts from a target industry/show
Subject:  Specific value prop, no personalization
Body:     2-3 short paragraphs, one strong CTA, free offer below the fold
Schedule: Tue/Wed/Thu 9-10am recipient-local
Throttle: 30/hr if Gmail, 100/hr if custom domain
Follow-up: Plan E2 reminder 3 days later
```

### 20.2 T-N reminder before an event

```
Audience: Same list as kickoff
Subject:  "{{ShowName}} opens in N days — your free setup"
Body:     Acknowledge the previous send, restate the offer briefly, urgency tied to event date (NOT fake scarcity)
Schedule: 5-7 days before event for T-5; 24h before for T-1
Frequency: Max 2-3 sends in the week before
```

### 20.3 Day-of email

```
Audience: Same list, or filtered to non-openers from T-1
Subject:  "{{ShowName}} is open — 90 seconds to set up"
Body:     Energy + urgency tied to the event (not fake). Three-step block ("1, 2, 3").
Schedule: 30-60 min BEFORE event opens, in recipient timezone
```

### 20.4 Post-event follow-up

```
Audience: Everyone, including the ones who didn't engage
Subject:  "What we saw at {{ShowName}} this year"
Body:     Recap value, then soft CTA to the next event or product
Schedule: 1-3 days after event ends (Monday after, ideally)
```

### 20.5 Re-engagement / recovery

```
Audience: Recipients who opened but didn't click previous sends
Subject:  "Just making sure this reached you" (Sunday-style framing)
Body:     Acknowledge the previous send, restate the value, no pressure
Schedule: 7-14 days after the last send
```

---

## 21. Five mistakes worth memorizing

1. **Local-time / UTC confusion** when setting `scheduled_start_at`. Always echo both timezones before save (section 2).
2. **Watchdog cascade-pauses a sequence.** Bounce spike on E1 auto-pauses; unpausing without addressing bounced rows triggers re-pause. → Transition bounced → suppressed FIRST (section 10).
3. **Multiple paused campaigns resume simultaneously.** Recipients see 3-4 emails arrive at once. → Stagger unpauses ≥1 hour (section 12).
4. **Raw click counts look lower than you expected.** Scanner and gateway traffic is filtered out; only human clicks are reported. → Cross-check against your analytics layer via UTMs (section 11).
5. **Sending the 4th email in 5 days to the same list.** Open rate degrades, unsubscribes spike. → Frequency check (section 12). Acknowledge over-send in body if you must send.

---

## 22. When to invoke this skill

1. Creating any new campaign (always)
2. Designing or editing email HTML (section 6)
3. Writing or revising subject lines (section 7)
4. Building or importing a recipient list (section 8)
5. Scheduling or rescheduling (sections 2 + 12)
6. Pausing or unpausing (section 10)
7. Debugging "0 clicks" symptoms (section 11)
8. Investigating watchdog auto-pauses (section 10)
9. Designing a multi-step sequence (section 14)
10. Building a lead-capture form (section 15)
11. Pulling per-campaign analytics (section 17)
12. Anyone asks "why didn't this email send" — work the pre-flight checklist (section 3) in reverse
13. Onboarding a new operator who hasn't run a Gaspar campaign before — read sections 1 → 13

**If the action isn't covered here and you don't know what to do, stop and ask.** Don't experiment in production. Recipient lists are unrecoverable.

---

## Related projects

- **[gaspar-mcp](https://github.com/hidagama/gaspar-mcp)** — Model Context Protocol server. Wraps the Gaspar REST API as MCP tools so an AI assistant can call them directly. Install both for best results: the MCP gives the agent the verbs, this skill gives it the judgement.
- **Gaspar app** — `https://gaspar.hidagama.com/app` (browser UI for campaign management)
- **MCP setup guide** — [gaspar.hidagama.com/mcp-setup](https://gaspar.hidagama.com/mcp-setup)

## License

MIT
