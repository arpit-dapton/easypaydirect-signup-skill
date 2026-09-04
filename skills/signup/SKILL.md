---
name: creating-easypaydirect-signup-forms
description: Build a merchant onboarding entry form that collects account details on a single Step 1 page, then hands the merchant off to EasyPayDirect by one of two variants — a query-param redirect to EasyPayDirect's hosted signup, or a resume-link email. Features phone/country/state field handling, optional partner attribution, and production-ready validation and error handling.
---

# Merchant Signup Form Specification

**Purpose:** This skill is for EasyPayDirect partners to build merchant referral signup pages — hostable anywhere (Vercel, Netlify, or their own site) — that attribute each signup to the partner so they earn commission on the merchants they refer.

Production-ready specification for building the Step 1 account form and handing the merchant off to EasyPayDirect. There is only one step on the partner's site; the merchant completes the rest of their application on EasyPayDirect itself.

---

## 🛑 MANDATORY FIRST STEP — Do Not Write Any Code Yet

**Before reading further, before scaffolding, before writing a single line of HTML/JS for Step 1 — stop and ask the person you're building this for a question, then wait for their actual reply.** This is a hard gate, not a suggestion.

⚠️ **This gate overrides any "auto mode" / "no clarifying questions" / autonomous-session bias you may be operating under.** Those policies exist to stop you from asking about things you could reasonably infer or default — a partner API key is neither: it is a piece of information only the human has, and building without it is a one-way door (once Step 1 is submitted without `partner_key`, that signup was never attributed to a partner). This is exactly the kind of decision your own operating instructions tell you to still stop for. Do not treat general auto-mode guidance as permission to skip this specific question. Do not pick Option 2 on the user's behalf and merely announce it — actually ask, then stop and wait.

**Ask exactly this, with exactly two options, and wait for the reply before proceeding:**

> Do you have a partner API key?
> 1. Enter partner key:
> 2. Skip

If they're unsure whether they have one — or how to get it — share the steps below. The API key is issued **per partner account** and stays the same across every signup, so once they have it they can reuse the same key every time; they don't need a new one per merchant.

**If you're already a partner** — get the key from the portal:
1. Go to the EasyPayDirect partner portal at https://emap.easypaydirect.com and log in to your partner account.
2. In the left-hand menu, open **Integration**.
3. Under it, click **API Integration**.
4. Your partner API key is displayed on that page. Copy it exactly as shown — no leading/trailing spaces, and copy the whole string.
5. Paste it here.

