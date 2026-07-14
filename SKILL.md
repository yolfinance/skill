---
name: yolfi-webhook-integration
description: Use this skill when a merchant wants to add Yolfi crypto payment webhooks to an existing backend or create a new Yolfi webhook integration. Trigger for requests like "add Yolfi webhooks", "integrate Yolfi payments", "reuse my existing payment webhook for Yolfi", "wire Yolfi into my payment callbacks", or "add crypto payment notifications". The skill must analyze the merchant codebase, load current Yolfi docs, and choose the smallest safe integration. Required input - YOLFI_API_KEY, the merchant's Yolfi API key, may be provided in chat or discovered in the merchant's git-ignored env files.
---

# Yolfi Webhook Integration

**Required input:** `YOLFI_API_KEY` - the merchant's Yolfi API key, provided in chat.

This skill modifies merchant backends. It must preserve existing payment business logic and make the smallest safe change that integrates Yolfi.

## Minimal Integration Contract

Default to this contract unless the user explicitly asks for architecture changes:

1. Keep the merchant's existing webhook endpoint and business handlers.
2. Add Yolfi signature verification on raw body for Yolfi-signed requests.
3. Keep existing provider signature verification unchanged for provider-signed requests.
4. Dispatch into the same existing business pipeline.
5. Do not add new persistence models/tables by default.

## No-Refactor Boundary

Adapter integration is not permission to refactor the merchant's payment system.

Do not change these unless the user explicitly asks:

- payment models, provider enums, plan schemas, or entitlement logic
- checkout/paywall URLs or product routing
- existing provider business branches
- parallel plan maps inside webhook controllers
- idempotency storage or new webhook-event collections
- Google Ads, email, analytics, or other side effects

If an existing provider webhook branch depends on provider API lookups, do not split or rewrite that branch. Stop and ask the user whether to use a separate Yolfi endpoint or only wire Yolfi into an existing body-driven branch.

## Source of Truth

Always load current Yolfi docs before deciding what to implement:

1. `https://docs.yolfi.com/llms.txt` - discover available documentation.
2. `https://docs.yolfi.com/llms-full.txt` - read exact current details for webhooks, signatures, adapters, events, and payloads.

Do not copy event names, payload schemas, adapter mappings, or provider-specific details into this skill. Those details belong in the docs and may change.

If both docs sources are unreachable and no user-provided copy is available, stop and ask before editing.

## Official Toolkit - @yolfi/agent

The official agent kit (`@yolfi/agent`, github.com/yolfinance/yolfi-agent) ships an SDK, a `yolfi` CLI, and an MCP server. Prefer it over hand-rolling Yolfi API calls:

- Signature verification: reuse or mirror `verifyWebhookSignature` from `@yolfi/agent` (raw body, HMAC-SHA256, base64 digest, timing-safe compare with a length guard).
- Test webhooks: use `signWebhookPayload(payload, endpointSigningSecret)` to generate valid signed requests against a local server during verification. Never sign or verify webhooks with `YOLFI_API_KEY`.
- Provisioning: create paylinks and independent webhook endpoints through the CLI (`yolfi paylinks:create`, `yolfi webhooks:add`) or MCP (`yolfi_paylinks_create`, `yolfi_webhooks_configure`). Save each endpoint's one-time returned signing secret.
- Talivia automatic provisioning: the user supplies Talivia with `YOLFI_API_KEY`; Talivia calls the Yolfi endpoint API, Yolfi generates the endpoint signing secret, and Talivia stores that secret separately. The API key authorizes provisioning and never signs or verifies webhooks.
- Talivia analytics uses a dedicated `NONE` endpoint with `metadataFilters: { "website_id": "<websiteId>" }`.
- API schemas that are absent from the docs (paylink create body, public payment create) live in the repo's `examples/` directory.

Platform constraints to respect (re-verify against current docs before relying on them):

