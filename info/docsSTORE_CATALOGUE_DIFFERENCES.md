# Store Page vs Catalogue (Cloud Services) – Test Scenarios

This document lists all operations and checks performed by the **Store page** (AJS `overview.html` + related controllers/directives/services) and maps them to the **Catalogue page** (Cloud Services component). Use the **Pass/Fail** column to track test results.

---

## 1. Page Structure & Sections

| #   | Store Page Behavior                                                                                    | Catalogue Equivalent                                                                  | Pass/Fail | Notes                                                                    |
| --- | ------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------ |
| 1.1 | Three sections: Elastica Apps, Securlets, Gatelets                                                     | Single table; no separate Elastica/Securlets/Gatelets sections                        |           | Catalogue is unified list; filter by stat boxes/quick filters            |
| 1.2 | Permission-gated sections: `store-panel-elastica-apps`, `store-panel-securlets`, `store-panel-gateway` | Permissions: `create.custom.gatelet`, `store.tile.gateway`, `audit.service.info.tags` |           | Different permission model; verify catalogue respects feature visibility |
| 1.3 | Subtitle: "Apps, Securlets & Gatelets"                                                                 | Tab/context: Cloud Services list                                                      |           | UI copy only                                                             |
| 1.4 | Loading overlay when `!apps`                                                                           | `loading$` / `statsLoading$` with progress bar and spinner                            |           |                                                                          |

---

## 2. Search & Filtering

| #   | Store Page Behavior                                                                         | Catalogue Equivalent                                                                                                           | Pass/Fail | Notes                                                               |
| --- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------- | ------------------------------------------------------------------- |
| 2.1 | Global search: `filterText` (search input), filters Elastica/Securlets/Gatelets by app name | `query` + `searchQuery()`; DSL query with `is_cloud = true`; quick filters + custom filter                                     |           | Store is client-side filter on name; Catalogue uses server-side DSL |
| 2.2 | `appFilter(app)`: filter by `app_name` or `cloudServiceMapping(appName)` vs `filterText`    | Search applied via `buildQuery()` and `GetCloudServices(query)`                                                                |           |                                                                     |
| 2.3 | Analytics on search: `analytics-event="store:main:search-overview"`                         | Not visible in cloud-services component                                                                                        |           | May be in parent or analytics layer                                 |
| 2.4 | No quick filters / stat boxes on Store overview                                             | Quick filters, stat boxes (All Services, Discovered, Active Securlets, Active Gatelets, Active Custom Gatelets), custom filter |           | Catalogue has more filter options                                   |
| 2.5 | Stat box click filters list (Catalogue only)                                                | `onStatBoxClick(box)`, `mergeStatBoxFilterWithQuery()`                                                                         |           | Store has no stat boxes                                             |

---

## 3. Elastica Apps (Store Only)

| #   | Store Page Behavior                                                                       | Catalogue Equivalent                                                                                         | Pass/Fail | Notes                                                                      |
| --- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ | --------- | -------------------------------------------------------------------------- |
| 3.1 | Show Elastica Apps panel when `apps.elasticaApps.length`                                  | No dedicated Elastica section; services from catalogue may include cloud + gatelet data                      |           | Elastica apps are "Symantec CloudSOC" apps; catalogue shows cloud services |
| 3.2 | Tiles: `orderBy:['popularity', '_id']`, `limitTo: appsViewConfig.elasticaApps.appsToShow` | Table sort (e.g. default BRR, service_id), pagination (limit/offset)                                         |           | Different data source and sort                                             |
| 3.3 | Link to app details: `#/appdetails/{{ app._id }}`                                         | Row menu "View Details" / securlet details navigation; APP_STORE_APP_DETAILS / APP_DISCOVERY_SERVICE_DETAILS |           |                                                                            |
| 3.4 | Tile component: `drv-app-prime-tile`                                                      | Table row with icon, name, BRR, api_capable, epd_capable, tags, categories, actions                          |           |                                                                            |
| 3.5 | "See all" / "See less" for Elastica: `toggleSeeAllState(appEnums.ELASTICA, …)`            | Pagination (e.g. 10/20/50 rows)                                                                              |           |                                                                            |

---

## 4. Securlets

