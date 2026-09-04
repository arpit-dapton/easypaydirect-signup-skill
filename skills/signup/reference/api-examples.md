# API Implementation Examples

Ready-to-use code for the Step-1-only signup form and its two handoff variants.

---

## Configuration — the single base URL

Every URL in this skill (dropdown fetches, the Variant 1 redirect, and the Variant 2 API calls) is built from **one** host. Define it once and reference it everywhere — to point the form at a different environment, change this single value and nothing else:

```javascript
// The ONE place the host is configured. Change this to switch environments.
const BASE_URL = 'https://emap.epd.dev';
```

All the examples below assume this `BASE_URL` is in scope and build their URLs from it as `` `${BASE_URL}/…` ``. Do not hardcode the host anywhere else. For cURL, set it as a shell variable once:

```bash
BASE_URL="https://emap.epd.dev"
```

---

## Authentication

⚠️ **None of these endpoints require authentication.** Do not send `X-API-Key` or `Authorization` headers to the dropdown endpoints, to `POST /api/v1/signup`, or to `POST /api/v1/signup/resume-link`.

The one exception is partner attribution, and it is **not** authentication — signup succeeds with or without it. Ask the implementer whether they have a partner API key before including it — see [skill.md](../SKILL.md) → "MANDATORY FIRST STEP". The same key is carried differently by each variant:
- **Variant 2** sends it as an **optional `partner_key` field in the payload body** (not a header) of `POST /api/v1/signup`.
- **Variant 1** (redirect, no API call) forwards the same key as a **`secretKey` query param** on the `/signup` redirect — EasyPayDirect resolves it to the partner and records `partner_id` on the created application.

---

## Dropdown APIs

Step 1 has three dropdowns — Countries, US States, and Industry Types. Countries and US States use the `code` field as the option value; Industry Types use the `name` field (see [DROPDOWNS_REFERENCE.md](DROPDOWNS_REFERENCE.md)).

### Load Countries

**JavaScript**:
```javascript
async function loadCountries() {
  try {
    const response = await fetch(`${BASE_URL}/api/partner/countries`);
    if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
    const result = await response.json();
    return result.data;  // [{name: "United States", code: "US"}, ...]
  } catch (error) {
    console.error('Failed to load countries:', error);
    return [];
  }
}

// Populate dropdown
const countries = await loadCountries();
const countrySelect = document.querySelector('select[name="country"]');
countries.forEach(country => {
  const option = document.createElement('option');
  option.value = country.code;         // use code, not id
  option.textContent = country.name;
  countrySelect.appendChild(option);
});
```

**cURL**:
```bash
curl -X GET "$BASE_URL/api/partner/countries" \
  -H "Content-Type: application/json"

# Response:
# {
#   "success": true,
#   "message": "Country list",
#   "data": [
#     { "name": "United States", "code": "US" },
#     { "name": "Canada", "code": "CA" }
#   ]
# }
```

### Load States

States populate the `business_state` dropdown, which is shown/required only when `country = "US"`.

```javascript
async function loadStates() {
  const response = await fetch(`${BASE_URL}/api/partner/states`);
  const result = await response.json();
  return result.data;  // [{name: "California", code: "CA"}, ...]
}
```

### Load Industry Types

Populates the `industry_type` dropdown. The list has only `name` and `slug` — **use `name` as the option value** (EasyPayDirect resolves the industry by its display name, not the slug).

```javascript
async function loadIndustries() {
  const response = await fetch(`${BASE_URL}/api/partner/industry-types`);
  const result = await response.json();
  return result.data;  // [{name: "Retail", slug: "retail"}, {name: "Other", slug: "other"}, ...]
}

// Populate dropdown — value is the NAME, not the slug
const industries = await loadIndustries();
const industrySelect = document.querySelector('select[name="industry_type"]');
industries.forEach(ind => {
  const option = document.createElement('option');
  option.value = ind.name;              // use name, not slug
  option.textContent = ind.name;
  industrySelect.appendChild(option);
});

// When "Other" is selected, show/require the free-text industry_type_other field.
industrySelect.addEventListener('change', (e) => {
  const isOther = e.target.value === 'Other';
  const otherGroup = document.getElementById('industry_type_other_group');
  otherGroup.hidden = !isOther;
  otherGroup.querySelector('[name="industry_type_other"]').required = isOther;
});
```

---

## Step 1 form values

Both variants read the same Step 1 fields off the form. Collect them once:

```javascript
function readStep1(form) {
  return {
    first_name:     form.querySelector('[name="first_name"]').value,
    last_name:      form.querySelector('[name="last_name"]').value,
    email:          form.querySelector('[name="email"]').value,
    phone:          form.querySelector('[name="phone"]').value,      // E.164, e.g. +12015551234
    name:           form.querySelector('[name="name"]').value,        // company name
    website:        form.querySelector('[name="website"]').value,
    country:        form.querySelector('[name="country"]').value,     // code, e.g. "US" (Variant 2 payload)
    country_name:   form.querySelector('[name="country"] option:selected')?.textContent.trim() || '', // NAME, e.g. "United States" (Variant 1 redirect)
    annual_sales:   form.querySelector('[name="annual_sales"]').value,
    industry_type:  form.querySelector('[name="industry_type"]').value, // the NAME, e.g. "Retail"
    industry_type_other: form.querySelector('[name="industry_type_other"]')?.value || '',
    business_state: form.querySelector('[name="business_state"]')?.value || '',
    promo_code:     form.querySelector('[name="promo_code"]')?.value || ''
  };
}
```

---

## Variant 1 — redirect to EasyPayDirect (no API call)