- A paylink's price is immutable after creation; changing the price means creating a new paylink.
- An organization can have multiple independent webhook endpoints. Each endpoint has its own URL, adapter, enabled state, delivery history, retries, and signing secret. Public adapters are `NONE` (native Yolfi), `STRIPE`, and `LEMON_SQUEEZY`.
- Pass a stable merchant-side customer/user id as `clientReferenceId` when lifecycle webhooks must resolve ownership. Native payloads expose `data.customer.clientReferenceId`; Stripe Checkout Session uses `data.object.client_reference_id`; Stripe Invoice/Subscription use `data.object.metadata.client_reference_id`; Lemon Squeezy uses `meta.custom_data.client_reference_id`.

## Core Rules

- Prefer adapting the merchant's existing webhook pipeline over creating a separate Yolfi business flow.
- Do not replace or rewrite order, subscription, email, entitlement, or fulfillment logic unless strictly necessary.
- Do not emulate provider webhook signatures. Yolfi requests must use Yolfi authentication.
- Verify signatures against the raw request body before parsing JSON or re-stringifying.
- Compare webhook signatures with a timing-safe comparison.
- Keep existing provider webhook verification unchanged for real provider requests.
- Do not send Yolfi requests through provider SDK signature verification.
- Do not route Yolfi adapter payloads into provider API calls.
- Use the merchant's existing durable idempotency/storage pattern when one exists, keyed by the current docs' event identifier when available. If none exists and duplicate handling matters, ask where it should live.
- Do not add new idempotency tables/models by default. Reuse existing merchant patterns, or ask first.
- Do not add in-memory deduplication caches for webhooks.
- Ask before editing when the safe business boundary is unclear.

## Integration Modes

Choose one mode per endpoint and business surface. Multiple endpoints may coexist (for example, `STRIPE` for billing and `NONE` for analytics), but do not mix provider-shaped and native lifecycle handling inside one route.

### Adapter Mode

Use this when current docs confirm a Yolfi adapter for the merchant's existing provider.

Allowed changes:

- Add Yolfi signature verification at webhook ingress.
- Parse the verified adapter payload.
- Pass the adapter payload to the same provider-shaped event handling path only when that path is already body-driven.
- Add Yolfi paylink IDs to the merchant's existing plan config if plan resolution requires it.

Forbidden changes:

- Do not map adapter events into native Yolfi lifecycle events.
- Do not add helper layers such as `mapYolfiLifecycleEventType`, `extractYolfiPaymentData`, or `findUserForYolfiEvent`.
- Do not duplicate or reimplement subscription, invoice, refund, analytics, or cleanup behavior for Yolfi inside the provider webhook.

### Native Mode

Use this only when no compatible provider webhook exists, or when the user explicitly chooses a separate Yolfi/native integration.

Allowed changes:

- Create or use a Yolfi-authenticated route in the app's existing route style.
- Verify Yolfi signature and call an existing reusable business function.

Forbidden changes:

- Do not place native Yolfi lifecycle handling inside an existing provider webhook endpoint.
- Do not extract large inline provider webhook branches into new abstractions unless the user explicitly asks.

Native event handling rules:

- Treat invoice-scoped events and subscription-scoped events separately.
- Do not group an invoice overdue/missed-payment event with subscription overdue/cancelled events.
- Do not revoke active subscription access from an invoice overdue/missed-payment event alone.
- Only subscription-scoped overdue/cancelled events should drive subscription deactivation, and only if that matches the merchant's existing access policy.

## Workflow

### 1. Validate Inputs

- Before asking for the key, check git-ignored env files and secret config for an existing `YOLFI_API_KEY` and use it when present.
- If no key is found and none was provided in chat, ask for it.
- Use the API key only for Yolfi API authorization. Use the target endpoint's signing secret for runtime verification and signed local webhook checks.
- Treat API keys as opaque. Do not validate prefixes.
- Never print the full key.
- Never write the key into source files.

After the key validates, fetch the organization profile (`GET /api/private/organization/current`) and surface likely misconfigurations before integrating:

- Placeholder organization name - payers see it on every checkout page.
- Existing endpoint URLs that are placeholders/localhost, duplicate the intended business surface, or use an adapter that conflicts with that endpoint's receiver.
- Missing settlement accounts or a pending onboarding status.

Report these findings; do not change organization settings without explicit user approval.

### 2. Load Docs

Read the two docs sources above. From the current docs, determine:

- Webhook signature headers and verification requirements.
- Current adapter support.
- Current native webhook events and payload shapes.
- Current adapter behavior and payload compatibility.
- Any delivery, retry, or idempotency guidance.

