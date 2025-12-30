# Mobile Services Integration Analysis

## Executive Summary

This document provides a detailed analysis of how mobile services are identified in the audit-compare-services component and how to integrate mobile services display into the catalogue cloud-services component.

## 1. How Mobile Services Are Identified in Audit-Compare-Services

### 1.1 AppType Determination

**Location**: `products/ui/ng/app-new-gen/libs/audit/feature/src/lib/components/audit-compare-services/src/audit-compare-services/audit-compare-services.component.ts`

**Key Mechanism**:
- Mobile services are identified by `appType === AppTypeEnum.MOBILE`
- The component gets appType from `ServiceVisibilityConfig` via `FindAndCompareGridService.getSvc()`
- Default is `AppTypeEnum.CLOUD`

**Code Flow**:
```typescript
// Line 209-226: FindAndCompareGridService.getSvc()
getSvc() {
  return this.auditStateService.getActiveVisibilityConfig$().pipe(
    tap(({ appType }) => {
      this.appType = appType?.toString() || AppTypeEnum.CLOUD.toString();
      this.isAppTypeMobile = this.auditDataService.isAppTypeMobile(appType);
    }),
  );
}

// Line 411-413: AuditDataService.isAppTypeMobile()
isAppTypeMobile(appType = this.getAppType()): boolean {
  return appType === AppTypeEnum.MOBILE;
}
```

### 1.2 Data Sources for Mobile Services

**Primary Data Sources**:

1. **AzMobileMetadata** (Mobile Metadata)
   - **Location**: `azMobileMetadata` from `AzMetadataState`
   - **Structure**: Contains mobile apps indexed by `app_id`
   - **Fields**: 
     - `app_id`: Unique mobile app identifier
     - `service_id`: Links mobile app to cloud service (nullable - can be null for mobile-only apps)
     - `sc_i`: iOS risk score
     - `sc_a`: Android risk score
     - `mos`: Mobile OS (ANDROID, IOS, ANDROID_IOS)
     - `s`: Service ID reference (if linked to cloud service)

2. **Metadata Function**
   - **Location**: `FindAndCompareGridService` constructor (line 85-89)
   - **Usage**: `metadataFn(this.isAppTypeMobile).s` - returns mobile or cloud metadata based on flag
   - **When `isAppTypeMobile = true`**: Returns `azMobileMetadata.s` (mobile apps)
   - **When `isAppTypeMobile = false`**: Returns `azMetadata.s` (cloud services)

3. **Crossfilter Service IDs**
   - **Location**: `CrossfilterState.getAppliedCrossFilter`
   - **Process**: Filters services based on crossfilter selections
   - **Output**: Array of service/app IDs to display

### 1.3 Service Qualification Logic

**Key Insight**: A service can be BOTH cloud and mobile:
- **Cloud Service**: Has `service_id` in `azMetadata`
- **Mobile App**: Has `app_id` in `azMobileMetadata`
- **Hybrid Service**: Has both `service_id` AND `app_id`, where mobile metadata has `service_id` field linking to cloud service

**Identification Logic** (from `AuditDataService.getAppTypeConfig`, line 570-602):
```typescript
getAppTypeConfig(serviceId, azMetadata, appType) {
  const serviceObj = azMetadata.s[serviceId] || {};
  
  // If viewing mobile apps, check if this mobile app has a linked cloud service
  appTypeConfig.isCloud = this.isAppTypeMobile(appType) 
    ? !!azMetadata.s[serviceId].s  // Has service_id reference
    : true;  // Cloud services are always cloud
  
  // Check mobile OS support
  if ('mos' in serviceObj) {
    switch (serviceObj.mos) {
      case MobileOs.ANDROID:
      case MobileOs.IOS:
      case MobileOs.ANDROID_IOS:
        // Mobile app indicators
    }
  }
}
```

### 1.4 Data Flow in Audit-Compare-Services

```
1. Component Initialization
   ↓
2. FindAndCompareGridService.getSvc()
   ↓
3. Gets ServiceVisibilityConfig with appType
   ↓
4. Sets isAppTypeMobile flag based on appType
   ↓
5. Subscribes to serviceDataGrid BehaviorSubject
   ↓
6. Combines multiple observables:
   - serviceIds (from CrossfilterState)
   - tenantApps
   - metadataFn (azMetadata or azMobileMetadata based on isAppTypeMobile)
   - tenantTags
   - serviceTags
   - tenantCustomApps
   ↓
7. Maps serviceIds to gridData using appropriate metadata
   ↓
8. For mobile apps, adds mobileRiskScore (iOS/Android scores)
   ↓
9. Emits { appType, gridData } to serviceDataGrid
   ↓
10. Component receives and displays services
```

## 2. Current Materialized View Structure

### 2.1 Catalogue Services MV