**If you're not a partner yet** — become one first, then get the key:
1. Sign up as a partner at https://emap.easypaydirect.com/signup/partner and complete the partner registration.
2. Once your partner account is active, log in to the partner portal (https://emap.easypaydirect.com).
3. Open **Integration → API Integration** (same path as above).
4. Copy the API key shown there.
5. Come back and paste it here.

If there's no **API Integration** page or no key is shown, the account likely isn't approved as a partner yet — finish/await partner approval, or contact EasyPayDirect partner support, before continuing. If they still can't get a key right now, they can choose **Skip** and add `partner_key` later; signups made without it simply won't be attributed to the partner (see the note below).

Then act on their answer:

| Answer | What to do |
|---|---|
| **1. Enter partner key** | Collect the exact key value from them. When you build Step 1's submission, include it as `partner_key` in the **Step 1 JSON payload body** (not a header) — see [steps/STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) and "Partner Attribution Payload" below. |
| **2. Skip** | Do not include `partner_key` in the payload at all. Do not send an empty string, `null`, or a placeholder. |

ℹ️ Both variants carry the partner key, just differently: **Variant 2** sends it as the `partner_key` field in the `POST /api/v1/signup` payload; **Variant 1** (redirect) has no API call, so it forwards the same key as a `secretKey` query param on the `/signup` redirect, which EasyPayDirect resolves to the partner and records as `partner_id`. Either way, collect the key up front (the merchant may not have picked a variant yet).

Only after the user has actually answered this question should you continue to the next mandatory step below.

---

## 🛑 MANDATORY SECOND STEP — Choose the Signup Flow

**After the partner-key question above, and still before writing any code, ask which signup experience to build.** This is also a hard gate — do not default to a variant on the user's behalf, and do not skip this because it seems like the obvious choice. The person answering is very likely not a developer and has no reason to know internal terms like "steps," "flows," "redirect," or "resume-link" — ask in plain language about what the merchant visiting their site will actually experience, not in the codebase's own jargon.

**Ask exactly this, with exactly two options, and wait for the reply before proceeding.** Display it verbatim — do not annotate the options with implementation terms (e.g. "(Step 1)", "(redirect)", "(Variant)") pulled from the table further below. That table is internal guidance for you to act on after they answer; none of its terminology belongs in what the person actually sees.

> How do you want the sign-up process to work for merchants on your site?
> 1. **Quick start, then continue on EasyPayDirect** — the merchant enters their basic contact info on your website. As soon as they submit it, they're taken straight to EasyPayDirect's own website to finish the rest of their application there.
> 2. **Quick start, then we email them a link** — the merchant enters their basic contact info on your website. As soon as they submit it, EasyPayDirect emails them a link they can use to finish the rest of their application whenever they're ready.

Then act on their answer:

| Answer | What to do (internal) |
|---|---|
| **1. Quick start, then continue on EasyPayDirect** (Variant 1 — redirect) | Build **only** the Step 1 page/form. On a valid submit, make **no API call** — instead redirect the browser to EasyPayDirect's hosted-signup `/signup` page with the Step 1 values as query params: `{base_url}/signup?first_name={first_name}&last_name={last_name}&company_name={name}&phone={phone}&email={email}&annual_sales={annual_sales}&website={website}&country={country_name}&industry_type={industry_type}` (`base_url` defaults to `https://emap.epd.dev` — see "Required Implementation Parameters"). URL-encode every value (use `URLSearchParams`); **`company_name` maps to the field named `name`**; **`country` is the country's display name, not the `US`-style code** (`/signup` resolves country by name — forward the selected option's label); **`industry_type` is the industry's display name, not the slug** (append `&industry_type_other=…` when the merchant picks "Other"); append `&business_state={code}` (the 2-letter state code) when the merchant is in the US — `/signup` resolves it to the state id; append `&promo_code=…` when the merchant entered one; append `&secretKey={partner_key}` when the implementer supplied a partner key (this is how the redirect variant attributes the signup to a partner — `/signup` records `partner_id` from it); do **not** send `form_id`. `business_state` also drives the in-form state conditional. **Strip every "N of N" / multi-step signal** — no progress bar, no "Step 1 of …" subtitle, no step counter. There is exactly one step on this site. See [steps/STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) → "Form Submission & Handoff" and "Variant 1 & 2 Backend Endpoints" below. |
| **2. Quick start, then we email them a link** (Variant 2 — resume email) | Build **only** the Step 1 page/form. On a valid submit, `POST /api/v1/signup` (with `"step_count": 1`, and `partner_key` if the implementer supplied one) to create the application, then — automatically, no extra click — `POST /api/v1/signup/resume-link` with the merchant's `email` (read straight from the just-submitted Step 1 data; never prompt them to re-enter it). Then persist a completion flag and show a "check your email" confirmation view. **Strip every multi-step signal** here too — this site has one step. See [steps/STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) → "Form Submission & Handoff" and "Variant 1 & 2 Backend Endpoints" below. |

### Variant 1 & 2 Backend Endpoints

- **Variant 1 (redirect)** needs no backend at all — it's a pure client-side redirect to `{base_url}/signup?<query params>`. The `/signup` page prefills its own form from the params. Nothing in the API changes or is called.
- **Variant 2 (resume email)** — `POST /api/v1/signup` then `POST /api/v1/signup/resume-link` (body `{"email": "<merchant's email>"}`). The latter returns `{"status":true,"message":"Resume link sent"}` (200, always — never reveals whether an email matches an account) and emails the merchant a resume link via EasyPayDirect's existing "finish later" template; `422` for a missing/invalid email, `429` if rate-limited (5 requests / 5 minutes per IP). Both endpoints are unauthenticated. Include `"step_count": 1` in the signup payload so the emailed link resumes past Step 1 rather than at the start.

Full detail and implementation notes are in [reference/BACKEND_DEPENDENCIES.md](reference/BACKEND_DEPENDENCIES.md).

---

## Form Overview

| Step | Title | File | Fields | Features |
|------|-------|------|--------|----------|
| 1 | Account Information | [STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) | 11 | Phone formatting, country/state selection, optional partner attribution |

**Total Fields**: 11 (Step 1 is the only step on the partner's site; the merchant completes the rest of their application on EasyPayDirect).

The two variants (see "MANDATORY SECOND STEP") both build this one Step 1 page and differ only in what happens on submit — a query-param redirect to EasyPayDirect (Variant 1), or a `POST /api/v1/signup` + resume-link email (Variant 2).

---

## Quick Reference Files

**Supporting Documentation**:
- [reference/DROPDOWNS_REFERENCE.md](reference/DROPDOWNS_REFERENCE.md) - Country & US-State dropdown values and implementation
- [reference/api-examples.md](reference/api-examples.md) - Copy-paste JavaScript & cURL code for both variants
- [reference/BACKEND_DEPENDENCIES.md](reference/BACKEND_DEPENDENCIES.md) - What each variant needs from EasyPayDirect

Conditional/dependent field logic lives in the Step 1 file's own "Dependent Fields" section — there is no separate reference file for this; the step file is the single source of truth.

---

## Global Implementation

⚠️ Have you asked the partner-API-key question and the flow-selection question yet? See "MANDATORY FIRST STEP" and "MANDATORY SECOND STEP" at the top of this file. Do not proceed past this point until you have.

### Required Implementation Parameters

Before building the form, configure this **required** parameter:

| Parameter | Type | Description |
|-----------|------|-------------|
| **base_url** | string | The single host every URL in this skill is built from — dropdown fetches, the Variant 1 redirect target, and the Variant 2 API calls. Defaults to `https://emap.epd.dev`. |

⚠️ **Configure the host in exactly one place.** Define it once as a `BASE_URL` constant and build every URL as `` `${BASE_URL}/…` `` — never hardcode `https://emap.epd.dev` inline in more than one spot. Switching environments (staging, a self-hosted instance, etc.) must be a one-line change. The URLs written out below and in the reference files show `{base_url}` as a placeholder for exactly this value. See [reference/api-examples.md](reference/api-examples.md#configuration--the-single-base-url) → "Configuration".

---

### Authentication

⚠️ **None of these endpoints require authentication.** No header is required to submit Step 1 (`/api/v1/signup`), to send the resume-link email (`/api/v1/signup/resume-link`), or to fetch dropdown data — call every endpoint listed in this skill directly with `base_url`, no key needed. (Variant 1 doesn't call the API at all.)

```javascript
headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json'
}
```

### Partner Attribution Payload (Step 1 Only)

Reference for the `partner_key` field asked about above:

```json
{
  "first_name": "John",
  "...": "...other Step 1 fields...",
  "partner_key": "the-partner-api-key-value"
}
```

This is **not authentication** — it only attributes the signup to a partner account for commission/reporting purposes:
- If `partner_key` is present but doesn't match any user's key, Step 1 still succeeds; the signup just isn't attributed to a partner.
- Carried by **both** variants: as the `partner_key` payload field in Variant 2's `POST /api/v1/signup`, and as a `secretKey` query param on Variant 1's `/signup` redirect (same key, same effect — the created application gets `partner_id` set).

---

### Error Handling

Only Variant 2 calls the API, so error handling applies to Variant 2 only (Variant 1 just redirects). The two endpoints it hits are `POST /api/v1/signup` and `POST /api/v1/signup/resume-link`.

**Status codes actually returned by the backend** (verified against `SignupAPIController` and its FormRequest classes):

| Code | When | Endpoint |
|------|------|----------|
| 200 | Success — **every** success response uses 200, never 201 | Both |
| 200 | ⚠️ "Company already exists" during signup — `status:false` but HTTP 200 (no error code set) | `/api/v1/signup` |
| 400 | Bad request — generic exception caught | `/api/v1/signup` |
| 403 | Blocked region (geo-check) | `/api/v1/signup` |
| 422 | Validation failed (e.g. missing/invalid `email`) | Both (via FormRequest `failedValidation`) |
| 429 | Rate-limited (5 requests / 5 minutes per IP) | `/api/v1/signup/resume-link` |

⚠️ **Always check the `status` boolean in the response body — do not rely on the HTTP status code alone.** The signup "Company already exists" case returns HTTP 200 with `status:false`.

**422 Validation Error** (thrown by the FormRequest's `failedValidation`):
```json
{
  "status": false,
  "message": "Validation failed",
  "errors": {
    "email": ["Email is required"],
    "country": ["Country must be a valid country ID"]
  }
}
```

**400 Generic Error** (uncaught exception on `/api/v1/signup`):
```json
{ "status": false, "message": "Error", "data": "<exception message>" }
```

**403 Blocked Region** (`/api/v1/signup`, geo-IP check):
```json
{ "status": false, "message": "Unauthorised access." }
```

**Recommended client handling** (Variant 2):
```javascript
function handleSignupResponse(response, httpStatus) {
  if (response.status === true) {
    // signup succeeded — go on to send the resume-link email
    return;
  }
  if (httpStatus === 422) {
    // response.errors is an object: { field: [messages] }
    for (const [field, messages] of Object.entries(response.errors || {})) {
      displayErrorForField(field, messages[0]);
    }
    return; // stay on Step 1, do not change URL
  }
  // 400/403/429, or status:false with HTTP 200 (e.g. "Company already exists")
  // — show response.message to the user
  showError(response.message);
}
```

### Network Resilience — Retry Once on Transient Fetch Failure

⚠️ **Observed in testing**: a `fetch()` call to a `v1` signup endpoint can occasionally fail on the very first attempt (the promise rejects or the response body fails to parse) even though the backend actually processed the request successfully and a plain retry with the same payload immediately succeeds. Symptom: the user clicks "Continue," sees a generic network-error banner and stays on Step 1, then clicks again with no other change and it proceeds normally. This looks like a transient hiccup on the cross-origin connection to the EasyPayDirect host (e.g. a cold-connection first request), not a payload or endpoint bug.

**Do not make the user click twice to work around this.** In Variant 2, both API calls — the Step 1 `POST /api/v1/signup` and the `POST /api/v1/signup/resume-link` — must retry once automatically before showing an error, using a shared helper (Variant 1 makes no fetch call, so this doesn't apply to it):

```javascript
function fetchWithRetry(url, options, retries) {
  retries = retries === undefined ? 1 : retries;
  return fetch(url, options).then(function(r) {
    return r.json().then(function(body) {
      return { status: r.status, body: body };
    });
  }).catch(function(err) {
    if (retries > 0) {
      return fetchWithRetry(url, options, retries - 1);
    }
    throw err;
  });
}
```

Use `fetchWithRetry(url, options)` in place of the raw `fetch(url, options).then(r => r.json()...)` pattern for Variant 2's signup call and its resume-link call. Only the final failure (after the retry is exhausted) should reach the `.catch()` that shows "Network error. Please check your connection and try again." to the user.

---

## Guidance

### There Is Only One Step

Both variants build a single Step 1 page on the partner's site; the merchant finishes their application on EasyPayDirect. So there is no multi-step navigation, no passing a `uuid` between steps, and no "back to a previous step" here.

- **Variant 1 (redirect)** makes no API call and returns no `uuid`. On a valid submit it hands off to EasyPayDirect via a query-param redirect (see [steps/STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) → "Form Submission & Handoff").
- **Variant 2 (resume email)** submits once to `POST /api/v1/signup` (returning a `uuid`), then calls `POST /api/v1/signup/resume-link`. The `uuid` is not reused for any further step — resume-link is keyed by `email`.

### Page Refresh Behavior

- **Variant 1**: the merchant is redirected off-site the instant Step 1 is valid, so there's nothing to persist. If they return to the partner site, they see a fresh, blank Step 1.
- **Variant 2**: after a successful submit + resume-email, a refresh must show the "check your email" confirmation view, **not** the Step 1 form again (resubmitting the same email would 422 with "email already registered").

⚠️ **Completion is a distinct persisted state.** In Variant 2, persist an explicit flag (e.g. `localStorage.setItem('signup_completed', 'true')`) *before* showing the confirmation view. On page load, check this flag **first**, before rendering the form: if it's set, render the confirmation view directly; otherwise render Step 1 normally. Skipping this check is what makes a refresh after a real, successful submission land back on the Step 1 form. See [steps/STEP1_ACCOUNT_INFORMATION.md](steps/STEP1_ACCOUNT_INFORMATION.md) → "After Submission".

### Form Data Persistence (Refresh Safety)

Persisting Step 1's values isn't strictly required (there's no back-navigation), but it's still useful — e.g. so a mid-typing refresh before submit doesn't wipe the form. If you persist, use these helpers:

```javascript
function saveStep1Data(formElement) {
    const data = {};
    $(formElement).find('input, select, textarea').each(function() {
        const name = $(this).attr('name');
        if (!name) return;
        if ($(this).attr('type') === 'checkbox') {
            if (!data[name]) data[name] = [];
            if ($(this).is(':checked')) data[name].push($(this).val());
        } else if ($(this).attr('type') === 'radio') {
            if ($(this).is(':checked')) data[name] = $(this).val();
        } else {
            data[name] = $(this).val();
        }
    });
    localStorage.setItem('signup_step_1_data', JSON.stringify(data));
}

function restoreStep1Data(formElement) {
    const saved = localStorage.getItem('signup_step_1_data');
    if (!saved) return;
    const data = JSON.parse(saved);
    Object.entries(data).forEach(([name, value]) => {
        const field = $(`[name="${name}"]`, formElement);
        if (Array.isArray(value)) {
            value.forEach(v => $(`[name="${name}"][value="${v}"]`, formElement).prop('checked', true));
        } else if (field.attr('type') === 'radio') {
            $(`[name="${name}"][value="${value}"]`, formElement).prop('checked', true);
        } else {
            field.val(value);
        }
    });
}
```

**After restoring, re-trigger the one conditional** (`business_state` visibility depends on `country`):

```javascript
$(document).ready(function() {
    restoreStep1Data('#step1Form');
    $('#country').trigger('change'); // show/hide + require business_state for US
});
```

**localStorage keys**:

| Key | Contents |
|-----|----------|
| `signup_step_1_data` | Step 1 field values (optional refresh safety) |
| `signup_completed` | `'true'` once Variant 2's submit + resume-email succeed (drives the confirmation view) |

### Field Format Rules

| Field Type | Rule |
|------------|------|
| Phone fields | Must be a valid, correctly formatted phone number before submission (E.164 from intl-tel-input) |
| Currency / amount fields | `annual_sales` is submitted/forwarded as an integer — no decimals, symbols, or commas |

### The Step 1 Country

`country` is a normal Step 1 dropdown. Its in-form job is the one conditional — show/require `business_state` when `country = "US"`. The `<option>` **value** is the country **code/slug** (`"US"`, `"CA"`, ...), never the numeric `id` — use that value for the conditional and for the Variant 2 payload. The two variants forward it differently: Variant 2 sends the **code** in the signup payload; Variant 1's redirect sends the country **name** (the option's label, e.g. `"United States"`) because `/signup` resolves country by name. Both come from the same dropdown — read `.value` for the code, the selected option's text for the name.

---

## Testing Checklist

⚠️ **Never submit real requests to `base_url` to verify your work.** For Variant 2, confirm every item by reading the code, inspecting the request in browser devtools before sending, or intercepting/mocking the `fetch` calls — not by letting `POST /api/v1/signup` or resume-link actually hit the real API. `base_url` points at EasyPayDirect's real signup backend (production or staging), and every successful signup submission creates a real, persisted merchant application record there. There is no sandbox/test mode, so "just submit it once to check it works" leaves behind a real test deal someone has to clean up. If you genuinely need a live submission, ask the person you're building this for first and wait for explicit confirmation — don't decide it on their behalf. (Variant 1 makes no API call, so its redirect can be safely exercised.)

**Both variants**:
- [ ] Country and US-State dropdowns load with real `<option>` **values** (the `code`, not `undefined` or the numeric `id`) — check the actual value, not just that labels appear.
- [ ] Industry Type dropdown loads from `/api/partner/industry-types` with the **name** as each option value (e.g. `"Retail"`), not the slug.
- [ ] `business_state` appears (and is required) only when `country="US"`, hidden otherwise.
- [ ] `industry_type_other` appears (and is required) only when `industry_type="Other"`, hidden otherwise.
- [ ] Required-field validation fires on all Step 1 fields before any handoff.
- [ ] No "Step 1 of N" / multi-step progress signal is shown anywhere — there is exactly one step.
- [ ] Phone submits/forwards as a valid E.164 value; `annual_sales` is an integer.

**Variant 1 (redirect)**:
- [ ] On a valid submit the browser is sent to `{base_url}/signup?...` with all params present and URL-encoded.
- [ ] `company_name` in the URL carries the value of the `name` field; `country` carries the country **name** (not the `US`-style code); `industry_type` carries the industry **name** (not slug); `industry_type_other` is appended when "Other" is chosen; `business_state` (the 2-letter code) is appended when the merchant is in the US; `promo_code` is appended when entered; `secretKey` is appended when a partner key was supplied; no `form_id` is included; no API call is made.

**Variant 2 (resume email)**:
- [ ] On a valid submit, `POST /api/v1/signup` includes `industry_type` (the name), `business_state` (when `country="US"`) and `step_count:1` (plus `promo_code` when entered, `industry_type_other` when "Other", and `partner_key` only if supplied), then `POST /api/v1/signup/resume-link` is called automatically with the same email — no second click, no re-entering the email.
- [ ] After success the "check your email" confirmation view is shown and `signup_completed` is persisted.
- [ ] A page refresh after completion shows the confirmation view, not the Step 1 form.
- [ ] 422 responses display field errors without changing the URL.
- [ ] Both API calls go through `fetchWithRetry` and work with no auth header (signup optionally accepts `partner_key` in the body for attribution).