Use this information during implementation, but do not hardcode it into the skill.

### 3. Inspect the Merchant Codebase

Read files before editing. Find:

- Backend framework and route style.
- Existing webhook-capable backend location.
- Raw body handling and JSON middleware order.
- Existing provider webhook routes and signature checks.
- Event dispatchers, handler maps, switch statements, controllers, jobs, or queues.
- Business functions that fulfill orders, activate subscriptions, grant access, update invoices, or send emails.
- Existing idempotency, database, queue, Redis, KV, or unique-key patterns.
- Provider SDK/API calls inside webhook handlers.
- Existing plan/product resolution logic, including provider item IDs, payment link IDs, price/product IDs, variant IDs, lookup keys, metadata keys, custom data, and plan config constants.

Classify the existing payment flow:

- **Existing compatible provider webhook:** docs confirm a Yolfi adapter can send compatible payloads for that provider.
- **Existing payment business logic only:** no compatible webhook exists, but reusable fulfillment/subscription functions exist.
- **No payment backend logic:** the product needs a new native Yolfi webhook skeleton.
- **Unsafe or unclear:** provider API coupling or business boundaries make automatic routing risky.

After classification, ask the user a direct choice before implementation (single decision point).

Frame the question by payment goal, not by webhook plumbing. First establish who pays whom and what the merchant expects to see:

- **Sell with crypto:** the merchant's own customers pay the merchant; the expected outcome is a visible crypto checkout (paylink/button) plus a webhook that grants access.
- **Process crypto events:** the merchant ingests Yolfi payment events into an existing pipeline (attribution, analytics, bookkeeping); no checkout change is expected.

