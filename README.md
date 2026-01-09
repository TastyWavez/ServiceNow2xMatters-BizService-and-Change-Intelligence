# ServiceNow to xMatters: Business Service (cmdb_ci_service) + Change Intelligence Sync (change_request)

This repository contains ServiceNow update sets that synchronize:

1. **CMDB Business Services** (`cmdb_ci_service`) → **xMatters Services**
2. **Change Requests** (`change_request`) → **xMatters Change Intelligence change records** 

These changes enable Major Incident Managers, SREs and Devops teams to do more with services in xMatters and ServiceNow. If you’re using the Everbridge Flow Designer (EBFD) app in ServiceNow alongside xMatters, you'll benefit from shared context by having the same service names, ownership, and change history show up in the places where people are actively working on responding to and resolving incidents. With these changes you can bridge the gap between impacted services and ownership bringing clarity to Incident Commanders when delegating resolver responsibilities. Services can be connected to xMatters StatusPages and can be updated throughout the incident lifecycle from initial impact to resolution  all while keeping stakeholders informed.

What you get by syncing Services and Changes (CMDB → xMatters)

- A service catalog that incident workflows can take advantage of. In xMatters Incident Management, impacted services can be connected to an incident during initiation so the incident starts with the appropriate impacted service context. 
- Services can also be notified and will engage the responsible support group. Allowing Incident Commanders to focus on "what" is impacted rather than "who" do I need to notify. 
- The service record in xMatters links back to the ServiceNow CI, responders can pivot straight into the CMDB record without hunting by clicking the related service link in the xMatters service view. 
- Change context allows teams to review recent changes to identify potential root causes for an incident. 

---

## Update Sets

### 1) Main Update Set
**File:** `xMattersServiceCenter.xml`

**Includes:**
- Script Includes:
  - `xmApiClient`, `xm_logger`
  - `serviceBatchSync` (ServiceNow services → xMatters services)
  - `changeBatchSync` (ServiceNow change_request → xMatters /changes)
- Business Rules:
  - `xM Service Sync - Insert/Update` (cmdb_ci_service)
  - `xM Service Sync - Delete` (cmdb_ci_service)
  - `xMatters Change Intelligence` (change_request)
- Event Registrations:
  - `x_xma_eb_fd.service.sync`
  - `x_xma_eb_fd.service.delete`
  - `x_xma_eb_fd.change.sync`
- Script Actions:
  - `xM Service Sync`, `xM Service Delete`
  - `xM Change Sync`, `xM Change Sync create`
- App table:
  - `x_xma_eb_fd_xm_change_map` (record keeping, which ensures changes only sync once when closed.)
- System property:
  - `x_xma_eb_fd.service_and_change_sync_credential_name` (default: `xMattersServiceSync` update to use a configured credential from the EBFD application.)
- Cross-scope privileges needed by the app (included in update set):
  - `GlideRecord.setWorkflow`
  - Read `change_request`
  - Write `cmdb_ci_service`

### 2) Service batch job (optional)
**File:** `xMatters_Service_Batch_Sync.xml`

Creates scheduled job:
- `xMatters Service Batch Sync` (Run Type: **on_demand** but can be set to run at a specified interval (not recommended))

### 3) Change batch queue job (optional)
**File:** `xMatters_Change_Intelligence_Batch_Queue.xml`

Creates scheduled job:
- `xMatters Change Intelligence Batch Queue` (Run Type: **on_demand** but can be set to run at a specified interval (not recommended))

---

## Prerequisites

### ServiceNow
- Tables used:
  - `cmdb_ci_service` (Business Services)
  - `change_request`
- App scope:
  - Update sets are in scope of Everbridge Flow Designer ServiceNow application **`x_xma_eb_fd`** thus the EBFD application must be installed in ServiceNow. 
  - Network connectivity from ServiceNow MID/server-side outbound to xMatters API

### xMatters
- xMatters tenant with:
  - **Services** 
  - **Change Intelligence** 
- API access and credentials/token configured

---

## Installation (Import Update Sets)

Import and commit update sets in this order:

1. **Core:** `xMattersServiceCenter.xml`
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

### 2) Check group ownership mapping works (recommended)

`serviceBatchSync` attempts to set `ownedBy.targetName` on xMatters services using the ServiceNow group name (e.g., `Software`).  
If the group does not exist in xMatters the service will exist but not be owned in xMatters, it will be orphaned.

For the best results, ensure the ServiceNow groups you want to map as xMatters service owners exist and are **ACTIVE** in xMatters. 

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
Default in XML is **on_demand** and queues events for recently closed changes, adjust 'lookbackDays' and 'maxToQueue' variables as needed:
```js
new x_xma_eb_fd.changeBatchSync().runBatch({
  lookbackDays: 100,
  maxToQueue: 1000,
  dryRun: false
});
```
---

## Testing

Use the steps below to validate the **Scheduled Jobs** and **Business Rules** included in this integration.

---

### Scheduled Jobs

#### xMatters Service Batch Sync

**Testing:** Bulk-sync all *Production* Business Services (`cmdb_ci_service`) from ServiceNow to xMatters.

1. In ServiceNow, go to **System Definition → Scheduled Jobs**.
2. Open **xMatters Service Batch Sync**.
3. Click **Execute Now**.
4. Open **System Logs → All** and confirm the run completed successfully (no REST/auth/payload errors).
5. In xMatters, confirm Services exist/updated for the expected Production Business Services.

**Expected results**
- All qualifying Business Services are created/updated in xMatters.

---

#### xMatters Change Intelligence Batch Queue

**Testing:** Backfill closed Change Requests that may have missed real-time processing.

1. In ServiceNow, go to **System Definition → Scheduled Jobs**.
2. Open **xMatters Change Intelligence Batch Queue**.
3. Click **Execute Now**.
4. Open **System Logs → All** and confirm changes were processed successfully.
5. In xMatters, confirm closed Change Requests appear as Change Intelligence records.

**Expected Results**
- Closed Changes are synced without duplicates.
  
---

### Business Rules

#### Business Service Insert / Update

**Table:** `cmdb_ci_service`  
**Condition:** `used_for = Production`

1. Create a new Business Service with **Used for = Production**.
2. Save the record.
3. Verify a corresponding Service is created in xMatters.
4. Update a mapped field (e.g., **Name** or ownership/support group if mapped).
5. Verify the update is reflected in xMatters.

**Expected results**
- Only Production Business Services trigger synchronization.

---

#### Business Service Delete

**Table:** `cmdb_ci_service`

1. Delete a Business Service that meets the sync condition (e.g., **Used for = Production**).
2. Review **System Logs → All** to confirm the delete flow executed successfully.
3. Verify the corresponding Service is removed (or deactivated, depending on implementation) in xMatters.

**Expected results**
- Deleted Business Services are removed/deactivated in xMatters.

---

#### Change Request Close

**Table:** `change_request`  
**Trigger:** Change transitions to a closed state (e.g., `closed_at` populated)

1. Create a new Change Request.
2. Transition the Change to a closed state.
3. Verify a Change Intelligence record appears in xMatters for the closed Change.
4. Re-open the Change and close it again.
5. Verify the integration does **not** create a duplicate Change Intelligence record.

**Expected results**
- Each Change is synced once upon closure.
- No duplicates are created.

---

### Validation Checklist

- Scheduled jobs execute successfully via **Execute Now**
- Business Rules trigger only under expected conditions
- No duplicate Services created in xMatters
- No duplicate Change Intelligence records created
- Errors (if any) are visible and actionable in ServiceNow logs

