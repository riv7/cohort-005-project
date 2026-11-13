# Research: third-party notification providers vs the in-house default

Ticket: riv7/cohort-005-project#2 (part of map #1). Gathered 2026-11-13.
Findings and facts only. No recommendation; the human decides.

Baseline (from map #1): `notifications` table in the existing SQLite/Drizzle DB, a single `notify` service, polling (~30s unread-count route), in-app feed only, one row per recipient.

Providers compared: Novu, Knock, Courier. Only first-party pages (pricing, docs, repo) were used. Pricing pages are marketing pages and change; re-check before relying on figures.

## Summary table

| Topic | Novu | Knock | Courier |
| --- | --- | --- | --- |
| Free tier | 10,000 workflow runs/mo, 5 workflows, 3 team members, 24h activity-feed retention | 10,000 messages/mo, 500 guide recipients/mo, 30-day log retention | 10,000 sends/mo |
| First paid tier | Pro from $30/mo (30,000+ runs, 20 workflows, 7-day retention, extra runs $1.20 per 1K) | Starter $250/mo (50,000 messages, then $0.005 each) | Business $0.005 per send (pay as you go) |
| Self-hosting | Yes (community edition, MIT core, some features cloud-only) | Not mentioned on pricing page; no self-host offering found in sources read | Not mentioned in sources read |
| Recipient key | App-defined `subscriberId` | App-defined user id string | App-defined `user_id` string |
| In-app feed | `<Inbox />` component, in-app workflow step | Feed channel, websocket, read/seen/archive tracking, SDKs | Inbox channel, `<CourierInbox />` |
| Add email later | Add an Email step + provider integration to the workflow | Add email channel + email step; commit/promote | Add `email` to routing channels + email provider |

## Pricing and free tier

- Novu: Free $0 (10,000 workflow runs/month, 5 workflows, 2 environments, 3 team members, "24 hours" activity feed retention, all channels). Pro from $30/month, Team from $250/month (90-day retention, RBAC), Enterprise custom (self-hosted deployment options listed there). Source: https://novu.co/pricing
- Knock: Free $0 (10,000 messages/month; 500 guide recipients). Starter $250/month with 50,000 messages then $0.005 each. A message is "a single message delivered to a single user on a single channel"; failed/bounced messages are excluded from billing. Enterprise custom. Source: https://knock.app/pricing
- Courier: Developer (free) 10,000 sends/month. Business $0.005 per send. One send = one template to one recipient (multi-channel counts once). Test and custom-environment sends count toward limits. No per-seat or channel add-on fees. Source: https://www.courier.com/pricing
- The billing unit differs per vendor (workflow run vs message vs send), so the free limits are not directly comparable. Knock and Courier define their unit per recipient explicitly; Novu's page says "workflow runs" only.

## Self-hosting

- Novu: self-hosting is documented. Single-VM: 4 vCPU / 8 GB RAM for services, MongoDB (2 GB RAM, 20 GB disk), Redis (2 GB RAM, AOF enabled), 10 GB min S3 storage. The recommended multi-VM setup is much larger (MongoDB Atlas M20+, 2 Redis clusters). The docs say self-hosting does not support social login and that certain cloud-exclusive features (docs list Inbox features, email activity tracking, SSO, RBAC, compliance certifications) are unavailable. The page does not say exactly which Inbox features. Source: https://docs.novu.co/community/self-hosting-novu/overview
- Novu licence: repo README states an open-core model; core is MIT, the `enterprise` folder is under a separate commercial licence. Source: https://github.com/novuhq/novu
- Knock and Courier: the pricing pages do not mention self-hosting. No self-host documentation was found in the sources read; not confirmed either way.
- Relevance to baseline: self-hosted Novu adds MongoDB and Redis, which conflicts with the single-SQLite-file premise in map #1 (it references ADR 0006; the ADR was not re-read for this note).

## How users and recipients map

- Novu: each subscriber has an app-defined unique `subscriberId`. Profile holds user data (email, phone, firstName, locale), arbitrary custom `data` JSON, and channel credentials. Subscribers can be created just-in-time on workflow trigger, ahead of time, or by bulk import (up to 500). Source: https://docs.novu.co/platform/concepts/subscribers
- Knock: recipients identified by a string user id (or object reference). Stored: identifiers, properties (name, email, phone...), preferences, channel data. Identified ahead of time or lazily via inline identification at trigger. Source: https://docs.knock.app/concepts/recipients
- Courier: `user_id` is any string from your own system. The Inbox channel needs no profile field; authenticating a user in the Inbox component registers them. Other channels need a profile (`POST /profiles/{user_id}`). Source: https://www.courier.com/docs/in-app/send-to-the-inbox , https://www.courier.com/docs/recipients/overview
- In all three, the Cadence user id can be the key. Cadence roles (Student, Instructor, Platform Admin, Team Admin) are not a vendor concept; the app would resolve recipients itself and call the vendor per recipient (vendor topics/fan-out not evaluated).

## Data duplication

- All three store recipient identity/profile data and message state (feed items, read/seen/archived) on the vendor side, so recipient ids and any profile fields sent (e.g. email, name) are duplicated outside the Cadence DB.
- Knock keeps a per-user message list for the feed ("per-user list of in-app messages"). Source: https://docs.knock.app/in-app-ui/overview
- Vendor log retention differs by plan (Novu 24h free / 7 days Pro / 90 days Team; Knock 30 days on Free/Starter). Whether feed items expire on the same schedule is not stated on the pages read.
- Click-through targets (course, question, refund request) would be passed to the vendor as payload data.

## Delivery and client integration

- Novu: `<Inbox />` is the in-app delivery endpoint; messages are routed to the subscriber's inbox in real time. Source: https://docs.novu.co/framework/in-app-channel
- Knock: feed with "websocket support, no infrastructure setup required", badge counts, read/seen/archive tracking; SDKs for React, JS, React Native, Swift, Kotlin, Flutter. Source: https://docs.knock.app/in-app-ui/overview
- Courier: client `courier.shared.signIn({ userId, jwt })` then `<CourierInbox />`; server sends with `POST https://api.courier.com/send` and `routing.channels: ["inbox"]`. Source: https://www.courier.com/docs/in-app/send-to-the-inbox
- Map #1 settled on polling with swappable transport and "mark all read, no delete". Per-vendor support for mark-all-read / no-delete was not verified (Knock lists "archive").
- Courier's browser client authenticates with a JWT, implying Cadence would add a token-issuing route; equivalents for the other vendors were not verified.

## Vendor lock-in

- Notification history and feed state live in the vendor. No vendor export guarantees were found in the pages read.
- Workflow/template definitions live in the vendor (Novu workflows, Knock workflows with a CLI `workflow pull/push`, Courier templates). Novu also has a code-first Framework (`step.inApp`, `step.email`) where workflows are defined in application code. Sources: https://docs.novu.co/framework/introduction , https://docs.knock.app/tutorials/sender-domain-migration
- Novu is the only one of the three where the sources show a self-host exit path (MIT core; some features cloud-only).
- Front-end components/SDKs are tied to each vendor's API.

## Cost of adding email later

- Novu: add an Email step and configure a provider integration (SendGrid, Mailgun, SMTP, etc.). Docs show in-app, then delay, then email-if-unread as a supported pattern. Sources: https://docs.novu.co/platform/workflow/add-and-configure-steps , https://docs.novu.co/platform/integrations/email/adding-email
- Knock: create an email channel (built-in, or Amazon SES, Mailgun, Postmark, Resend, SendGrid, SMTP, etc.), add an email step to the workflow, then commit and promote. Source: https://docs.knock.app/mapi-reference/workflows/schemas/workflow_email_step
- Courier: add `email` to `routing.channels` with an email provider configured; email needs a profile with an address (Inbox does not). Source: https://www.courier.com/docs/integrations/email/overview
- In-house baseline: not researched (no external source applies). Email would need a provider/SMTP integration, a delivery step in `notify`, and delivery-status handling; the recipient email already exists on the Cadence user.
- Pricing effect (from the pricing pages): Knock bills per message per channel; Courier bills per send regardless of channel count; Novu bills per workflow run.

## Gaps and unverified points

- Knock and Courier self-hosting: absent from sources read; not confirmed as unavailable.
- Which Novu "Inbox features" are cloud-only when self-hosted.
- Whether feed items expire with log retention on each plan.
- Mark-all-read and no-delete support per vendor.
- Free-tier fit for Cadence volume (map #1 has no volume estimate).
- Some docs URLs (Novu self-hosted-deployment, two Courier docs paths) returned 404 on direct fetch; figures come from pages that loaded or from search-result excerpts of the cited official docs.