| #   | Store Page Behavior                                                                                         | Catalogue Equivalent                                                                                                           | Pass/Fail | Notes                                                       |
| --- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | --------- | ----------------------------------------------------------- |
| 4.1 | Securlets panel when `apps.apiApps.length`; permission `store-panel-securlets`                              | Securlet status in table column `api_capable`; status labels (Enable, Configure, Manage, Deactivate, etc.)                     |           |                                                             |
| 4.2 | **Licensed/Used usage:** Total Licensed (users + GB/Day), Currently Used (users + GB/Day)                   | Usage Metrics section: `UsageMetricsState`, chart from `meterGroupValue$`, `saasUsagePercentage$`, `bandwidthUsagePercentage$` |           | Store shows raw counts; Catalogue shows chart + percentages |
| 4.3 | Tiles: `orderBy:['popularity', '_id']`, `filter: appFilter`, `limitTo: apiApps.appsToShow`                  | Server-side sort and pagination                                                                                                |           |                                                             |
| 4.4 | Link: `getDetailPageUrl(app)` → `#/appdetails/{{ app._id }}` when `app._hasDetails`                         | Row menu: Configure/Manage/Enable Securlet, "Securlet Details" → APP_STORE_APP_DETAILS                                         |           |                                                             |
| 4.5 | `drv-app-prime-tile` with `tenant-name`, `primary-domain`                                                   | Account popover when securlet activated: `toggleAccountPopover`, `getSecurletActivationInfo`, `securletDetailsMap$`            |           | Catalogue has account count badge + popover                 |
| 4.6 | Securlet tile actions: Enable, Configure, Manage, Deactivate, Remove (with overlay), Request Access/Pending | Row menu: Enable Securlet, Configure Securlet, Manage Securlet, Deactivate Securlet, Activate Securlet (configure), Details    |           |                                                             |
| 4.7 | Multi-pack / combination checks (e.g. O365 + Yammer), licensed count vs current                             | Catalogue does not show same multi-pack modal on overview; activation flows may be in parent/dialogs                           |           | Verify bulk "Activate Gatelets" and single Securlet actions |
| 4.8 | Analytics: `analytics-event="store:main:securlet-details"` on tile click                                    | Not visible in component                                                                                                       |           |                                                             |
| 4.9 | Add-via-API status notification from route params (`id`, `status`, `apiResponse`, `text`, `email`)          | No direct equivalent in cloud-services component                                                                               |           | May be handled by parent or routing                         |

---

## 5. Gatelets (Full + Custom)

