# Security policy

Gaspar sends email on behalf of its users from their own connected mailboxes, so we treat anything touching authentication, token handling, or another tenant's data as high severity. Reports are welcome, including from people who are not customers.

## Reporting a vulnerability

Email **hello@hidagama.com** with `SECURITY` in the subject line. Please do not open a public issue, a pull request, or a discussion thread for a suspected vulnerability — a public report exposes other users before there is a fix.

Useful things to include, as far as you have them:

1. What the issue lets an attacker do, stated plainly.
2. The steps to reproduce it, including the exact request or URL.
3. Anything needed to tell a real finding from a false positive — a response body, a screenshot, a short script.
4. Whether you have shared it with anyone else, or intend to.

You do not need a polished write-up. A rough report of something real is worth more than a tidy report of something theoretical.

## What to expect

We aim to acknowledge a report within three business days and to tell you whether we can reproduce it. If it is valid we will keep you updated through to the fix, and we are happy to credit you by name once it is resolved — or to leave you unnamed if you prefer.

We are a small team. If you have not heard back within a week, send a follow-up rather than assuming the report was dismissed.

## Scope

In scope:

- The hosted Gaspar service — `api.hidagama.com` and `gaspar.hidagama.com`.
- This repository and [`hidagama/gaspar-mcp`](https://github.com/hidagama/gaspar-mcp).

Out of scope:

- Denial of service, volumetric or resource-exhaustion testing.
- Social engineering of our staff, users, or vendors, and physical attacks.
- Automated scanner output with no demonstrated impact.
- Vulnerabilities in third-party services we depend on. Report those to the vendor; tell us if the exposure reaches Gaspar users.
- Missing hardening headers or best-practice deviations with no exploitable consequence. Still tell us, but expect them to be triaged low.

## Testing safely

Please test only against an account you control. Do not access, modify, or retain another user's campaigns, recipients, mailbox contents, or connected-account tokens — if you stumble into someone else's data, stop and tell us what you saw so we can assess the exposure.

Do not use a vulnerability to send email to anyone who has not agreed to receive it. Gaspar is a sending platform, and unsolicited mail causes real harm to the recipients and to the reputation of the domains involved.

Research conducted in good faith under this policy is welcome, and we will not pursue action over it. Deliberately accessing other people's data, disrupting the service, or extracting information beyond what a finding requires falls outside that.

## API keys

If you believe a Gaspar API key has leaked — yours or one you found in the wild — revoke it from Settings in the dashboard and tell us. Keys are stored hashed and shown exactly once at creation, so we cannot recover the value, but we can help confirm what it was used for.
