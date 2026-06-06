# Gaspar API Skill

Operational runbook for working with the [Gaspar](https://gaspar.hidagama.com) email-campaign API. **Read this before creating, scheduling, pausing, or unpausing any campaign.** Every section maps to a real failure mode someone has hit in production — the checklist exists because someone skipped it.

> If you only have 30 seconds: jump to the [Pre-flight checklist](#3-pre-flight-checklist) and follow it religiously.

---

## 1. API endpoints

The Gaspar backend lives at `https://api.hidagama.com/api/gaspar/*`. The browser app is at `https://gaspar.hidagama.com/app` (UI only — don't try to call API paths against that host).

**Auth:** Bearer token in the `Authorization` header.

```bash
export GASPAR_API_KEY=...    # provision from gaspar.hidagama.com → Settings → API
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  https://api.hidagama.com/api/gaspar/campaigns
```

Never commit the key. Don't print it in chat output. Use a secrets manager (macOS Keychain, 1Password CLI, env-var, GitHub Actions secret) — never inline.

**Common endpoints:**

| Verb | Path | Returns |
|---|---|---|
| GET | `/api/gaspar/campaigns` | Campaign list with headline stats (sent/opens/bounces/replies). **`scheduled_start_at` is NOT in this response** — must hit the detail endpoint. |
| GET | `/api/gaspar/campaigns/{id}` | `{ campaign: {...full config including body_html_template, scheduled_start_at, track_clicks}, stats: {...} }` — note the nested shape |
| GET | `/api/gaspar/campaigns/{id}/recipients?limit=N&status=sent` | Per-recipient state + `last_event_type` (open / click / reply / none) + `last_event_at` |
| POST | `/api/gaspar/campaigns` | Create campaign — required body shape in section 4 |
| PATCH | `/api/gaspar/campaigns/{id}/schedule` | Reschedule a campaign safely (use this, not raw PATCH of scheduled_start_at) |
| POST | `/api/gaspar/campaigns/{id}/pause` | Pause a running campaign |
| POST | `/api/gaspar/campaigns/{id}/resume` | Resume a paused campaign (read section 6 first) |
| GET | `/api/gaspar/accounts` | Sender accounts (Gmail / Outlook / custom-domain) connected to current user |

**Wrong paths that return "Not found":**
- `gaspar.hidagama.com/api/campaigns` — wrong host (that's the app frontend)
- `api.hidagama.com/api/gaspar/v1/campaigns` — there is no `v1` prefix
- `api.hidagama.com/gaspar/campaigns` — missing the `/api/` segment

If you get "Not found", you're almost certainly on the wrong path or host. Don't add a version prefix to "fix" it.

---

## 2. Time conversion — the #1 mistake

Gaspar stores `scheduled_start_at` as **UTC milliseconds since epoch**. Operators almost always think in their local time. Confusing the two is how campaigns ship 3 hours late.

| Operator says | What you must compute | Sanity check |
|---|---|---|
| "Send Friday at 6pm IL" | IL is UTC+3 in summer → UTC 15:00 Friday → ms epoch | `date -u -r $((TS_MS/1000))` should print Fri 15:00 UTC |
| "Send Monday morning Chicago time" | Chicago is UTC-5 in summer → UTC ~13:00 Mon → ms epoch | Don't confuse operator-locale vs recipient-locale |
| "Send tomorrow at 9am" | **ALWAYS ASK: 9am whose time?** | Never guess. Operator/recipient/UTC ambiguity bites every time. |

**Always do the conversion explicitly and echo it back before scheduling:**

> "Friday 18:00 IL = Friday 15:00 UTC = epoch ms 1780521060000. Confirm?"

Don't say "scheduling for 6pm" — that's ambiguous. Say the full conversion.

```bash
TZ=Asia/Jerusalem date -d "2026-06-08 09:00"     # local → readable
TZ=UTC date -d "@1780521060"                      # epoch → UTC readable
TZ=America/Chicago date -d "@1780521060"          # epoch → recipient-local
```

---

## 3. Pre-flight checklist

Run every box before pressing Schedule. Skipping any one of these has cost real campaigns.

- [ ] **1. Audience scope confirmed in writing** — "all ~N non-suppressed contacts from list X" not "all contacts" (ambiguity → wrong recipients)
- [ ] **2. Throttle rate sane for sender** — Gmail OAuth caps at ~30/hr safely. Verified custom domains allow much more. Verify the connected account at `/api/gaspar/accounts`.
- [ ] **3. `scheduled_start_at` echoed in BOTH operator-local AND UTC** before save (see section 2)
- [ ] **4. Subject ≤60 chars** — Gmail mobile preview truncates at ~60. Run `echo -n "subject" | wc -c`.
- [ ] **5. Body uses `{{firstName | fallback}}` syntax — NOT `{{firstName ?? 'fallback'}}` or `{{firstName, 'fallback'}}`**. See section 5 for full template syntax.
- [ ] **6. Unsubscribe link present in footer**. Format: `https://api.hidagama.com/api/gaspar/u?e={{email}}&c={{campaign_id}}`. Hard requirement under CAN-SPAM + GDPR.
- [ ] **7. CTA link includes UTM**: `?utm_source=gaspar&utm_medium=email&utm_campaign=<your_campaign_slug>`. Critical — see section 7 on click-tracking. UTM lets a GA4/analytics layer catch the click even when Gaspar's wrapper misses.
- [ ] **8. Test-send to yourself first.** Inspect the actual sent message — subject, body render, CTA URL, unsubscribe link works. Note: test-sends skip merge-field substitution (section 5), so confirm placeholders by re-checking the template HTML before scheduling.
- [ ] **9. Audience size ≥ 25** if you want the auto-generated PDF completion report. Smaller campaigns skip the report.
- [ ] **10. Frequency check** — has this list received an email in the past 48 hours? If yes, justify the next send in writing or risk fatigue + unsubscribe spike (section 9).

If you cannot tick every box, do not press Schedule.

---

## 4. Creating a campaign — request body shape

```json
POST /api/gaspar/campaigns
{
  "name": "Q4 Trade Show — T-1 reminder",
  "subject_template": "One last note before the show opens",
  "body_html_template": "<!doctype html><html>...</html>",
  "body_text_template": "Plain-text fallback for non-HTML clients",
  "from_name": "Your Brand",
  "reply_to": "you@yourdomain.com",
  "gaspar_account_id": "<your-account-id-from-/api/gaspar/accounts>",
  "throttle_per_hour": 30,
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

**Recipient shape gotchas:**
- **FLAT structure only** — every merge field is a top-level key. NO nested `merge_fields: { firstName: ... }`. Every top-level key becomes a `{{key}}` substitution in the template directly.
- **No `id` field at create time** — Gaspar mints IDs internally.
- **`email` is the dedup key** — sending the same campaign with overlapping recipient emails creates one row each, but only the first will receive.

---

## 5. Template syntax

| What works | What does NOT work |
|---|---|
| `{{firstName}}` | `{{ firstName }}` — whitespace breaks substitution |
| `{{firstName \| there}}` — pipe with fallback | `{{firstName ?? 'there'}}` — JS syntax |
|  | `{{firstName, 'there'}}` — Mustache syntax |
|  | `{{ firstName \| 'there' }}` — Jinja-style quoting |
| `{{email}}` — special, substituted at send time for the recipient's own email; used in the unsubscribe URL | `{{recipient.email}}` — no nested paths |
| `{{campaign_id}}` — special, substituted by Gaspar at send time; used in the unsubscribe URL | `{{campaign.id}}` — no nested paths |

**Test-sends skip merge-field substitution.** When you send a test to yourself, `{{firstName}}` renders as the literal string `{{firstName}}`. This is a Gaspar limitation, not a bug. Verify substitution by inspecting the production HTML template at `GET /campaigns/{id}` — what you see is what gets sent (with substitutions overlaid at send time).

**No first-class unsubscribe variable.** Hand-build the URL: `https://api.hidagama.com/api/gaspar/u?e={{email}}&c={{campaign_id}}`. There is no `{{unsubscribe_url}}` variable.

**No List-Unsubscribe header.** Gaspar doesn't set the standards-compliant header automatically — body-link unsubscribe is the only path today.

---

## 6. The watchdog — and how to unpause without re-pause

Gaspar runs a server-side watchdog every few minutes that **auto-pauses any campaign with a bounce rate exceeding ~5%** over the trailing 24-hour window.

**Math:**
```
bounce_rate = bounces_in_last_24h / total_sent_in_last_24h
if bounce_rate > 0.05: pause campaign
```

**The trap:** if you unpause a paused campaign WITHOUT changing the math, the next watchdog tick re-pauses it. Symptom — campaign briefly resumes, fires 3-5 more, gets re-paused.

**Safe unpause workflow:**

1. Identify the bounced rows: `GET /campaigns/{id}/recipients?status=bounced`
2. **Do NOT delete them** — foreign-key constraints from the events table will fail.
3. Mark them as `suppressed` instead — there's an admin endpoint for this on Gaspar Pro plans, or use the bulk-status-update flow.
4. `suppressed` means "blocked from sending" (unsubscribed, hard-bounced previously, manual block) — does NOT count in the watchdog's bounce-rate math.
5. Resume: `POST /campaigns/{id}/resume`
6. Watch the next 1-2 watchdog ticks (~10 min). If it stays running, you're done.

**`bounced` vs `suppressed` — important distinction:**
- `bounced` = recent bounce, counts in watchdog math
- `suppressed` = blocked from sending, does NOT count in watchdog math

---

## 7. The click-tracking gotcha

**Known limitation:** Gaspar's link-rewriter wraps the unsubscribe link automatically, but does NOT wrap body CTAs even when `track_clicks: 1` is set. Result: every click on the main email CTA goes straight to the destination URL — invisible to Gaspar's click counter. Campaign stats show `clicks: 0` even when real humans are clicking.

**Workaround: attribute clicks via your analytics layer (GA4, Plausible, etc.):**

1. **Every CTA must include UTM**: `?utm_source=gaspar&utm_medium=email&utm_campaign=<your_slug>`
2. **GA4 → Acquisition → Traffic Acquisition** filtered by `Session source = gaspar` shows real arrivals.
3. **Realtime view** during a fresh fire shows clicks within seconds.

**Do not "fix" 0 clicks** by debugging copy or design until you've checked your analytics layer. The clicks ARE happening — they're just not counted by Gaspar.

This is a tracked limitation. When the wrapper is patched, this section will get a "fixed in version X" note.

---

## 8. Frequency rules — don't burn the list

Industry baselines for B2B campaigns:
- **Healthy cadence:** 1-2 sends per week to the same list
- **Re-engagement OK:** up to 3 sends in a campaign window (kickoff + 2 reminders)
- **Fatigue zone:** 4+ sends in a 7-day window — unsubscribe rate jumps, open rate degrades, spam complaints rise

**Before scheduling a new send to an existing list:**

1. Check the last send time. For each recipient, find the most recent `sent_at` across all campaigns to that address.
2. If the last send was < 48h ago, the new send should either:
   - **Segment** to only non-receivers / non-openers of the previous send, OR
   - **Carry an explicit acknowledgement** (e.g. "just making sure this reached you — apologies if it's a duplicate")
3. If you've already sent ≥3 emails to this list in 7 days, **default to NOT sending** — confirm with the campaign owner. The deliverability cost outweighs the marginal reach.

---

## 9. Staggered campaign sequences

When running a sequence of campaigns to the same list (kickoff + reminder 1 + reminder 2 + post-show, etc.):

1. **Distinct `scheduled_start_at` per campaign.** Never leave two sequential campaigns at the same timestamp.
2. **If you ever need to unpause multiple campaigns at once, stagger them by ≥1 hour.** Otherwise they all resume at the same moment and fire in parallel — the recipient's inbox sees 3 emails arrive at once.
3. **Before resuming, verify `scheduled_start_at` is a FUTURE timestamp.** If it's in the past, the campaign fires immediately on resume. Use the PATCH `/schedule` endpoint to push forward if needed.
4. **After unpause, watch the next 2 watchdog ticks** (~10 min) before walking away.

---

## 10. Completion report (≥25 recipients)

For campaigns with ≥25 sent recipients, Gaspar auto-generates a PDF report at completion: clones a master Sheet template → populates Summary + Recipients tabs → exports PDF → attaches to the completion email sent to `reply_to`.

**If the report doesn't generate, check:**
- Campaign reached ≥25 sent (not just ≥25 total recipients — bounces don't count)
- `completed_at` is set on the campaign object
- The connected sender account has Drive/Sheets scopes (Gaspar uses these to clone the template)

---

## 11. Quick reference — commands you'll actually run

```bash
# Provision once and cache
export GASPAR_API_KEY=...

# List all campaigns with headline stats
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/campaigns" | \
  jq '.campaigns[] | {name, status, sent_count, total_recipients, human_opens, bounce_count}'

# Get a single campaign's full config + stats
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/campaigns/$ID" | \
  jq '{name: .campaign.name, sched: .campaign.scheduled_start_at, stats: .stats}'

# Inspect recipient-level events
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/campaigns/$ID/recipients?limit=300" | \
  jq '.recipients | group_by(.last_event_type) | map({event: .[0].last_event_type, count: length})'

# Reschedule safely
NEW_MS=$(TZ=UTC date -d "2026-06-08 13:30" +%s%3N)
curl -sS -X PATCH \
  -H "Authorization: Bearer $GASPAR_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"scheduled_start_at\": $NEW_MS}" \
  "https://api.hidagama.com/api/gaspar/campaigns/$ID/schedule"

# Pause / resume
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/campaigns/$ID/pause"
curl -sS -X POST -H "Authorization: Bearer $GASPAR_API_KEY" \
  "https://api.hidagama.com/api/gaspar/campaigns/$ID/resume"
```

---

## 12. When to invoke this skill

1. Creating a new campaign (always)
2. Scheduling or rescheduling (section 2 + section 9)
3. Pausing or unpausing (section 6)
4. Debugging "0 clicks" symptoms (section 7)
5. Designing the HTML body (section 5)
6. Investigating watchdog auto-pauses (section 6)
7. Anyone asks "why didn't this email send" — work the checklist (section 3) in reverse
8. Onboarding a new operator who hasn't run a Gaspar campaign before — read sections 1 → 11

If the action isn't covered here and you don't know what to do, **stop and ask** — don't experiment in production. Recipient lists are unrecoverable.

---

## 13. Pattern summary — five mistakes worth memorizing

1. **Local-time / UTC confusion when setting `scheduled_start_at`.** → Always echo both timezones before save (section 2).
2. **Watchdog cascade-pauses a sequence.** A spike in bounces on the first campaign auto-pauses it; when the operator unpauses without addressing the bounced rows, the next watchdog tick re-pauses. → Transition bounced → suppressed FIRST (section 6).
3. **Multiple paused campaigns resume simultaneously, firing in parallel.** Recipient inbox sees 3-4 emails arrive at once. → Stagger unpauses ≥1 hour (section 9).
4. **172 humans opened, dashboard says 0 clicks.** Wrapper limitation, not low intent. → UTM every CTA, verify with analytics layer (section 7).
5. **Sending the 4th email in 5 days to the same list.** Open rate degrades, unsubscribes spike. → Frequency check (section 8). Acknowledge the over-send in the body if you must send anyway.

---

## License

MIT
