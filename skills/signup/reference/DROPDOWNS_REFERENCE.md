# API Dropdowns Reference

API-backed dropdown values used by the signup form. Step 1 is the only step, and it has three dropdowns — **Countries**, **US States**, and **Industry Types** — all fetched from the API.

⚠️ **None of these endpoints requires an auth header** (see skill.md → Authentication).

## Contents
- Countries
- US States
- Industry Types
- Dropdown implementation (fetch, populate, validate)
- Value vs Label

---

## API-Based Dropdowns

| Endpoint | Value field | Used by |
|---|---|---|
| `/api/partner/countries` | `code` | Step 1 (`country`) |
| `/api/partner/states` | `code` | Step 1 (`business_state`, US only) |
| `/api/partner/industry-types` | `name` | Step 1 (`industry_type`) |

Countries and US States return a `code` field — always use `code` (the slug, e.g. `"US"`) as each `<option>`'s value, never the numeric `id`. Submitting the `id` is rejected by the backend. **Industry Types are the exception**: that list has only `name` and `slug`, and EasyPayDirect resolves the industry **by its display name** — so use `name` (e.g. `"Retail"`) as the option value, not `slug`.

### Countries
**Endpoint**: `GET /api/partner/countries`

**Response Format**:
```json
{
  "success": true,
  "data": [
    { "name": "United States", "code": "US", "id": "1" },
    { "name": "Canada", "code": "CA", "id": "2" },
    { "name": "Mexico", "code": "MX", "id": "3" }
  ]
}
```

**⚠️ CRITICAL: Use the `code` field as option value, NOT `id`.**

**Form Submission**: Use code values (`"US"`, `"CA"`, `"MX"`, ...).

**Conditional Logic**: Use code values for show/hide logic — show `business_state` only when `country = "US"`.

---

### US States
**Endpoint**: `GET /api/partner/states`
**Conditional**: Shown only when `country = "US"` (compare against the country `code`, not `id`).

**Response Format**:
```json
{
  "success": true,
  "data": [
    { "name": "California", "code": "CA" },
    { "name": "New York", "code": "NY" },
    { "name": "Texas", "code": "TX" }
  ]
}
```

---

### Industry Types
**Endpoint**: `GET /api/partner/industry-types`

**Response Format** (note: `name` + `slug` only — no `code`, no `id`):
```json
{
  "success": true,
  "data": [
    { "name": "Retail", "slug": "retail" },
    { "name": "E-commerce", "slug": "ecommerce" },
    { "name": "SaaS", "slug": "saas" },
    { "name": "Other", "slug": "other" }
  ]
}
```

**⚠️ CRITICAL: Use the `name` field as option value, NOT `slug`.** EasyPayDirect matches this field by its display name, so `<option value="Retail">Retail</option>`.

**Conditional**: when `industry_type = "Other"`, show and require the free-text `industry_type_other` field (max 255 chars).

---

## Dropdown Implementation

Both `country` and `business_state` are **plain, normal `<select>` elements** — no searchable/live-search UI, no third-party select-enhancement library.

**HTML Template**:
```html
<div class="form-group">
    <label for="country">Country *</label>
    <select id="country" name="country" class="form-control" required>
        <option value="" disabled selected>Select Country...</option>
        <!-- Options populated via JavaScript -->
    </select>
    <div class="invalid-feedback" id="country_error"></div>
</div>
```

**JavaScript — fetch & populate**. Pass the value field (`code`) explicitly and fail loudly if an item is missing it, rather than silently rendering `value="undefined"`:
```javascript
async function loadDropdown(endpoint, selectEl, valueKey) {
    const res = await fetch(endpoint).then(r => r.json());
    res.data.forEach(item => {
        const value = item[valueKey];
        if (value === undefined) {
            console.error(`${endpoint}: item has no "${valueKey}" field`, item);
            return; // skip rather than render value="undefined"
        }
        const option = document.createElement('option');
        option.value = value;
        option.textContent = item.name;
        selectEl.appendChild(option);
    });
}

// Country and State dropdowns use 'code'; Industry Types use 'name'.
loadDropdown('/api/partner/countries', document.querySelector('select[name="country"]'), 'code');
// States load the same way; show/require the field only when country === 'US'.
loadDropdown('/api/partner/industry-types', document.querySelector('select[name="industry_type"]'), 'name');
// When industry_type === 'Other', show/require the free-text industry_type_other field.
```

**Conditional visibility** — when `country` changes, show/hide `business_state`:
```javascript
$('#country').on('change', function () {
    const isUS = $(this).val() === 'US';
    $('#business_state_group').toggle(isUS);
    $('#business_state').prop('required', isUS);
    if (!isUS) $('#business_state').val('');
});
```

---

## Value vs Label

- **Label** (displayed to the user): "United States", "California", "Retail".
- **Value** (submitted/forwarded): for Countries and States, the `code` (e.g. "US", "CA"); for Industry Types, the `name` (e.g. "Retail").

For Countries and States, always submit the `code`, never the display label. For Industry Types the value **is** the name — label and value are the same string.