**Location**: `common/db/audit_utils.py:2790` - `create_materialized_combined_collection()`

**Current Data Sources**:
1. `elastica_apparazziservices` - Cloud services metadata
2. `tenant_db.catalogue_services` - Tenant-specific service data
3. `tenant_db.tenant_discoveryservices` - Tags mapping
4. `tenant_db.tenant_tags` - Tag definitions

**Current Fields in MV**:
- `service_id` (from apparazzi)
- `service_name`
- `is_research_completed`
- `brr` (from audit API or fallback)
- `categories`
- `tags`
- `is_active`, `securlet_state`, `gatelet_state`
- `is_custom`
- **NO mobile-specific fields currently**

### 2.2 Missing Mobile Data

The materialized view **does NOT currently include**:
- `app_id` (mobile app identifier)
- `mobileRiskScore` (iOS/Android scores)
- `is_mobile` flag
- Mobile-specific metadata

## 3. Integration Strategy

### 3.1 Backend Changes Required

#### Option A: Extend Materialized View (Recommended)

**Modify `create_materialized_combined_collection()`** to include mobile data:

1. **Add Mobile App Lookup**:
   ```python
   # Query mobile apps from elastica_admin
   mobile_apps_cursor = elastica_admin.elastica_mobileservices.find(
       {}, 
       {
           '_id': 0, 
           'app_id': 1, 
           'app_name': 1, 
           'service_id': 1,  # Links to cloud service
           'android': 1,
           'ios': 1,
           'total_score': 1,
       }
   )
   mobile_apps = {
       doc['app_id']: doc for doc in mobile_apps_cursor
   }
   
   # Create mapping: service_id -> [app_ids]
   service_to_mobile_apps = {}
   for app_id, app_doc in mobile_apps.items():
       service_id = app_doc.get('service_id')
       if service_id:
           service_id_str = str(service_id)
           if service_id_str not in service_to_mobile_apps:
               service_to_mobile_apps[service_id_str] = []
           service_to_mobile_apps[service_id_str].append({
               'app_id': app_id,
               'app_name': app_doc.get('app_name'),
               'ios_score': app_doc.get('ios', {}).get('score'),
               'android_score': app_doc.get('android', {}).get('score'),
           })
   ```

2. **Add Mobile Fields to Combined Document**:
   ```python
   combined_doc.update({
       # ... existing fields ...
       
       # Mobile app data
       'mobile_apps': service_to_mobile_apps.get(str(service_id), []),
       'is_mobile': len(service_to_mobile_apps.get(str(service_id), [])) > 0,
       'mobile_ios_score': mobile_apps[app_id]['ios']['score'] if app_id in mobile_apps else None,
       'mobile_android_score': mobile_apps[app_id]['android']['score'] if app_id in mobile_apps else None,
   })
   ```

3. **Handle Mobile-Only Apps** (apps without service_id):
   ```python
   # After processing catalogue services, add mobile-only apps
   for app_id, app_doc in mobile_apps.items():
       service_id = app_doc.get('service_id')
       if not service_id:  # Mobile-only app
           combined_doc = {
               'service_id': None,
               'app_id': app_id,
               'service_name': app_doc.get('app_name'),
               'is_mobile': True,
               'is_cloud': False,
               # ... other mobile-specific fields
           }
           batch.append(combined_doc)
   ```

#### Option B: Query Mobile Data Separately

Keep MV as-is, query mobile data on-demand from frontend (less efficient but simpler).

### 3.2 Frontend Changes Required

#### 3.2.1 Add Mobile Services Tab Component

**Create**: `mobile-services.component.ts` (similar to `cloud-services.component.ts`)

**Key Differences**:
1. **Query Filter**: Add `is_mobile: true` or `app_id: exists` to DSL query
2. **Columns**: Use mobile-specific columns (iOS Rating, Android Rating instead of BRR)
3. **Data Source**: Query same `catalogue_services_mv` but with mobile filter

**Example Query**:
```typescript
// In mobile-services.component.ts
buildQuery() {
  let query = '* where is_mobile == true';  // Filter for mobile services
  
  if (this.search.query) {
    query += ` and ${this.search.query}`;
  }
  
  // ... rest of query building
}
```

#### 3.2.2 Update Cloud Services Component

**Option 1**: Add tab switching logic to show mobile services in same component
**Option 2**: Create separate mobile-services component and route to it

**Recommended**: Option 2 (separate component) for cleaner separation

#### 3.2.3 Column Configuration

**Mobile Services Columns** (from audit-compare-services, line 668-691):
```typescript
if (isAppTypeMobile) {
  this.columns.unshift(
    {
      field: ColumnField.IOS_RATING,
      header: 'iOS',
      sort_key: 'mobileRiskScore.ios',
      // ...
    },
    {
      field: ColumnField.ANDROID_RATING,
      header: 'Android',
      sort_key: 'mobileRiskScore.android',
      // ...
    },
  );
}
```

