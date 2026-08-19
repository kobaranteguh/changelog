# WasapFlow Bridge — Changelog

**Current API Version:** 2.7.0
**Last Updated:** 20 August 2026

This changelog covers **two** kinds of change:

1. **Bridge API changes** — new endpoints, new webhook events, response shape changes.
2. **Meta platform changes** — pricing, policy, and behaviour changes that Meta applies to every WhatsApp Business Platform integration, including yours.

You are responsible for the second category as much as we are. Bridge is a transparent relay: when Meta changes a rate or a rule, that change reaches your clients through us, but the commercial impact lands on you. This page is how we tell you ahead of time.

---

## How you are notified

You do not need to poll this page. **Every Bridge API response carries notice headers:**

| Header | Meaning |
|---|---|
| `X-Bridge-Api-Version` | Current Bridge API version |
| `X-Bridge-Changelog` | URL of this page |
| `X-Bridge-Notice` | Comma-separated IDs of active notices (absent when there are none) |
| `X-Bridge-Notice-Level` | Highest severity among active notices: `info`, `action_required`, or `breaking` |

**Every webhook** also carries the same information in a `meta` object:

```json
{
  "event": "message.delivered",
  "waba_id": "...",
  "phone_number_id": "...",
  "timestamp": 1786160000,
  "data": { "...": "..." },
  "meta": {
    "api_version": "2.2.0",
    "changelog": "https://partner.wasapflow.com/bridge/docs?tab=changelog",
    "notices": [
      {
        "id": "meta-service-message-billing-2026-10-01",
        "severity": "action_required",
        "effective": "2026-10-01"
      }
    ]
  }
}
```

**Recommended handling:** log `X-Bridge-Notice-Level` and alert your team when it is `action_required` or `breaking`. Notice IDs are stable forever, so you can suppress ones you have already acted on.

> The `meta` object is inside the signed body. If you verify webhook signatures, no change is needed — the signature is computed over the complete body as sent.

