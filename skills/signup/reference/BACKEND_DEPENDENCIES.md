---
name: signup-backend-dependencies
description: What each of the two Step-1-only signup variants needs from EasyPayDirect — the query-param redirect (Variant 1) and the resume email (Variant 2)
---

# Backend Dependencies — the Two Signup Variants

Both variants build only Step 1 on the partner's site. They differ only in what happens after Step 1.

> **`{base_url}` is a single configurable host.** Every URL below — the redirect target and both API endpoints — is `base_url` + a path. It defaults to `https://emap.epd.dev`; change that one value to point at another environment. In code it's the `BASE_URL` constant (see [api-examples.md → Configuration](api-examples.md#configuration--the-single-base-url)).

---

## Variant 1 (redirect) — no backend needed

Variant 1 makes **no API call at all**. After the merchant fills Step 1, redirect the browser to EasyPayDirect's hosted-signup `/signup` page with the Step 1 values as query params — that page prefills its own form from them:

```
{base_url}/signup?first_name={first_name}&last_name={last_name}&company_name={name}&phone={phone}&email={email}&annual_sales={annual_sales}&website={website}&industry_type={industry_type}
```

- **URL-encode every value** with `encodeURIComponent`. The E.164 phone's leading `+` becomes `%2B`.
- **`company_name` maps to the Step 1 field named `name`** (the company-name field is `name`, not `company_name`).
- **`industry_type` is the industry's display name** (e.g. `Retail`), not the slug — the `/signup` page resolves the industry by name. If the merchant picked "Other", also append `&industry_type_other={free text}`.
- `country`, `business_state`, `promo_code`, and `partner_key` are still collected on the form but are not part of the redirect.
- No `form_id` is sent — it is not required for this flow.

Because nothing is submitted to `/api/v1/signup`, there is no `uuid`, no persisted application, and no partner attribution in this variant.

---

## Variant 2 (resume email) — two API calls

Variant 2 submits Step 1, then triggers an email so the merchant can continue on EasyPayDirect.

### 1. `POST /api/v1/signup`

Standard Step 1 submission (see [steps/STEP1_ACCOUNT_INFORMATION.md](../steps/STEP1_ACCOUNT_INFORMATION.md)). Include `"step_count": 1` so the application is recorded as having reached Step 1 (the emailed resume link then lands the merchant past Step 1 rather than back at the start). Include `partner_key` only if the implementer supplied one. Returns a `uuid`.

### 2. `POST /api/v1/signup/resume-link`

Unauthenticated. Call it immediately after the signup POST succeeds, with the same email the merchant just entered.

**The `/api` prefix is not optional.** EasyPayDirect's CORS config only allows cross-origin requests under `api/*` — this route only exists on the server as `/api/v1/signup/resume-link`. Calling the bare `/v1/...` path from a partner-hosted form isn't covered by CORS: the OPTIONS preflight falls through to default routing without proper `Access-Control-Allow-*` headers, and the browser reports a network error with no real response. If a resume-email call fails this way, check the URL for a missing `/api` first.

⚠️ Like every fetch to a `v1` endpoint, this call can occasionally fail on the very first attempt due to a transient cross-origin connection hiccup and succeed on a plain retry. Call it through the `fetchWithRetry` helper (see [skill.md § Network Resilience](../SKILL.md#network-resilience--retry-once-on-transient-fetch-failure)) rather than a raw `fetch(...).catch(...)`.

**Request**:
```json
{ "email": "merchant@example.com" }
```

**Response** (200, always when `email` is valid — never reveals whether an email matches an account):
```json
{ "status": true, "message": "Resume link sent" }
```

- `422` — `email` missing/invalid, e.g. `{ "status": false, "message": "The email field is required." }`.
- `429` — rate-limited (5 requests / 5 minutes per IP).

---

## Summary

| Variant | What it needs from EasyPayDirect |
|---|---|
| 1 (redirect) | Nothing — client-side redirect to `{base_url}/signup?...` with the mapped query params (incl. `industry_type` as the industry name) |
| 2 (resume email) | `POST /api/v1/signup` (with `step_count:1`), then `POST /api/v1/signup/resume-link` with the email |
