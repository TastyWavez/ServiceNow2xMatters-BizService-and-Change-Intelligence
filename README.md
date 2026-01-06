# ServiceNow → xMatters Service Center + Change Intelligence Sync

This repository contains ServiceNow update sets that synchronize:

1. **CMDB Business Services** (`cmdb_ci_service`) → **xMatters Services**
2. **Change Requests** (`change_request`) → **xMatters Change Intelligence change records** (`POST /changes`)

The integration uses an **eventQueue → Script Action → Script Include** pattern for asynchronous processing, plus optional scheduled batch jobs for backfill/reconciliation.

---

## Contents (Update Sets)

### 1) Core integration (required)
**File:** `xMattersServiceCenter4.xml`

Includes:
- Script Includes:
  - `xmApiClient`, `xm_logger`
  - `serviceBatchSync` (ServiceNow services → xMatters services)
  - `changeBatchSync` (ServiceNow change_request → xMatters /changes)
  - `changeIntelligenceSync` (present but not required by the current BR path)
- Business Rules:
  - `xM Service Sync - Insert/Update` (cmdb_ci_service)
  - `xM Service Sync - Delete` (cmdb_ci_service)
  - `xMatters Change Intelligence` (change_request)
- Event Registrations:
  - `x_xma_eb_fd.service.sync`
  - `x_xma_eb_fd.service.delete`
  - `x_xma_eb_fd.change.sync`
  - `x_xma_eb_fd.change.create` (not used by default BR flow)
- Script Actions:
  - `xM Service Sync`, `xM Service Delete`
  - `xM Change Sync`, `xM Change Sync create`
- App table:
  - `x_xma_eb_fd_xm_change_map` (idempotency map for /changes)
- System property:
  - `x_xma_eb_fd.service_and_change_sync_credential_name` (default: `xMattersServiceSync`)
- Cross-scope privileges needed by the app (included in update set):
  - `GlideRecord.setWorkflow`
  - Read `change_request`
  - Write `cmdb_ci_service`

### 2) Service batch job (optional)
**File:** `xMatters_Service_Batch_Sync.xml`

Creates scheduled job:
- `xMatters Service Batch Sync` (default run type: **on_demand**)

### 3) Change batch queue job (optional)
**File:** `xMatters_Change_Intelligence_Batch_Queue.xml`

Creates scheduled job:
- `xMatters Change Intelligence Batch Queue` (default run type: **on_demand**)

---

## Prerequisites

### ServiceNow
- Tables used:
  - `cmdb_ci_service` (Business Services)
  - `change_request`
- App scope:
  - Objects are delivered in scope **`x_xma_eb_fd`**
- Network connectivity from ServiceNow MID/server-side outbound to xMatters API

### xMatters
- xMatters tenant with:
  - **Services** 
  - **Change Intelligence** 
- API access and credentials/token configured

xMatters API references used by this integration:
- **Services:** Create/modify via `POST /services` and delete via `DELETE /services/{serviceId}` (serviceId can be UUID or name/target name).  
- **Change Intelligence:** Create change records via `POST /changes`. Change records **cannot be modified or deleted** once created.  
  (See xMatters REST API Reference.)  
  - `POST /changes` returns `201 Created` with a Change object.  
  - Change records are immutable once created.  

---

## Installation (Import Update Sets)

Import and commit update sets in this order:

1. **Core:** `xMattersServiceCenter4.xml`
2. **Service batch job (optional):** `xMatters_Service_Batch_Sync.xml`
3. **Change batch job (optional):** `xMatters_Change_Intelligence_Batch_Queue.xml`

Steps (each update set):
1. Navigate to **System Update Sets → Retrieved Update Sets**
2. Click **Import Update Set from XML**
3. Upload the XML file
4. Open the retrieved update set → **Preview Update Set**
5. Resolve any conflicts
6. **Commit Update Set**

> Note: Import may prompt for cross-scope privilege approvals depending on instance policy. The required privileges are included in the core update set.

---

## Configuration

### 1) Configure xMatters credentials (required)

The Script Includes instantiate `xmApiClient` using the system property:

- **Property:** `x_xma_eb_fd.service_and_change_sync_credential_name`
- **Default value:** `xMattersServiceSync`

Ensure a matching credential record exists and is readable by the app.

> The credential record name must match the property value, or update the property accordingly.

### 2) Ensure group ownership mapping works (recommended)

`serviceBatchSync` attempts to set `ownedBy.targetName` on xMatters services using the ServiceNow group name (e.g., `Software`).  
If the group does not exist in xMatters the service will exist but not be owned in xMatters.

Ensure:
- The xMatters groups you want to map exist and are **ACTIVE**

---

## When do Services sync to xMatters?

### A) Dynamic (Business Rule driven)
**Business Rule:** `xM Service Sync - Insert/Update` on `cmdb_ci_service`  
Runs **after insert/update** when:

- `used_for` **is 'Production'**

Behavior:
- Queues event `x_xma_eb_fd.service.sync`
- Script Action `xM Service Sync` calls `serviceBatchSync`:
  - If name changed, it uses rename-aware update
  - Otherwise, it performs a normal upsert (create/update)

**Business Rule:** `xM Service Sync - Delete` on `cmdb_ci_service`  
Runs **before delete**:
- Queues event `x_xma_eb_fd.service.delete` with:
  - `parm1` = old service `name` (used as xMatters identifier)
  - `parm2` = CI sys_id
- Script Action `xM Service Delete` calls `serviceBatchSync.deleteServiceByTargetName()`

### B) Batch (Scheduled Job)
Scheduled job: `xMatters Service Batch Sync` (optional)  
Calls into `serviceBatchSync` to reconcile services in bulk.

---

## When do Changes sync to xMatters?

### A) Dynamic (Business Rule driven)
**Business Rule:** `xMatters Change Intelligence` on `change_request`  
Runs **after update** when:
- `active=true`
- `closed_at` **is not empty**
- AND the script confirms this is the first time it became closed (prevents re-queueing)

Behavior:
- Queues event `x_xma_eb_fd.change.sync` with:
  - `parm1` = change sys_id
  - `parm2` = origin string (e.g., `dynamic`)
- Script Action `xM Change Sync` calls `changeBatchSync.processChangeEvent(current, event)`
- `changeBatchSync` posts to xMatters **`POST /changes`**

### B) Batch (Scheduled Queue Job)
Scheduled job: `xMatters Change Intelligence Batch Queue` (optional)  
Default in XML is **on_demand** and queues events for recently closed changes:

```js
new x_xma_eb_fd.changeBatchSync().runBatch({
  lookbackDays: 100,
  maxToQueue: 200,
  dryRun: false
});