Build EasyPayDirect's hosted-signup URL from the Step 1 values and navigate to the `/signup` page, which prefills its own form from the query params. `company_name` maps to the `name` field; `industry_type` carries the industry **name**; `country` carries the country **name** (not the code — `/signup` resolves country by name). Forward `business_state` (the 2-letter code) when the merchant is in the US, `promo_code` when present, and the partner key as `secretKey` so the created application is attributed to the partner. No `form_id`, no API request.

```javascript
function redirectToEmap(step1, partnerKey) {
  const params = new URLSearchParams({
    first_name:    step1.first_name,
    last_name:     step1.last_name,
    company_name:  step1.name,           // company-name field is `name`
    phone:         step1.phone,          // '+' is encoded to %2B automatically
    email:         step1.email,
    annual_sales:  step1.annual_sales,
    website:       step1.website,
    country:       step1.country_name,   // the country NAME (e.g. "United States") — /signup resolves country by name, not code
    industry_type: step1.industry_type   // the industry NAME, e.g. "Retail"
  });
  // If the merchant picked "Other", forward their free-text industry too.
  if (step1.industry_type === 'Other' && step1.industry_type_other) {
    params.set('industry_type_other', step1.industry_type_other);
  }
  // US state — forward the 2-letter code when present (/signup resolves it to the state id).
  if (step1.business_state) params.set('business_state', step1.business_state);
  // Optional referral code — forward only when provided.
  if (step1.promo_code) params.set('promo_code', step1.promo_code);
  // Partner attribution — forward the partner key as `secretKey` so EasyPayDirect records
  // partner_id on the created application. Only when the implementer supplied a partner key.
  if (partnerKey) params.set('secretKey', partnerKey);
  window.location.href = `${BASE_URL}/signup?${params.toString()}`;
}

// Usage — PARTNER_KEY is the configured partner key ('' or undefined if none)
document.getElementById('signupForm').addEventListener('submit', (e) => {
  e.preventDefault();
  if (!validateStep1()) return;        // run the same field validation as always
  redirectToEmap(readStep1(e.target), PARTNER_KEY);
});
```

`URLSearchParams` handles encoding, so `phone: "+12015551234"` becomes `phone=%2B12015551234` and `name: "John Company"` becomes `company_name=John+Company`.

---

## Variant 2 — submit Step 1, then email a resume link

Submit to `POST /api/v1/signup`, then on success call `POST /api/v1/signup/resume-link` with the merchant's email, then show a "check your email" view. Use `fetchWithRetry` (see [skill.md § Network Resilience](../SKILL.md#network-resilience--retry-once-on-transient-fetch-failure)) for both calls.

```javascript
async function submitVariant2(step1, partnerKey) {
  const payload = {
    first_name: step1.first_name,
    last_name: step1.last_name,
    email: step1.email,
    phone: step1.phone,
    name: step1.name,
    website: step1.website,
    country: step1.country,
    annual_sales: parseInt(step1.annual_sales, 10),
    industry_type: step1.industry_type,   // the industry NAME, e.g. "Retail"
    business_state: step1.business_state || null,
    step_count: 1
  };
  // Only when the merchant picked "Other"
  if (step1.industry_type === 'Other' && step1.industry_type_other) {
    payload.industry_type_other = step1.industry_type_other;
  }
  // Optional referral code — include only when the merchant entered one
  if (step1.promo_code) payload.promo_code = step1.promo_code;
  // OPTIONAL: only if the implementer supplied a partner key (see skill.md → MANDATORY FIRST STEP)
  if (partnerKey) payload.partner_key = partnerKey;

  // 1. Create the application
  const signup = await fetchWithRetry(`${BASE_URL}/api/v1/signup`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    body: JSON.stringify(payload)
  });

  if (!signup.body.status) {
    // 422 → field errors in signup.body.errors; 200 status:false → "Company already exists"
    handleSignupError(signup);
    return;
  }

  // 2. Email the resume link (endpoint does an email lookup — needs the app created above)
  await fetchWithRetry(`${BASE_URL}/api/v1/signup/resume-link`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    body: JSON.stringify({ email: step1.email })
  });

  // 3. Persist completion and show the confirmation view (see skill.md → Page Refresh Behavior)
  localStorage.setItem('signup_completed', 'true');
  showCheckYourEmailView(step1.email);
}
```

**cURL — the two calls**:
```bash
curl -X POST "$BASE_URL/api/v1/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "John", "last_name": "Doe", "email": "john@example.com",
    "phone": "+12015551234", "name": "Acme Corp", "website": "https://acme.com",
    "country": "US", "annual_sales": 500000, "industry_type": "Retail",
    "business_state": "CA", "step_count": 1
  }'
# Success (200): { "status": true, "uuid": "...", "step_count": 1, "message": "Account created successfully" }

curl -X POST "$BASE_URL/api/v1/signup/resume-link" \
  -H "Content-Type: application/json" \
  -d '{ "email": "john@example.com" }'
# Success (200): { "status": true, "message": "Resume link sent" }
```

---

## Error Handling

See [skill.md](../SKILL.md) → Error Handling for status codes, response shapes, and a worked `handleStepResponse()` example. The relevant codes here are 200 (incl. the 200 `status:false` "Company already exists" case), 400, 403, 422, and 429 (resume-link rate limit).

---

## Quick Copy-Paste Template

```javascript
// const BASE_URL = 'https://emap.epd.dev';  // configured once (see "Configuration" above)

// Dropdown fetch
const response = await fetch(`${BASE_URL}/api/partner/{ENDPOINT}`);

// Form submission (Variant 2)
const response = await fetch(`${BASE_URL}/api/v1/{ENDPOINT}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
```