Docs mirror on GitHub: **[github.com/kobaranteguh](https://github.com/kobaranteguh)** — `api`, `guide`, and `changelog` repos are updated in the same pass as this dashboard, so both are always in sync.

---

## Active notices

### `meta-service-message-billing-2026-10-01` — action required

**Effective 1 October 2026.** Meta will begin charging for two message types that are free today.

| What | Free since | Charged from |
|---|---|---|
| Service messages — any non-template reply inside the 24-hour customer service window | 1 Nov 2024 | 1 Oct 2026 |
| Utility templates delivered inside an open customer service window | 1 Jul 2025 | 1 Oct 2026 |

Key facts, confirmed against Meta's documentation:

- **Nothing about how you send changes.** Service messages remain free-form and still require **no template and no Meta pre-approval**. The 24-hour window rules are unchanged. This is purely a billing change.
- Service messages are charged **per message at the same rate as utility and authentication messages**, set by market.
- **There are no volume tiers for service messages.** Utility and authentication keep their tiers; service does not. Sending more does not lower the rate.
- The **72-hour free entry point window** (Click to WhatsApp Ads, Page CTA) stays free.
- **One charge per message.** A non-template message containing promotional content does *not* additionally incur a marketing charge.
- Meta will publish the rates that take effect 1 October **by 1 September 2026**.

**What you should do now:** estimate your exposure. Every `message.sent` / `message.delivered` webhook now includes a `pricing` object (see 2.2.0 below). Messages with `pricing.type` of `free_customer_service` are exactly the messages that become billable on 1 October. Count them for a month and you have your projection; multiply by the rate when Meta publishes it.

Meta reference: [Upcoming pricing updates](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing/non-template-messages)

### `meta-business-agent-token-billing-2026-08-01` — info

**Effective 1 August 2026.** Meta launched its own AI agent, **Meta Business Agent**, as a new message category alongside service. It is billed per token at USD $2.00 per 1M tokens (roughly 4–5 US cents per message), as a single charge covering both AI processing and message delivery. Meta will not deliver Business Agent messages unless a credit line is attached.

**This does not change anything about messages sent through Bridge.** Meta's own documentation is explicit that any non-template message not powered by Meta Business Agent — including messages from a human agent *or a third-party AI solution* — is a **service** message. Your traffic through Bridge is service traffic.

It matters to you for two reasons:

- If one of your clients enables Meta Business Agent directly on their number, your app stops being the active handler for those conversations. See **Standby webhooks** under Planned below.
- Meta now publishes a direct cost comparison between Business Agent and third-party AI solutions. If you resell an AI product, this is a competitor with published pricing.

Meta reference: [Conversations 2026 announcement](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing/non-template-messages)

---

## 2.7.0 — 20 August 2026

All of this came from one partner asking six questions. Nothing here is new
capability — it is capability that already worked and was never written down,
plus one gap we found while checking.

### Documented — catalog and product messages

`POST /messages/interactive` sends your `interactive` object to Meta unchanged,
so `product`, `product_list` and `catalog_message` have always worked. Nobody
knew, because `catalog_id` and `product_retailer_id` appeared nowhere in these
docs. They do now.

`catalog_id` and `product_retailer_id` come from you. We do not store client
catalogues, inject ids, or validate them.

There is no catalogue *management* API and none is planned — no `/catalog`, no
`/products`. Bridge sends messages that reference a catalogue and relays the
orders that come back. To read products or stock, use Meta's Catalog API directly.

### Documented — `raw.message` carries everything

The flat fields on `message.received` are a convenience layer. `data.raw.message`
is Meta's own message object with nothing removed, which is where these live:

| | |
|---|---|
| Cart / order | `raw.message.order` |
| Product enquiry | `raw.message.interactive` |
| Reply context | `raw.message.context` |

**Watch the trap:** for `order` and `interactive`, the flat `text` is `null`.
Integrations that only read `text` see a cart as an empty message.

### Documented — the two media paths are opposites

`POST /messages/media` with `media.link` is a **passthrough**: Meta fetches your
URL, and every limit is Meta's. `POST /media/upload` is the reverse — **we**
fetch the file and upload it to Meta, returning a reusable `media_id`. Similar
names, opposite mechanics. Use the second for anything sent repeatedly.

### Changed — media endpoints now obey the rate limit, and downloads are logged

`GET /media/{id}` and `POST /media/upload` bypassed the rate limiter completely.
They now count against your account's `rate_limit_per_sec` like every other
endpoint.

**This closes a gap rather than tightening a limit.** It is the same limit
already applied everywhere else (200/s by default), and real usage is nowhere
near it — `/media/upload` averages under 6 calls a day across all partners.

Media downloads were also not logged at all, so the traffic was invisible to us.
They now appear as `/media/download` in your usage. We added the logging before
the limit deliberately: setting a threshold on traffic you cannot measure is a
guess.

---

## 2.6.1 — 17 August 2026

Three fixes, all from one partner's integration questions. Every one of them was
something we had got wrong rather than something they had.

### Fixed — the popup now tells you when onboarding is cancelled or fails

The connect popup only ever posted `WASAPFLOW_CONNECT_SUCCESS`. If the customer
dismissed Meta's dialog, or registration was rejected, the popup showed a message
to the customer and told your window **nothing**.

So an integration waiting on an event stayed in "connecting…" until the customer
closed the popup by hand, with no way to know whether they had given up or it had
broken. Cancel and error handlers written against earlier versions never fired.

Two new messages, both carrying `state` like the success one:

| `type` | Extra fields |
|---|---|
| `WASAPFLOW_CONNECT_CANCEL` | `reason` |
| `WASAPFLOW_CONNECT_ERROR` | `message`, `code` |

Keep any `popup.closed` polling you added to work around this — a customer can
still close the window before reaching either path.

### Fixed — onboarding no longer dies at the last step

The connect token was checked twice: when the page opened, and again when
registration completed. A customer who opened the popup at minute 29 and worked
through Meta's flow — portfolio, number, verification code — arrived at the final
call after expiry and was rejected, losing everything they had just done. With
the bug above, that rejection was also silent.

Completion is now allowed up to 30 minutes past expiry. Opening is unchanged and
still strict: that is the moment a leaked link could be reused, and it is the one
worth guarding. Meta's `code` is short-lived and single-use, so accepting it a
little late costs nothing.

### Fixed — rotating a partner key now revokes unfinished connect sessions

Connect sessions reference the partner, not the key, so rotating a key left every
unfinished session working for up to 30 more minutes. Since a rotation almost
always means the old key is considered exposed, that defeated the point.

Rotation now deletes unconsumed, unexpired sessions and reports how many. WABAs
already connected are untouched — each holds its own Meta token, so messaging
does not stop.

---

## 2.6.0 — 17 August 2026

### Security — stop putting your API key in a browser URL

Until now the way to start onboarding was:

```
https://officialapi.wasapflow.com/bridge/connect?partner_key=wf_live_…
```

That is your **full API credential**, the same key that authorises every send,
every WABA and every client you manage — sitting in a URL. From there it reaches
your client's address bar, their browser history, the `Referer` header their
browser sends to other hosts, and our own nginx access logs in plain text.

We found it in our logs while investigating an unrelated onboarding failure. No
misuse has been detected, and no partner did anything wrong — this was our design.

**The new way is server-to-server:**

```http
POST /connect/session
x-partner-key: wf_xxx

{ "display_name": "Client Name", "state": "your-own-client-id-123" }
```

```json
{
  "success": true,
  "connect_url": "https://officialapi.wasapflow.com/bridge/connect?token=93d29ead…",
  "expires_in": 1800
}
```

Send your client to `connect_url`. The token is scoped to that one onboarding and
expires in 30 minutes; it cannot be used to call any other endpoint.

**The old form still works and has no removal date** — nothing breaks today. But
please migrate, and once you have, ask us to rotate the key: a credential that
has been through a browser should be treated as exposed.

**Correction (2.6.1):** this originally said to rotate the key "in the partner
portal". There is no self-serve rotation — it is a support request. Sorry for
sending anyone looking for a button that does not exist.

### Added — `state` is now returned to you

`POST /connect/session` accepts an optional `state` (max 255 characters, opaque to
us). It comes back untouched when the connection completes, both in the
`postMessage` payload and in the `/clients/register-from-code` response body, so
you can match the finished connection to your own client record.

Before 2.6.0 the connect page accepted a `state` query parameter and discarded it
without telling anyone. If you passed one and it never came back, that is why.

---

## 2.5.0 — 16 August 2026

### Fixed — a BSUID in `to` was sent to Meta as a phone number

**If you are on 2.4.0 and reply to username-only customers, this affects you and you should upgrade.**

2.4.0 added `recipient` for BSUID sends. It did not stop `to` from being forwarded to Meta's *phone number* field, and nothing checked whether the value in `to` was actually a phone number.

So a partner who put a BSUID in `to` — the most natural thing to do, since our own inbound webhook carries `bsuid` and no number at all — got this:

```
we sent      "MY.1777397056778201"
Meta received "1777397056778201"
```

The market prefix was dropped and the remaining digits reached Meta as a phone number belonging to nobody. Meta answered `131026 Message undeliverable` — **after** we had already returned `2xx`. The send looked successful and the message never arrived.

Click-to-WhatsApp ads are where this bites hardest: those leads frequently have a username and no phone number, so every one of them was unreachable through the API while the same reply sent from the WhatsApp Business app went through normally.

**From 2.5.0 all three field names work, and a BSUID is never treated as a phone number:**

```json
{ "to":        "MY.1777397056778201", "text": "..." }
{ "user_id":   "MY.1777397056778201", "text": "..." }
{ "recipient": "MY.1777397056778201", "text": "..." }
```

`user_id` is new in 2.5.0, for integrations that prefer an explicit field over us detecting the shape of `to`. `recipient` still works.

**Sending to a phone number is unchanged, byte for byte.** When a phone number and a BSUID are both supplied, the phone number wins — Meta's own precedence rule.

### Added — the wamid is now also returned in Meta's own shape

Send responses have always carried the Meta message id as `message_id`. Partners arriving from Meta's Cloud API documentation look for `messages[0].id` instead, find nothing, and conclude no id is returned. Both keys are now present and hold the same value:

```json
{
  "success": true,
  "message_id": "wamid.HBg...",
  "messages": [ { "id": "wamid.HBg..." } ]
}
```

Nothing was removed. If you already read `message_id`, keep reading it.

Store the id: every `message.sent` / `message.delivered` / `message.read` / `message.failed` callback is keyed on it. Without it a `message.failed` — the only way you learn a send did not arrive — cannot be matched to anything.

### Added — authentication templates to a BSUID now fail loudly

Meta refuses one-tap, zero-tap and copy-code authentication templates addressed to a BSUID (error `131062`), and this is permanent: OTP always needs a real phone number. That used to surface as a generic `META_ERROR`. It now returns:

```json
HTTP 400
{
  "success": false,
  "error": {
    "code": "AUTH_TEMPLATE_NEEDS_PHONE",
    "meta_code": 131062,
    "retryable": false
  }
}
```

Do not retry it. Collect a phone number instead.

---

## 2.4.0 — 11 August 2026

### Added — send to a BSUID with `recipient`

2.3.0 told you Meta had made BSUID support mandatory, and admitted Bridge could not yet send to one. That gap is closed.

Every send endpoint now accepts **`recipient`** (a BSUID) as an alternative to **`to`** (a phone number):

```json
POST /messages/send
{
  "recipient": "MY.2026206508304015",
  "text": "Terima kasih! Order anda dalam proses."
}
```

Applies to `/messages/send`, `/messages/template`, `/messages/media`, `/messages/interactive`, `/messages/location`, `/messages/reaction`, and broadcasts.

**Nothing changes for existing integrations.** When both are supplied, `to` wins — that is Meta's own precedence rule, not ours. Code that sends `to` behaves exactly as before; you only reach for `recipient` when you have no phone number to use.

If neither is supplied you now get a clearer error:

```json
{ "success": false, "error": { "code": "MISSING_FIELDS",
  "message": "Either \"to\" (phone number) or \"recipient\" (BSUID) is required" } }
```

**Broadcasts** accept the same on each contact. A contact may stay a bare phone string, or become an object:

```json
{ "contacts": [
    "60123456789",
    { "recipient": "MY.2026206508304015", "params": ["Aiman"] }
] }
```

A contact carrying neither is skipped rather than sent with an empty recipient.

**Verified against Meta, not just documented.** We sent a live message to a BSUID on Graph API v24.0 and Meta accepted it:

```json
{ "contacts": [{ "input": "MY.2026206508304015", "user_id": "MY.2026206508304015" }],
  "messages": [{ "id": "wamid.HBgTTVkuMjAyNjIwNjUwODMwNDAxNRUU..." }] }
```

**One response-shape difference worth knowing:** when you send to a BSUID, Meta's response carries `contacts[0].user_id` and **no `wa_id`**. Code that reads `wa_id` from a send response to record the recipient will get `undefined`. Read `user_id` when you sent to a `recipient`.

### What Meta still does not allow

BSUID recipients are **rejected** for:

- One-tap and zero-tap authentication templates
- Copy-code authentication templates

Meta returns error **`131062`** — *"Business-scoped User ID (BSUID) recipients are not supported for this message"*.

This is permanent, not a rollout gap. **OTP flows will always need a real phone number.** If you verify users by sending a code over WhatsApp, keep collecting a phone number for that path — a BSUID cannot carry it.

### Still on the roadmap

- **`user_id_update` webhook** — Meta regenerates a BSUID when a user changes their phone number. Until we relay it, a stored BSUID can go stale.
- **Parent BSUIDs** (`US.ENT.…`) — relevant only to multi-portfolio businesses.

---

## 2.3.0 — 11 August 2026

**No endpoint changed in this release.** It documents a requirement Meta has already made mandatory, and a data-model trap that will silently corrupt delivery data in most integrations built on this platform — including ours, until we fixed it.

### First, what a BSUID actually is

If you only read one part of this release, read this.

WhatsApp is rolling out **usernames**. A customer can now be reachable as `@aiman` instead of by phone number, and they can hide their phone number from businesses they talk to. Meta still has to identify that person to you somehow — so it gives you a **BSUID**, a stable id scoped to your business:

```
MY.2026206508304015
│  └─ up to 128 alphanumeric characters
└──── ISO country code, then a dot
```

Multi-portfolio businesses also see a parent form with `ENT` inserted: `US.ENT.11815799212886844830`.

Four things worth knowing:

1. **It is stable.** The same customer keeps the same BSUID with you, even if they change their phone number. That makes it a *better* primary key than a phone number.
2. **It is scoped to your business.** The same person has a different BSUID with a different business. You cannot look someone up across businesses with it, and you cannot message a BSUID from a phone number belonging to another portfolio.
3. **You can send to it.** Since July 2026 the Messages API accepts `recipient: "<BSUID>"` in place of `to: "<phone>"`. We tested this against a live number on Graph API v24.0 and Meta accepted it — no version bump needed.
4. **It is not a phone number.** This is the part that causes damage. A courier cannot ring it, WooCommerce cannot use it as `billing.phone`, and — critically — it will survive a naive phone normaliser and come out the other side looking like a real number.

**When do you get one instead of a phone number?** Bridge already sends `bsuid` alongside `from` on every inbound message, so today you get both. You start receiving the BSUID *without* a phone number when a customer has adopted a username **and** there has been no interaction with your number in the last 30 days. Per Meta, `wa_id`, `from` and `recipient_id` are then *"omitted entirely"*.

So the change arrives gradually, customer by customer — not on a single date.

### Displaying a customer who has no phone number

Most inboxes fall back to showing the phone number when a contact has no name. That fallback breaks in a way users notice immediately.

If your code does something like `contact.name || ('+' + contact.phone)`, a BSUID contact renders as:

```
+MY.2026206508304015
```

Support staff read that as a broken system and raise a ticket. It is worth checking how often you would hit this: in our own fleet, **45% of contacts have no profile name**, so this is not an edge case.

A display order that holds up:

```
1. profile name          (Meta still sends contacts[0].profile.name)
2. collected phone number (once your order flow has asked for it)
3. "WhatsApp customer · 304015"   <- last 6 characters of the BSUID
```

Putting the collected phone second matters: as soon as your order flow captures a real number, that contact stops showing an id and starts showing something a human recognises.

**Use 6 characters, not 4.** We checked this against real data before deciding. In our largest workspace, 924 contacts with BSUIDs already produce **41 collisions** on the last 4 characters — different people rendering identically in the list. Extrapolated to that workspace's 2,422 nameless contacts, 4 characters would give roughly 293 collisions; 6 characters brings it to about 3.

Also show the **full BSUID in the contact detail view**, not just the fragment. The fragment is for scanning a list; the full value is for when someone needs to be sure two rows are different people.

### Action required — stop treating the WhatsApp number as the customer's phone number

Meta states plainly: *"Supporting business-scoped user IDs (BSUID) is **required** for all partners and directly-integrated businesses on the WhatsApp Business Platform."*

Bridge has been forwarding `bsuid` on inbound messages and `recipient_bsuid` on status events since 2.0. What changes now is what you must **do** with it.

**The problem.** Once a customer adopts a WhatsApp username, Meta may omit their phone number from the webhook entirely. Per Meta's docs, `wa_id`, `from`, and `recipient_id` are *"omitted entirely"* when the user has a username and there has been no interaction in the last 30 days. You receive a BSUID like `MY.2035200694071263` and nothing else.

A BSUID is a valid identifier for **sending WhatsApp messages**. It is not a phone number. A courier cannot call it, and WooCommerce cannot use it as `billing.phone`.

**The trap.** Almost every integration has a phone normaliser shaped like this:

```javascript
let n = phone.replace(/\D/g, '');
if (n.startsWith('0')) n = '6' + n;
if (!n.startsWith('60')) n = '60' + n;
```

Feed it a BSUID and you get **`602035200694071263`** — a value that looks like a Malaysian phone number, passes naive validation, and flows all the way to your courier. It does not throw. Nothing appears in your logs. Your seller ships to a customer nobody can reach.

Check your own normaliser now. If it strips non-digits before validating, you have this bug.

### What you must do

**1. Split one field into two.** Most schemas store a single `phone` on the contact and use it for everything. Separate the roles:

| Role | Contents | Used for |
|---|---|---|
| Canonical id | Phone **or** BSUID | Sending WhatsApp, matching a contact back |
| Reachable phone | Always a real phone, may be empty | Courier, invoices, voice calls |

**2. Reject BSUID in phone validation — do not "clean" it.** Return null and treat it as *we have no number*. No number beats a fake one.

```javascript
// A BSUID is: 2-letter country code, dot, alphanumerics.
// Parent BSUIDs insert ENT: US.ENT.11815799212886844830
const BSUID = /^(?:whatsapp:)?[A-Z]{2}\.(?:ENT\.)?[A-Za-z0-9]{1,128}$/;

function toPhone(value) {
    if (!value) return null;
    if (BSUID.test(String(value).trim())) return null;  // never normalise a BSUID
    if (/[A-Za-z]/.test(value)) return null;            // letters mean it is not a phone
    // ...your existing normalisation...
}
```

**3. Ask the customer for a phone number in your order flow.** If your flow collects name, address and postcode but not a phone — because the WhatsApp number was always assumed to be the phone — it now collects an order that cannot be delivered.

Keep the friction low. When you already hold a plausible number, ask them to **confirm** it rather than retype it:

> "Boleh sahkan no 012-345 6789 ni untuk kurier WhatsApp atau call ya?"

Only when you genuinely have nothing should you ask them to type it in:

> "Boleh bagi no telefon untuk kurier WhatsApp atau call masa hantar ya?"

Worth doing even before usernames arrive: customers routinely order from a work WhatsApp, or on behalf of someone else. The delivery number is frequently not the WhatsApp number, and you have probably been shipping with the wrong one already.

**4. If you push to WooCommerce or an ERP,** send the real phone as `billing.phone` and keep the canonical id in a separate meta field for matching webhooks back to the contact. Leave `billing.phone` **empty** rather than filling it with an id — an empty field prompts a human to ask; a fake number sends a courier to a dead end.

**5. Meta's `REQUEST_CONTACT_INFO` template button** (available since early July 2026) is the supported way to ask a customer to share their phone number. Parse the resulting contacts webhook and read the phone from the shared vCard.

### Also required by Meta, not yet in Bridge

These are on our roadmap; you may need them sooner depending on your integration:

- **Sending to a BSUID** — since July 2026 the Messages API accepts `recipient` (BSUID) in place of `to` (phone). Bridge send endpoints currently accept `to` only. Until we ship this, you cannot reply to a customer who has no phone number on file.
- **`user_id_update` webhook** — Meta regenerates a BSUID when the user changes their phone number. Miss it and your stored id goes stale.
- **Parent BSUIDs** (`US.ENT.…`) — only relevant for multi-portfolio businesses.

### Timing

Nothing is broken today. Across our own fleet we have recorded **zero** BSUID-only messages so far, and usernames are not yet live in Malaysia. This is preventive work — and it is far cheaper now than during a support queue.

Meta reference: [Business-scoped user IDs](https://developers.facebook.com/documentation/business-messaging/whatsapp/business-scoped-user-ids)

---

## 2.2.0 — 8 August 2026

### Added — `pricing` on message status webhooks

`message.sent`, `message.delivered`, `message.read`, and `message.failed` now include the per-message pricing data Meta reports, so you can attribute cost per message without a separate analytics call.

```json
{
  "event": "message.delivered",
  "data": {
    "message_id": "wamid....",
    "status": "delivered",
    "recipient": "60123456789",
    "timestamp": 1786160000,
    "pricing": {
      "billable": false,
      "pricing_model": "PMP",
      "type": "free_customer_service",
      "category": "service"
    },
    "conversation": {
      "id": "...",
      "origin": "service"
    }
  }
}
```

| Field | Values | Notes |
|---|---|---|
| `pricing.billable` | `true` / `false` | Whether Meta charges for this message |
| `pricing.pricing_model` | `PMP` | Per-message pricing |
| `pricing.type` | `regular`, `free_customer_service`, `free_entry_point` | **`free_customer_service` becomes `regular` on 1 Oct 2026** |
| `pricing.category` | `service`, `utility`, `marketing`, `authentication`, `referral_conversion` | Which rate applies |
| `conversation.origin` | same values as category | What opened the conversation |

`pricing` is `null` when Meta does not supply it (commonly on `read` receipts). Treat it as optional.

**This is additive.** Existing fields are unchanged and no field was removed.

### Added — notice headers on every API response

`X-Bridge-Api-Version`, `X-Bridge-Changelog`, `X-Bridge-Notice`, `X-Bridge-Notice-Level`. See [How you are notified](#how-you-are-notified). Headers only — response bodies are unchanged, so strict body parsers and schema validators are unaffected.

### Added — `meta` object on every webhook

Carries `api_version`, `changelog`, and active `notices`. Additive; existing fields unchanged.

### Added — standby webhooks (Meta Business Agent coexistence)

When a business enables **Meta Business Agent** on a phone number, Meta's agent and your app coexist on the same number and only one is the **active handler** at a time. While your app is the passive listener, Meta stops sending the normal `messages` field and sends `standby` instead. Bridge now relays all three standby sub-types:

| Event | Fired when |
|---|---|
| `standby.message_received` | A user sends a message while your app is **not** the active handler |
| `standby.message_echo` | Meta Business Agent sent a message on the client's behalf |
| `standby.message_status` | Delivery/read status for a Meta Business Agent message (carries `pricing`) |

Every standby payload includes `standby: true` and identifies the current handler, so you can branch cleanly:

```javascript
if (event.startsWith('standby.')) {
    // Meta Business Agent is handling this conversation.
    // Store for context — do NOT auto-reply.
    return saveContextOnly(data);
}
```

> ⚠️ **Sending a service message while in standby makes your app the active handler.** Per Meta's documentation this is the intended way to escalate to a human agent — but it also means an automated reply will silently seize control from the agent. If you run a bot, gate it on `standby !== true`.

`standby.message_status` events carry the same `pricing` object as regular status events and are recorded in the same ledger, so Business Agent traffic shows up in your cost reporting too.

#### Action required — show your client a warning banner

This is the most confusing failure mode on the platform, and it is not a failure at all.

When Meta Business Agent takes over, your client sees their automation stop replying. Nothing errors, and **nothing appears in your logs** — the messages never reach you. Your client will report "the bot is dead" and you will have nothing to point at.

**Please build a persistent warning banner** in your app header and in the inbox for the affected number. Raise it on any `standby.*` event; clear it when normal `message.received` events resume. Use a **warning** style (yellow/amber), not an error style — nothing has broken.

Suggested wording:

> 🤖 **Meta Business Agent is answering your customers — not your AI.** To take back control, turn it off in WhatsApp Manager → Account tools → Business Agent. Manual chat still works normally.

Make the raise idempotent — standby events can arrive many times a minute. Full implementation shape in the [API Reference](?tab=api#standby--meta-business-agent-is-handling-the-conversation-).

WasapFlow's own product ships this exact banner. We are asking you to mirror it so your clients get the same explanation instead of a mystery.

### Added — Coexistence offboard and reconnect

| Event | Fired when |
|---|---|
| `waba.offboarded` | A Coexistence client's WhatsApp Business App was offboarded (device change, reinstall, re-registration) |
| `waba.reconnected` | Meta finished reonboarding it automatically in the background |

Between these two events the number may not send. Treat `waba.offboarded` as a soft pause rather than a disconnection — Meta usually reconnects within a few minutes without any action from you.

### Added — `waba.pricing_tier_updated`

Fires when a WABA reaches a new **volume pricing tier** for a market–category pair, which changes the *rate* for utility and authentication messages.

```json
{
  "event": "waba.pricing_tier_updated",
  "data": {
    "waba_id": "123456789",
    "pricing_category": "UTILITY",
    "tier": "25000001:50000000",
    "region": "Malaysia",
    "effective_month": "2026-09"
  }
}
```

> Do not confuse this with `waba.tier_updated`, which is the **messaging** tier (how many unique recipients per 24 hours). This one is about price. **Service messages have no volume tiers** — this event will never fire for them.

Meta may send more than one webhook describing the same tier change; use the one with the smallest `tier_update_time`.

### Added — `message_status` on template sends

`POST /messages/template` now returns `message_status` when Meta is **pacing** the template — holding delivery back while it tests quality on a small audience first.

```json
{ "success": true, "message_id": "wamid....", "message_status": "held_for_quality_assessment" }
```

The field is **only present when pacing applies**, so its absence means normal delivery. This matters for broadcast reporting: HTTP 200 means Meta *accepted* the request, not that the message went out. When `message_status` is present, wait for the `message.sent` / `message.delivered` webhook before counting it as delivered.

---

## 2.1.0 — 5 June 2026

### Added — template lifecycle and account webhooks

- `template.status_updated` — template approved, rejected, or paused by Meta
- `template.quality_updated` — template quality rating changed
- `template.category_updated` — Meta recategorised a template
- `waba.account_updated` — WABA account-level change
- `waba.review_updated` — account review status changed
- `contact.synced` — Coexistence contact sync from the WhatsApp Business App

---

## 2.0.0 — Coexistence message sync

### Added

- `message.echo` — messages your client sends from the WhatsApp Business App directly
- `message.history` — past conversations backfilled after Coexistence onboarding

---

## 1.4.6

### Fixed — webhook fan-out to all partners

Previously, when the same WABA was registered under more than one partner, only the first partner received webhooks. All partners owning a `phone_number_id` now receive events, each signed with their own webhook secret.

---

## 1.4.5

### Changed — HTTP status codes on error responses

Before 1.4.5 the Bridge API returned `200 OK` even for failures. Failed responses now return an appropriate 4xx/5xx status. Existing code checking `body.success === false` continues to work unchanged; new code can rely on `res.ok`.

---

## Planned

Not yet implemented. Listed so you can plan; each will move to a released version with its own notice when it ships.

### Graph API version bump

Bridge currently calls Meta on **v24.0** (released 8 Oct 2025, supported until 18 Feb 2028). Meta's current version is v26.0. Everything Bridge relies on today works identically on v24.0 — we verified `pricing_analytics` returns the same data on both. We will move when a feature we need requires it, and will announce the change here first.

### Direct Send

Meta's **Direct Send** lets you send utility messages **without creating a template first** — Meta auto-generates the matching template behind the scenes, with content PII-redacted and language auto-detected. Utility became generally available on 31 July 2026; authentication is still in beta. This would remove template pre-approval from utility notification flows entirely.

### Groups

Cloud API Groups are now available to businesses with an Official Business Account. Sending uses the same Messages endpoint with `recipient_type: "group"`, and responses carry a `group_id`.

### Calling

Meta's Calling API opens a 24-hour customer service window when a **user calls** the business, not only when they message. Bridge does not surface call events today, which means a window can be open without your system knowing.

### Handover events (`messaging_handovers`)

The companion signal to standby webhooks. Where `standby.*` carries the **content** you observe while passive, `messaging_handovers` carries the **event** telling you control changed hands between Meta Business Agent and your app.

This is more reliable than inferring handover state from traffic. Inference has a real gap: if a conversation is handed back to you but the customer does not message again, you cannot tell you have control until their next message arrives.

We have subscribed to the field, but Meta's public documentation does not yet specify its payload shape for WhatsApp. Rather than ship a parser built on assumption, we log the first real payload and will write the parser from actual data. It will appear here with its event names once confirmed.

> **Worth copying:** whatever your webhook handler looks like, it is almost certainly a chain of `if (field === ...)` checks — which means any field Meta adds falls through **silently**. No error, no log, nothing. Add a fallback branch that logs unrecognised fields once, with the full payload. It costs a few lines and it is how you find out about a new Meta field in days rather than months.

---

## Meta platform calendar

Meta may change rates only on the first day of a quarter — **1 January, 1 April, 1 July, 1 October** — with minimum advance notice of 1 month for a rate card update, 3 months for a pricing model add-on, and 6 months for a pricing model change.

| Date | Change |
|---|---|
| 1 Nov 2024 | Service conversations became free |
| 1 Jul 2025 | Conversation-based pricing replaced by per-message pricing (PMP); utility templates in an open window became free |
| 1 Jan 2026 | Rate updates: India, France, Egypt, North America |
| 1 Apr 2026 | 8 new billing currencies including **MYR** (Malaysia), SGD, SAR, AED |
| 1 Jul 2026 | Rate updates: Hong Kong, Hungary, Italy, Poland, Qatar, Romania, Singapore, Spain, UK. Several markets moved off regional rates onto market-specific rate cards with market-specific volume tiers |
| 1 Aug 2026 | Meta Business Agent token billing begins |
| **1 Sep 2026** | **Meta publishes the rates effective 1 October** |
| **1 Oct 2026** | **Service messages and in-window utility templates become billable** |

Current rate cards, including MYR: [Pricing on the WhatsApp Business Platform](https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing)