| #    | Store Page Behavior                                                                                                                                                       | Catalogue Equivalent                                                                                                    | Pass/Fail | Notes                                    |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | --------- | ---------------------------------------- |
| 5.1  | Gatelets panel when `apps.gatewayApps.length`; permission `store-panel-gateway`                                                                                           | Gatelet status in column `epd_capable`; stat box "Active Gatelets", "Active Custom Gatelets"                            |           |                                          |
| 5.2  | **Create Custom Apps** button (with Reverse Proxy Beta note); permission `customAppPermission`; `createCustomGatelet()`                                                   | Action bar: "Create Custom App"; `createCustomApp` / `createCustomAppsFromSelected` (select all or multi-select)        |           | Catalogue supports create from selection |
| 5.3  | Reverse Proxy Beta: `drv-permission pid="store-configure-rp"`, tooltip about limited availability                                                                         | No explicit RP Beta callout in cloud-services template                                                                  |           |                                          |
| 5.4  | "See all" / "See less" for Gatelets: `toggleSeeAllState(appEnums.COMBINED_GATEWAY)`                                                                                       | Pagination                                                                                                              |           |                                          |
| 5.5  | Combined list: `apps.combinedApps` (gatewayApps + customApps); orderBy popularity, filter, limitTo                                                                        | Table shows all combined; `allCombinedApps` in state; filter by `is_custom`, `epd_capable`, etc.                        |           |                                          |
| 5.6  | **Predefined gatelet tile** (`drv-app-tile` when `!app.is_generic_customapp`): Activate, Deactivate, Finalize Deactivation, View Details, Edit App (if `display_edit_ui`) | Row menu: View Gatelet Details, Activate Gatelet, Deactivate Gatelet, Edit Gatelet                                      |           |                                          |
| 5.7  | **Custom gatelet tile** (`app.is_generic_customapp`): View Details, Deactivate, Activate, Finalize Deactivation, Edit App, Delete App                                     | Row menu: View Custom Gatelet Details, Activate Custom Gatelet, Deactivate Custom Gatelet, Edit Gatelet, Delete Gatelet |           |                                          |
| 5.8  | Reverse Proxy on tile: states (Available, Enabled, Disabled, Activating, Deactivation Pending); Activate/Deactivate RP only; View RP Details                              | Not visible in catalogue table; may be in gatelet detail/overlay                                                        |           | Store shows RP state on each tile        |
| 5.9  | Mirror Gateway on tile: same state pattern (Available, Enabled, Disabled, Activating, Deactivation Pending); Activate/Deactivate MG only; View MG Details                 | Not visible in catalogue table                                                                                          |           |                                          |
| 5.10 | Scan Request Payload / Edit App: `isTenantScanRequestPayloadEnabled`, `display_edit_ui` (from tenant config GW_GDF_UI_APPS)                                               | Edit Gatelet in row menu when applicable                                                                                |           |                                          |
| 5.11 | Risk score for custom app: `drv-risk-score` with `app.audit_data.brr`                                                                                                     | BRR column in table for all services                                                                                    |           |                                          |
| 5.12 | Tile states: pending_delete, status === 'DP', is_gateway_subscription_requested                                                                                           | Table shows activation status via `getGateletStatus()`; checkbox disabled for `generic_epd === true`                    |           |                                          |
| 5.13 | Analytics: `store:main:gatelets:see-all-or-less`                                                                                                                          | N/A                                                                                                                     |           |                                          |

---

## 6. Data Loading & APIs

| #   | Store Page Behavior                                                                                                                  | Catalogue Equivalent                                                                         | Pass/Fail | Notes                                                                   |
| --- | ------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- | --------- | ----------------------------------------------------------------------- |
| 6.1 | Load: `getTenantAndCustomApps(lang, true)` → `/admin/application/list/tenantandcustomapps`                                           | `LoadAllCombinedApps()`, `GetCloudServices(query)`, `GetFilterCounts()`                      |           | Different endpoints; catalogue uses filter_counts + cloud services list |
| 6.2 | Polling: `pollIntervals.appStoreFetchTenantApps`, callback `pollCombinedResponseHandler`                                             | No polling in cloud-services component; refresh via `refreshDataSources()` / `searchQuery()` |           |                                                                         |
| 6.3 | Session: `userService.getUserSession()` then `fetchTenantConfig('Securlets')` then `initialize(sessionModel)`                        | Session/tenant handled by app shell or parent                                                |           |                                                                         |
| 6.4 | Securlet usage: `subscriptionService.getSecurletUsers()` (licensed/current SaaS/IaaS)                                                | `UsageMetricsActions.LoadUsageMetrics()`, `UsageMetricsState`                                |           |                                                                         |
| 6.5 | Consolidated metadata for custom app create/edit: `preloadMetadata()` → `getApparazziMetadata()`                                     | `getConsolidateMetadata()` (CloudServicesListActions)                                        |           |                                                                         |
| 6.6 | Reverse Proxy status: `reverseProxyDataService.getReverseProxyStatus()`                                                              | Not in cloud-services component                                                              |           |                                                                         |
| 6.7 | Mirror Gateway / Monitoring: `isMirrorGatewayEnabledFromBop`, `isTenantScanRequestPayloadEnabled` from subscription/monitoring model | Not surfaced in catalogue template                                                           |           |                                                                         |
| 6.8 | Event listeners: `CUSTOM_APP_NOTIFIER` → refetch combined apps; `REVERSE_PROXY_ALERT` → set `isReverseProxyConfigured`               | No equivalent in cloud-services component                                                    |           |                                                                         |

---

## 7. Permissions & Feature Flags

