# Custom App Validation – Changes and Rationale

This document explains every change made to run **all** custom app validations before create (single, multiple, bulk) and to reuse the same validation as the create API, so there is only one place to maintain.

---

## 1. API Server (apiserver)

### 1.1 `CustomAppsResource.validate_uri_conflicts` (adminapi.py)

**What:** New POST endpoint `/api/admin/v1/customapps/validate_uri_conflicts/` that pre-validates custom app payloads **using the same `CustomAppsValidation` as create**.

**Why:**

- Users see validation errors (name, URIs, conflicts, audit_data) **before** submitting create, so invalid requests are caught early.
- There is **no duplicate validation logic**: the endpoint builds a minimal bundle and calls `CustomAppsValidation().is_valid(bundle, request)`, i.e. the same validator used by `obj_create`.

**Details:**

- Request body: `{ apps: [ { app_name, app_uris, is_active?, audit_data?, id? } ] }`.
- For each app, `bundle.data` is built from the same fields as create (`app_name`, `app_uris`, `is_active`, optional `audit_data`, optional `id`) so **all** validations run: app name (blank, length, special chars, already exists in custom/Elastica/tenant, " Suite" suffix), app_uris (at least one, URL format, URI conflicts with other custom apps and full gatelets), and when present, audit_data (attributes, question_answers).
- Returns `{ valid: true }` or `{ valid: false, errors: [ { app_name, error } ] }`.

### 1.2 No separate `validate_bulk_create_uri_conflicts` (validation.py)

**What:** Removed the duplicate function that reimplemented URI conflict and URL checks.

**Why:**

- Avoid maintaining two implementations.
- Pre-validate now reuses **all** of `CustomAppsValidation.is_valid()` (not only URI checks), so behaviour matches create exactly.

---

## 2. Admin Portal (Django)

### 2.1 `validate_custom_app_conflicts` view (views.py)

**What:** New view that proxies POST to the API server `validate_uri_conflicts` and returns the response.

**Why:**

- Frontend talks to the admin portal; the portal forwards to the API server so the same auth/session is used.
- Enables single-create and any client to pre-validate without calling the API server directly.

### 2.2 `_build_apps_to_check_for_conflicts(services)` (views.py)

**What:** Helper that builds `[ { app_name, app_uris } ]` from the bulk-create services list using the same extraction as `_process_single_service_background` (question_answers `85` or `service_name`).

**Why:**

- Bulk create (multiple + select all) needs to run the same validation **before** creating the task.
- Reusing the same extraction logic keeps behaviour consistent with the actual create path.

### 2.3 Pre-validate in `bulk_add_custom_app` (views.py)

**What:** After resolving the list of services (from payload or select_all_query), the view builds `apps_to_check`, POSTs to the API server `validate_uri_conflicts`, and if the response is `valid: false`, returns immediately with `api_response: 10` and `api_message` set to the first error (e.g. conflicting domain). No task is created and no apps are created.

**Why:**

- Bulk and select-all flows get the same validation errors **before** any work starts, so the user sees one clear error and no partial creation.
- Validation is still done only in one place (API server `CustomAppsValidation`).

### 2.4 URL route (urls.py)

**What:** New route for `validate_custom_app_conflicts`.

**Why:**

- Exposes the proxy so the frontend can call `/admin/application/validate_custom_app_conflicts`.

---

## 3. Frontend (Angular)

### 3.1 `processBulkCreateResponse` – handle pre-validation error (catalog-apps.state.ts)

**What:** When the bulk-create response has `api_response === 10` and `api_message` (e.g. from backend pre-validation), the state shows that message via `alertBarService.errorAlert`, patches state with failure, and closes the dialog.

**Why:**

- Backend can return 200 with `api_response: 10` and `api_message` when pre-validation fails (e.g. conflicting domain).
- The UI must treat this as a failure and show the same error to the user instead of treating it as success.

### 3.2 `validateCustomAppConflicts` (custom-apps-data.service.ts)

**What:** New method that POSTs to `/admin/application/validate_custom_app_conflicts` with `{ apps }` and returns an observable of `{ valid, errors }`, normalising `response.data` when present.

**Why:**

- Single-create (and any caller) can run the same validations as create before calling `addCustomApp`.
- One service method for pre-validate keeps the flow clear and testable.

### 3.3 Single-create pre-validate (create-custom-gatelet.component.ts)

**What:** In `createCustomApp()`, before calling `addCustomApp()`, the component calls `validateCustomAppConflicts([{ app_name, app_uris }])`. If `valid === false`, it shows the first error and returns without calling `addCustomApp`; otherwise it proceeds with create.

**Why:**

- User sees validation errors (including “Gatelet X already has a conflicting domain”) before the create request is sent.
- Same validation as create, so no duplicate rules.

### 3.4 GateletsUrl / customGateletUrl

**What:** No new URL constant was added in the store state for bulk create (it already uses `BULK_CREATE_CUSTOM_APPS`). The custom-apps data service has `VALIDATE_CUSTOM_APP_CONFLICTS` for the new proxy endpoint.

**Why:**

- Single place for the validate endpoint URL in the service that uses it.

---

## 4. Summary: one validation path

| Layer        | Change                                                                                                                                     | Why                                                             |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| API server   | `validate_uri_conflicts` calls `CustomAppsValidation().is_valid(bundle, request)` with full bundle (name, uris, is_active, audit_data, id) | Run **all** validations as on create; no second implementation. |
| API server   | Removed `validate_bulk_create_uri_conflicts`                                                                                               | Avoid maintaining two validation implementations.               |
| Admin portal | Proxy view + pre-validate in bulk_add_custom_app + helper                                                                                  | Frontend and bulk create use same API validation before create. |
| Frontend     | processBulkCreateResponse handles api_response 10 + api_message                                                                            | Show pre-validation errors from bulk create.                    |
| Frontend     | validateCustomAppConflicts + single-create pre-validate                                                                                    | Single create uses same validations before submit.              |

All custom app validations (name, URIs, conflicts, audit_data) are defined only in `CustomAppsValidation`; create and pre-validate both use it.

---

## 5. UI test cases added

- **catalog-apps.state.spec.ts**

  - `BulkCreateCustomGatelets`: **"should handle pre-validation error (api_response 10 and api_message) from bulk create"** – when the backend returns `api_response: 10` and `api_message` (e.g. conflicting domain), the state shows the error via `errorAlert`, closes the dialog with `false`, and does not treat the response as success.

- **custom-apps-data.service.spec.ts**

  - **validateCustomAppConflicts**:
    - **"should POST to validate endpoint and return valid: true when no errors"** – calls the validate URL with `{ apps }` and returns `{ valid: true }`.
    - **"should return valid: false and errors when validation fails"** – returns `{ valid: false, errors }` with the first error message.
    - **"should unwrap response.data when API returns wrapped shape"** – normalises `{ ok: true, data: { valid: true } }` to `{ valid: true }`.

- **create-custom-gatelet.component.spec.ts**
  - **"should call validateCustomAppConflicts first and not addCustomApp when validation fails"** – when validate returns `{ valid: false, errors: [...] }`, `addCustomApp` is not called and `errorAlert` is called with the first error (e.g. conflicting domain).
  - Existing `createCustomApp` tests now mock `validateCustomAppConflicts` to return `{ valid: true }` so the flow continues to `addCustomApp`.

**Run UI tests (from app-new-gen):**

```bash
npm run test
# or for specific libs:
npx nx test store-ng --no-cache
npx nx test shared-ui-common --no-cache
```