**Cloud Services Columns** (current):
```typescript
{
  field: ColumnField.RATING,
  header: 'Rating',
  sort_key: 'scoreObj.brr',
  // ...
}
```

### 3.3 Data Transformation

**Add Mobile Risk Score Calculation**:
```typescript
// In CloudServicesDataTransformationService or new MobileServicesDataTransformationService
transformMobileService(service: any): CloudServicesDashboard {
  return {
    ...service,
    mobileRiskScore: {
      ios: service.mobile_ios_score || 0,
      android: service.mobile_android_score || 0,
    },
    scoreObj: {
      brr: RiskScoreUtilService.getLowestAppBrrValue(
        service.mobile_ios_score,
        service.mobile_android_score
      ),
      isUnResearched: !service.is_research_completed,
    },
  };
}
```

## 4. Implementation Steps

### Phase 1: Backend - Extend Materialized View
1. Modify `create_materialized_combined_collection()` to include mobile apps
2. Add mobile fields to combined documents
3. Handle mobile-only apps (without service_id)
4. Test MV creation with mobile data

### Phase 2: Backend - API Updates
1. Ensure DSL parser supports `is_mobile` field queries
2. Update filter counts API to include mobile service counts
3. Test queries with mobile filters

### Phase 3: Frontend - Mobile Services Component
1. Create `mobile-services.component.ts` (copy from cloud-services)
2. Update column configuration for mobile
3. Add mobile-specific filters
4. Update action bar for mobile context
5. Add mobile service comparison support

### Phase 4: Frontend - Integration
1. Add mobile services tab to catalogue-lists component
2. Update routing for mobile services tab
3. Share common functionality between cloud and mobile components
4. Update comparison facade to handle mobile services

### Phase 5: Testing
1. Test mobile services display
2. Test filtering mobile services
3. Test mobile service comparison
4. Test hybrid services (both cloud and mobile)

## 5. Key Data Structures

### 5.1 Mobile Service in Metadata
```typescript
{
  app_id: string;
  app_name: string;
  service_id?: string;  // Links to cloud service if exists
  sc_i: number;  // iOS score
  sc_a: number;  // Android score
  mos: MobileOs;  // ANDROID | IOS | ANDROID_IOS
  s?: number;  // Service ID reference (if linked)
}
```

### 5.2 Combined Service in MV (Proposed)
```typescript
{
  service_id: number;
  service_name: string;
  is_mobile: boolean;
  is_cloud: boolean;
  mobile_apps: Array<{
    app_id: string;
    app_name: string;
    ios_score: number;
    android_score: number;
  }>;
  brr: number;  // Cloud BRR
  mobile_ios_score?: number;
  mobile_android_score?: number;
  // ... other existing fields
}
```

## 6. Important Considerations

### 6.1 Service Can Be Both Cloud and Mobile
- A service with `service_id` in mobile metadata is linked to a cloud service
- These should appear in BOTH cloud and mobile tabs
- Use `is_mobile` and `is_cloud` flags to filter appropriately

### 6.2 Mobile-Only Apps
- Some mobile apps don't have `service_id` (mobile-only)
- These should ONLY appear in mobile services tab
- Need special handling in MV creation

### 6.3 Performance
- Materialized view should be refreshed when mobile apps are updated
- Consider indexing `is_mobile` field for query performance
- Mobile apps lookup should be efficient (use dictionaries/maps)

### 6.4 Backward Compatibility
- Existing cloud services queries should continue to work
- Mobile fields should be optional/nullable
- Default `is_mobile: false` for existing services

## 7. References

**Key Files**:
- `products/ui/ng/app-new-gen/libs/audit/feature/src/lib/components/audit-compare-services/src/audit-compare-services/audit-compare-services.component.ts`
- `products/ui/ng/app-new-gen/libs/audit/common/src/lib/services/audit-find-and-compare/find-and-compare-grid.service.ts`
- `products/ui/ng/app-new-gen/libs/audit/common/src/lib/services/audit-data.service.ts`
- `products/ui/ng/app-new-gen/libs/catalogue/src/lib/catalogue-lists/components/cloud-services/cloud-services.component.ts`
- `common/db/audit_utils.py:2790` (create_materialized_combined_collection)

**Key Enums**:
- `AppTypeEnum.MOBILE` / `AppTypeEnum.CLOUD`
- `MobileOs.ANDROID` / `MobileOs.IOS` / `MobileOs.ANDROID_IOS`

**Key Services**:
- `FindAndCompareGridService` - Handles service data grid with appType filtering
- `AuditDataService` - Provides metadata and appType utilities
- `CloudServicesListState` - Manages catalogue services state











