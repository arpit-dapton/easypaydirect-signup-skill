# API Dropdowns Reference

API-backed dropdown values used by the signup form. Step 1 is the only step, and it has exactly two dropdowns — **Countries** and **US States** — both fetched from the API.

⚠️ **Neither endpoint requires an auth header** (see skill.md → Authentication).

## Contents
- Countries
- US States
- Dropdown implementation (fetch, populate, validate)
- Value vs Label

---

## API-Based Dropdowns

| Endpoint | Value field | Used by |
|---|---|---|
| `/api/partner/countries` | `code` | Step 1 (`country`) |
| `/api/partner/states` | `code` | Step 1 (`business_state`, US only) |

Both endpoints return a `code` field — always use `code` (the slug, e.g. `"US"`) as each `<option>`'s value, never the numeric `id`. Submitting the `id` is rejected by the backend.

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

// Both Step 1 dropdowns use 'code'.
loadDropdown('/api/partner/countries', document.querySelector('select[name="country"]'), 'code');
// States load the same way; show/require the field only when country === 'US'.
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

- **Label** (displayed to the user): "United States", "California".
- **Value** (submitted to the API): the `code`, e.g. "US", "CA".

Always submit the `code` value, never the display label.
