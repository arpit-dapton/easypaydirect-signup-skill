---
name: signup-step-1
description: Step 1 Account Information - First step of merchant signup with basic details and phone formatting
---

# STEP 1: Account Information

The only step built on the partner's site. Collects basic merchant contact and company information, then hands the merchant off to EasyPayDirect by one of two variants (see skill.md → "MANDATORY SECOND STEP"):

- **Variant 1 (redirect)** — no API call; redirect the browser to EasyPayDirect's hosted signup with the Step 1 values as query params.
- **Variant 2 (resume email)** — `POST /api/v1/signup`, then `POST /api/v1/signup/resume-link` so the merchant is emailed a link to continue on EasyPayDirect.

The form and its fields/validation are identical for both variants; only the submit action differs.

---

## Fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| first_name | text | Yes | UI cap: max 60 chars. (Backend actually allows up to 255 — 60 is a recommended UI limit, not a validation ceiling.) |
| last_name | text | Yes | UI cap: max 60 chars. Same note as first_name. |
| email | email | Yes | Valid email format. **Must also be unique** — backend rejects with 422 ("This email address is already registered") if the email already exists on a `users` row. Handle this case distinctly in the error UI (e.g. suggest logging in) rather than showing a generic validation message. |
| phone | tel | Yes | Use intl-tel-input library |
| name | text | Yes | Company name. UI cap: max 60 chars (backend allows up to 255 — same note as first_name). |
| website | url | Yes | Backend regex is domain-only with an optional single trailing slash — it does **not** accept a path (`https://facebook.com/mybusiness` fails). Despite the "or social media profile" framing below, only a bare social-media domain (no page/profile path) currently validates. Flag this with product before promising path-based social profile URLs work. |
| country | select | Yes | Normal dropdown (no search) - API: `/api/partner/countries` |
| business_state | select | Yes | Normal dropdown (no search) - Show only if country="US", API: `/api/partner/states` |
| annual_sales | number | Yes | Numeric only, no currency symbols or commas |
| promo_code | text | No | Optional referral code |
| partner_key | text | No | Optional partner API key for attribution — see skill.md → "MANDATORY FIRST STEP" (top of file). Ask the implementer if they have one before including it; omit entirely if not |
| step_count | — | **Yes (Variant 2 only)** | Not a form field — in the Variant 2 signup payload, always send the literal value `1` (not user-editable). Without it, the backend never records the application as having reached step 1, so the emailed resume link lands the merchant back at the start instead of past Step 1. Variant 1 sends no payload, so `step_count` does not apply there. |

---

## Country Dropdown

**Field**: `country`  
**Type**: SELECT dropdown  
**API Endpoint**: `GET /api/partner/countries`

**Response Format**:
```json
{
  "success": true,
  "data": [
    { "name": "United States", "code": "US", "id": "1" },
    { "name": "Canada", "code": "CA", "id": "2" },
    { "name": "Mexico", "code": "MX", "id": "3" },
    ...
  ]
}
```

**⚠️ CRITICAL: Use `code` (slug) as option value, NOT `id`**
```html
<!-- CORRECT -->
<option value="US">United States</option>
<option value="CA">Canada</option>
<option value="MX">Mexico</option>

<!-- WRONG - do NOT use id -->
<option value="1">United States</option>
<option value="2">Canada</option>
```

**Form Submission**: Submit the `code` value (e.g., "US", "CA", "MX")

---

## Libraries Required

- `intl-tel-input@22.0.2` - Phone formatting with country selector
- `dependent-fields.js` - Conditional field visibility

**Use these exact CDN URLs — do not guess or substitute a different CDN/path/version.** `intl-tel-input`'s build output path has changed across major versions, so a guessed path (e.g. `js/utils.js` instead of `build/js/utils.js`, or an unpkg URL instead of jsdelivr) is a common source of a silently-broken phone field. Verified working (HTTP 200) as of this writing:
```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/intl-tel-input@22.0.2/build/css/intlTelInput.css">
<script src="https://cdn.jsdelivr.net/npm/intl-tel-input@22.0.2/build/js/intlTelInput.min.js"></script>
```
```js
window.intlTelInput(phoneInputEl, {
  utilsScript: 'https://cdn.jsdelivr.net/npm/intl-tel-input@22.0.2/build/js/utils.js',
  initialCountry: 'us'
});
```

---

## API Integration

**Dropdown Data**:
```
GET /api/partner/countries
GET /api/partner/states
```