| #   | Store Page Behavior                                                                                                                                                  | Catalogue Equivalent                                                                                                                            | Pass/Fail | Notes |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----- |
| 7.1 | `customAppPermission`: from `data.permission_to_add` or `appStateService.isEnabled('create.custom.gatelet')`                                                         | `CataloguePermissions.CREATE_CUSTOM_GATELET` for Create Custom App action                                                                       |           |       |
| 7.2 | `showCustomApps`: from subscription sub_features (CUSTOM_GATELET_SUPPORT_FEATURE)                                                                                    | Create Custom App visibility (and custom gatelet actions)                                                                                       |           |       |
| 7.3 | `isReverseProxyEnabledFromBop`, `isMirrorGatewayEnabledFromBop`                                                                                                      | Passed to store/dialogs; not shown in catalogue table                                                                                           |           |       |
| 7.4 | `isTenantScanRequestPayloadEnabled`                                                                                                                                  | Used in dialogs (e.g. edit gatelet)                                                                                                             |           |       |
| 7.5 | Tile enabled: Elastica – always if not disabled; Securlets – `!appStateService.isDisabled('store.tile.securlets')`; Gatelets – `store.tile.gateway` + `isAuthorized` | `isTileEnabled(app)`: `VIEW_GATELETS` + `isAuthorized()`; `isAuthorized`: permissionToAdd or CREATE_CUSTOM_GATELET or isAuthorizedForActivation |           |       |

---

## 8. Create Custom App Flow

| #   | Store Page Behavior                                                                                                             | Catalogue Equivalent                                                                                                                               | Pass/Fail | Notes                        |
| --- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ---------------------------- |
| 8.1 | Single "Create Custom Apps" button → overlay with modal config (CREATE), consolidated meta, audit enabled, scan request payload | `createCustomApp` emit → parent opens create dialog                                                                                                |           |                              |
| 8.2 | Create Custom App disabled until metadata loaded (`isDisabledCreateAppBtn`)                                                     | Create Custom App button enabled based on permission; no explicit metadata-loaded guard in template                                                |           |                              |
| 8.3 | No "create from selected services" on Store                                                                                     | "Create Custom App" with Select All or multiple selection → `createCustomAppsFromSelected` with service names or `{ selectAllQuery, isSelectAll }` |           | Catalogue extends capability |

---

## 9. Bulk & Selection

| #   | Store Page Behavior                                | Catalogue Equivalent                                                                                                              | Pass/Fail | Notes          |
| --- | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | --------- | -------------- |
| 9.1 | No row selection or bulk actions on Store overview | Table: checkbox per row, "Select All", "Clear Selection"; `selectedCheckBoxServices`, `isAllServicesSelected$`, `selectAllQuery$` |           | Catalogue only |
| 9.2 | N/A                                                | **Activate Gatelets** (bulk): single or multiple selection or "Select All"; emit `activateGatelets(appsData)`                     |           |                |
| 9.3 | N/A                                                | **Tags**: require at least 1 service; emit `manageTags(selectedCheckBoxServices)`                                                 |           |                |
| 9.4 | N/A                                                | **Compare**: require at least 1 service; `comparisonFacadeService`, navigate to Compare tab                                       |           |                |
| 9.5 | N/A                                                | Select All: `SetSelectAll(query)` with current filter query; Clear Selection: `ClearSelectAll()`                                  |           |                |
| 9.6 | N/A                                                | Checkbox disabled for `generic_epd === true`                                                                                      |           |                |

---

## 10. Navigation & Links

| #    | Store Page Behavior                                                                            | Catalogue Equivalent                                                                                                                                               | Pass/Fail | Notes |
| ---- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------- | ----- |
| 10.1 | Elastica/Securlet: `#/appdetails/{{ app._id }}` or `getDetailPageUrl(app)`                     | Row menu "View Details" / "Securlet Details" → `APP_STORE_APP_DETAILS` + cleaned app name; "View Details" → audit `APP_DISCOVERY_SERVICE_DETAIL_PAGE` + service_id |           |       |
| 10.2 | Securlet Configure/Manage: various addl params routes (AWS, GitHub, O365, Workday, Okta, etc.) | `configureSecurlet` emit or `navigationService.navigateToUrl(url, { from_catalog: true })` when `_configurationUrl` / `_managementUrl` / `_activationUrl`          |           |       |
| 10.3 | Query param `q` for pre-filled search                                                          | `route.queryParamMap` → set `query`; `patchUrlWithQuery()` on filter change                                                                                        |           |       |

