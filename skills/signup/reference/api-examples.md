# API Implementation Examples

Ready-to-use code for the Step-1-only signup form and its two handoff variants.

---

## Authentication

⚠️ **None of these endpoints require authentication.** Do not send `X-API-Key` or `Authorization` headers to the dropdown endpoints, to `POST /api/v1/signup`, or to `POST /api/v1/signup/resume-link`.

The one exception: `POST /api/v1/signup` accepts an **optional `partner_key` field in the payload body** (not a header) for non-blocking partner attribution. Ask the implementer whether they have a partner API key before including it — see [skill.md](../SKILL.md) → "MANDATORY FIRST STEP". `partner_key` is only ever sent in Variant 2 (Variant 1 makes no API call).

---

## Dropdown APIs

Step 1 has exactly two dropdowns — Countries and US States. Both use the `code` field as the option value (see [DROPDOWNS_REFERENCE.md](DROPDOWNS_REFERENCE.md)).

### Load Countries

**JavaScript**:
```javascript
async function loadCountries() {
  try {
    const response = await fetch('https://emap.epd.dev/api/partner/countries');
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
curl -X GET "https://emap.epd.dev/api/partner/countries" \
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
  const response = await fetch('https://emap.epd.dev/api/partner/states');
  const result = await response.json();
  return result.data;  // [{name: "California", code: "CA"}, ...]
}
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
    country:        form.querySelector('[name="country"]').value,     // code, e.g. "US"
    annual_sales:   form.querySelector('[name="annual_sales"]').value,
    business_state: form.querySelector('[name="business_state"]')?.value || '',
    promo_code:     form.querySelector('[name="promo_code"]')?.value || ''
  };
}
```

---

## Variant 1 — redirect to EasyPayDirect (no API call)

Build EasyPayDirect's hosted-signup URL from the Step 1 values and navigate to it. Only 7 fields are forwarded; `company_name` maps to the `name` field. No `form_id`, no API request.

```javascript
function redirectToEmap(step1) {
  const params = new URLSearchParams({
    first_name:   step1.first_name,
    last_name:    step1.last_name,
    company_name: step1.name,          // company-name field is `name`
    phone:        step1.phone,         // '+' is encoded to %2B automatically
    email:        step1.email,
    annual_sales: step1.annual_sales,
    website:      step1.website
  });
  window.location.href = `https://emap.epd.dev/?${params.toString()}`;
}

// Usage
document.getElementById('signupForm').addEventListener('submit', (e) => {
  e.preventDefault();
  if (!validateStep1()) return;        // run the same field validation as always
  redirectToEmap(readStep1(e.target));
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
    business_state: step1.business_state || null,
    step_count: 1
  };
  // OPTIONAL: only if the implementer supplied a partner key (see skill.md → MANDATORY FIRST STEP)
  if (partnerKey) payload.partner_key = partnerKey;

  // 1. Create the application
  const signup = await fetchWithRetry('https://emap.epd.dev/api/v1/signup', {
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
  await fetchWithRetry('https://emap.epd.dev/api/v1/signup/resume-link', {
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
curl -X POST "https://emap.epd.dev/api/v1/signup" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "John", "last_name": "Doe", "email": "john@example.com",
    "phone": "+12015551234", "name": "Acme Corp", "website": "https://acme.com",
    "country": "US", "annual_sales": 500000, "business_state": "CA", "step_count": 1
  }'
# Success (200): { "status": true, "uuid": "...", "step_count": 1, "message": "Account created successfully" }

curl -X POST "https://emap.epd.dev/api/v1/signup/resume-link" \
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
// Dropdown fetch
const response = await fetch('https://emap.epd.dev/api/partner/{ENDPOINT}');

// Form submission (Variant 2)
const response = await fetch('https://emap.epd.dev/api/v1/{ENDPOINT}', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
```