**Form Submission** (Variant 2 only — Variant 1 makes no API call, see "Form Submission & Handoff" below):
```
POST /api/v1/signup
Headers: None required.
Payload: 
  country: "US" (use code/slug, not id)
  all other Step 1 fields
  step_count: 1
  partner_key: "..." (OPTIONAL — only include if the implementer has a partner
                API key; see skill.md → "MANDATORY FIRST STEP" (top of file). This is
                not authentication — signup succeeds even if omitted or invalid.)
Response: { uuid, step_count, message }
```

Then, on success, `POST /api/v1/signup/resume-link` with `{ "email": "<the merchant's email>" }` to email the resume link. See [reference/BACKEND_DEPENDENCIES.md](../reference/BACKEND_DEPENDENCIES.md).

---

## Validation

- First/Last name: required, UI cap 60 chars (backend allows up to 255)
- Email: required, valid format, must be unique (422 if already registered)
- Phone: required, valid number (intl-tel-input validates)
- Website: required, valid URL — domain only, no path (see Fields table note)
- Country: required, must be valid country code (e.g., "US", "CA")
- Business State: required if country="US" (use code, not id)
- Annual Sales: required, numeric only (no currency symbols or commas)

---

## Dependent Fields

- **business_state**: Show if country="US" (hidden by default)

---

## Form Submission & Handoff

The field validation is identical for both variants. What happens after a valid submit depends on the variant chosen at the flow gate (skill.md → "MANDATORY SECOND STEP").

### Variant 1 — redirect (no API call)

Build EasyPayDirect's hosted-signup URL from the Step 1 values and navigate to it. Only 7 fields are forwarded; `company_name` maps to the field named `name`; no `form_id`. Encode every value.

```javascript
// After the form passes validation:
const params = new URLSearchParams({
    first_name:   $('[name="first_name"]').val(),
    last_name:    $('[name="last_name"]').val(),
    company_name: $('[name="name"]').val(),        // company-name field is `name`
    phone:        $('[name="phone"]').val(),        // E.164; '+' becomes %2B
    email:        $('[name="email"]').val(),
    annual_sales: $('[name="annual_sales"]').val(),
    website:      $('[name="website"]').val()
});
window.location.href = `https://emap.epd.dev/?${params.toString()}`;
```

No API request, no `uuid`, and nothing to persist — the merchant leaves this site immediately. Because there is exactly one page here and the merchant is handed off after it, do not show any "Step 1 of N" / multi-step progress signal.

### Variant 2 — submit, then email a resume link

**On Success (HTTP 200)** of `POST /api/v1/signup`:
```javascript
// Response example:
{
  "status": true,
  "message": "Account created successfully",
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "step_count": 1
}

// 1. Email the resume link (endpoint does an email lookup — needs the app just created).
//    Use fetchWithRetry (skill.md → Network Resilience).
await fetchWithRetry('https://emap.epd.dev/api/v1/signup/resume-link', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    body: JSON.stringify({ email: $('[name="email"]').val() })
});

// 2. Persist completion so a refresh re-shows the confirmation view, not the form
//    (see skill.md → Page Refresh Behavior).
localStorage.setItem('signup_completed', 'true');

// 3. Show the "check your email" confirmation view.
```

**On Validation Error (HTTP 422)** (Variant 2): standard shape — see skill.md → Error Handling. Stay on Step 1, display field errors, do not change URL.

---

## After Submission

There is no back-navigation between steps here — Step 1 is the only step. What "already submitted" means, and how a page reload behaves, differs by variant:

- **Variant 1 (redirect)**: the merchant leaves this site the instant Step 1 is valid (`window.location.href` to EasyPayDirect). There is nothing to lock or persist — if they come back to the partner site later, they simply see a fresh, blank Step 1.
- **Variant 2 (resume email)**: after a successful submit + resume-email call, persist a completion flag and show a "check your email" confirmation view instead of the form. On page load, check this flag **first** — if it's set, render the confirmation view directly and never re-render the editable form. This prevents a refresh from showing Step 1 again (and prevents a resubmit, which would 422 with "email already registered").

### Variant 2 — completion check on load

```javascript
$(document).ready(function() {
    if (localStorage.getItem('signup_completed') === 'true') {
        // Already submitted in this browser — show confirmation, not the form.
        showCheckYourEmailView();      // render the "check your email" view
        return;                        // do not build/restore the editable Step 1 form
    }

    // Otherwise render Step 1 normally.
});
```

Offer a "start over" affordance on the confirmation view (clears `signup_completed` and the persisted Step 1 data, then reloads a blank Step 1) so a returning merchant is never permanently stuck on the confirmation screen.

---

## Field Summary

**Total Fields**: 11  
**Required Fields**: 9 (all except promo_code and partner_key)  
**Conditionally Required**: 1 (business_state - required if country=US)  
**Optional**: 2 (promo_code, partner_key)  
**Phone Fields**: 1 (intl-tel-input)