---

## 11. Row / Tile Actions Summary

| #     | Store (Tile) Action                                                        | Catalogue (Row Menu / Action Bar)                          | Pass/Fail |
| ----- | -------------------------------------------------------------------------- | ---------------------------------------------------------- | --------- |
| 11.1  | Securlet: Enable / Configure / Manage / Deactivate / Remove                | Enable Securlet, Configure, Manage, Deactivate Securlet    |           |
| 11.2  | Securlet: Request Access / Pending                                         | Shown via status; no separate "Request Access" in menu     |           |
| 11.3  | Gatelet: View Details                                                      | View Gatelet Details / View Custom Gatelet Details         |           |
| 11.4  | Gatelet: Activate / Deactivate / Finalize Deactivation                     | Activate Gatelet, Deactivate Gatelet (and custom variants) |           |
| 11.5  | Custom Gatelet: Edit App, Delete App                                       | Edit Gatelet, Delete Gatelet                               |           |
| 11.6  | Predefined Gatelet: Edit App (when display_edit_ui + scan_request_payload) | Edit Gatelet in menu when applicable                       |           |
| 11.7  | Reverse Proxy: Activate/Deactivate/Cancel, View RP Details                 | Not in catalogue row menu                                  |           |
| 11.8  | Mirror Gateway: Activate/Deactivate/Cancel, View MG Details                | Not in catalogue row menu                                  |           |
| 11.9  | View Audit Details                                                         | View Details (audit service details page)                  |           |
| 11.10 | Comments                                                                   | Comments..                                                 |           |
| 11.11 | Manage Tags                                                                | Manage Tags..                                              |           |

---

## 12. Display & UX

| #    | Store Page Behavior                                         | Catalogue Equivalent                                                                                 | Pass/Fail | Notes |
| ---- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | --------- | ----- |
| 12.1 | Tiles with image, name, short description, action button(s) | Table: icon/image, service name, BRR, api_capable, epd_capable, tags, categories, timestamp, actions |           |       |
| 12.2 | Securlet: Processed Log Size (Azure) on prime tile          | Not in catalogue table                                                                               |           |       |
| 12.3 | Beta badge on Securlet tile (`app.in_beta`)                 | Can be added in template if needed                                                                   |           |       |
| 12.4 | Carousel (commented out on Store)                           | N/A                                                                                                  |           |       |
| 12.5 | Column toggle, resizable columns, state storage (session)   | Table: column toggle, resizable, stateKey, stateStorage                                              |           |       |

---

## 13. Potential Gaps (Catalogue vs Store)

- **Separate Elastica Apps section**: Catalogue does not have a dedicated "Elastica Apps" panel; all are in one table with filters.
- **Reverse Proxy / Mirror Gateway on list**: Store shows RP/MG state and actions per tile; Catalogue does not show these in the table (may be in detail/overlay).
- **Add-via-API return notification**: Store shows notification from route params after connector add; Catalogue does not handle this in the component.
- **Polling**: Store polls tenant+custom apps; Catalogue relies on explicit refresh / filter.
- **Permission IDs**: Store uses `store-panel-*` and `store-configure-rp`; Catalogue uses different permission constants—ensure feature parity for gating.
- **Create Custom App**: Catalogue adds "create from selected" (single, multiple, select all); Store only has single create.
- **Licensed/Used text**: Store shows "Total Licensed" and "Currently Used" with users + GB/Day; Catalogue shows Usage Metrics chart—confirm same data and intent.

---

## 14. Suggested Test Execution

1. For each row, run the scenario on **Store** and **Catalogue** and mark **Pass** if behavior is equivalent or intentionally improved; **Fail** if required behavior is missing or wrong.
2. Add **Notes** for environment (permissions, feature flags, tenant config) and any follow-up (e.g. "RP/MG in overlay only").
3. Track **Potential Gaps** (Section 13) as separate test cases or backlog items.

---

_Generated from Store overview.html, overviewCtrl.js, overviewDataService.js, drvAppTile.js, drvAppPrimeTile.js, appTile.html, appPrimeTile.html, and Catalogue cloud-services.component.ts/html, cloud-services-menu.service.ts, cloud-services.config.ts._