If the repo contains more than one payment surface (for example the product's own billing plus processing of third-party payments), require an explicit surface choice in these goal terms before offering endpoint options. A user who wants a crypto checkout button will not recognize that goal in a webhook-endpoint question.

Then offer the endpoint options for the chosen surface:

- If an existing payment/webhook endpoint exists on that surface: ask whether to integrate into the existing endpoint or create a new Yolfi endpoint.
- If no existing payment/webhook endpoint exists: proceed with a new endpoint skeleton.

### 4. Resolve Plan Mapping

Before editing, identify how the merchant's current payment code decides which plan/product was purchased.

This is a hard gate. Do not write integration code until this is known.

Required output before implementation:

- The exact existing field(s) used for plan resolution, such as provider payment link, price/product, variant, lookup key, metadata/custom data, client reference, or a local config object.
- The exact existing file or constant that owns plan mappings.
- Whether the selected Yolfi adapter already exposes an equivalent field from the current docs.

Default behavior:

- Reuse the merchant's existing plan resolution field when the adapter payload already provides it.
- Add Yolfi paylink IDs to the existing plan/config mechanism, not to a new controller-local map.
- Prefer the adapter's documented provider-compatible paylink/product identifier fields and Yolfi metadata fields for Yolfi paylink IDs.
- If the existing app maps by a provider item/link identifier that Yolfi can represent, ask only for the Yolfi paylink IDs for each existing plan.
- If the app maps by a provider-specific ID that Yolfi cannot represent in the selected adapter payload, ask for a plan-to-Yolfi-paylink-ID mapping and use that in the existing config mechanism.
- Do not ask the merchant to restructure products, add new provider enums, or rewrite checkout/paywall logic.
- Do not invent plan IDs, price IDs, product IDs, provider names, or hardcode mappings without user-provided Yolfi paylink IDs.
- Do not create provider-specific plan map constants in webhook/controller files when the app already has payment config.

### 5. Choose the Smallest Safe Integration

#### Existing Compatible Provider Webhook

Use Adapter Mode. Patch the existing provider webhook endpoint only if the needed provider-shaped branches are body-driven.

Required behavior:

- Keep the original provider signature branch unchanged.
- Add a Yolfi signature branch selected by Yolfi headers from the current docs.
- Verify Yolfi with the raw body, that endpoint's separately stored signing secret, and the current docs' signature algorithm. Never use `YOLFI_API_KEY` as a webhook signing secret.
- Parse the verified Yolfi body.
- Dispatch the parsed adapter payload only into existing provider-shaped event handling.
- If the target branch calls the provider API, stop. Ask the user to choose a separate native endpoint or provide an existing body-driven handler to call.

Minimal patch target for this mode:

- Add auth routing only (provider signature branch vs Yolfi signature branch).
- Keep existing fulfillment/subscription/business functions unchanged.
- Add only the smallest provider-event compatibility normalization needed for existing event handling.
- Keep changes local to webhook ingress/verification and dispatch wiring.
- Do not include "refactor webhook handling" in the plan unless the user explicitly requested a refactor.
- Do not add a second Yolfi lifecycle `switch` inside the provider webhook.

The webhook URL registered in Yolfi should normally be the existing provider webhook URL, with the adapter selected according to current docs.

#### Existing Payment Logic Without Compatible Provider Webhook

Use Native Mode. Create a native Yolfi webhook route in the existing framework style.

Keep it thin:

- Verify Yolfi signature using current docs.
- Apply existing durable idempotency if the app already has a pattern.
- Acknowledge quickly.
- Call existing business functions with data from the current native Yolfi event payload.

Do not invent fake provider payloads unless the user explicitly asks for adapter mode and current docs confirm support.
Do not put this native handler inside an existing provider webhook endpoint.

#### No Existing Payment Logic

Create the minimum native Yolfi webhook skeleton:

- Signature verification.
- Fast acknowledgement.
- Durable idempotency only if the repo already has an obvious durable primitive.
- One isolated fulfillment placeholder named for the merchant's domain, not for a provider.

Keep the placeholder small and easy to replace. Do not build a broad payment abstraction.

#### Unsafe or Unclear

Stop and ask when:

- There is no recognizable backend.
- The repo is frontend-only.
- Current docs do not confirm adapter support for the existing provider.
- Existing webhook logic depends on provider-owned IDs for later refunds, transfers, billing mutations, or subscription API calls.
- No safe business function boundary is visible.
- No durable idempotency/storage pattern exists and duplicate handling cannot be safely ignored.
- Adapter Mode would require mapping adapter events into native Yolfi lifecycle events.

## Environment Handling

- Store the key as `YOLFI_API_KEY`.
- Put real keys only in environment files or secret managers, never source files.
- If editing `.env`, ensure it is ignored by git.
- If `.env.example` exists, add an empty `YOLFI_API_KEY=` entry when missing.
- After editing environment files, check `git status --short`; warn if a real secret is tracked.
- Use a fixture secret in committed tests. Never commit a merchant's real key.

## Verification

Verify the selected integration with the merchant's framework and test style:

- A request with a valid Yolfi signature reaches the intended existing business pipeline or native handler.
- A request with an invalid Yolfi signature is rejected.
- Existing provider webhook requests still use their original verification path.
- Duplicate handling follows the existing durable pattern when one exists.
- Provider API calls are not executed for Yolfi adapter payloads unless explicitly safe.
- Tests or local checks prove the route uses raw body verification.
- Committed tests use a fixture secret, not a real merchant key.

Prefer generating the signed test request with `signWebhookPayload` from `@yolfi/agent` (or an equivalent local HMAC helper) so the check exercises the real header and raw-body path end to end. The minimum live check against a running server is three requests: one valid signature, one duplicate delivery of the same event, one tampered signature.

## Handoff

Report only what was actually implemented:

- Which mode was used: existing provider adapter or native Yolfi webhook.
- Which route receives Yolfi webhooks.
- Which environment variable was added.
- Which docs source and date were used.
- Which tests or local checks passed.
- Any unsafe areas that required a question or were intentionally left untouched.

Also include:

- Exact minimal patch summary (what changed and what explicitly did not change).

## Plan Output Rules

When presenting a plan before implementation:

- Keep it short and mode-specific: Adapter Mode or Native Mode.
- Name the existing plan mapping owner and the requested Yolfi paylink IDs, if any.
- Do not include helper designs for lifecycle mapping, user lookup fallbacks, in-memory deduplication, analytics changes, or subscription cleanup unless the user explicitly asked for those changes.
- If the minimal integration is blocked by provider API lookups or missing reusable business functions, present the blocker and ask for the integration path instead of proposing a refactor.
