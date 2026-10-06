# Device monitor Logic App

This package builds the polling workflow. Azure deployment and the remaining infrastructure are left for you to configure.

## Files

- [workflow.json](workflow.json): Consumption Logic App Code View JSON.
- [azuredeploy.json](azuredeploy.json): ARM template; creates only a **disabled** Consumption Logic App with a system-assigned managed identity.
- [checkpoint.initial.json](checkpoint.initial.json): initial state; upload once as `checkpoint.json`.
- [review.md](review.md): generated structural review and workflow diagram.

## What is implemented

1. Run every minute with one workflow run at a time.
2. Read the persisted checkpoint from Blob Storage.
3. On first use, enumerate existing devices and remember their object IDs without generating detections.
4. Follow every Graph delta page sequentially.
5. Ignore removed objects and IDs already observed.
6. For each newly observed ID after baseline, create `<object-id>.json` in the detections container.
7. Save the final delta link and known IDs only when processing completes successfully.

Authentication uses managed identity for both Graph and Storage, without a client secret or interactive sign-in. Graph requires the **Device.Read.All application role**. Azure subscription Reader/Contributor does not grant Graph access. See [Microsoft's managed identity authentication guidance](https://learn.microsoft.com/en-us/azure/logic-apps/authenticate-with-managed-identity) and [Graph device delta permissions](https://learn.microsoft.com/en-us/graph/api/device-delta?view=graph-rest-1.0).

## Deploy later through the Azure portal

1. Open **Deploy a custom template > Build your own template in the editor**.
2. Load `azuredeploy.json` and save.
3. Select your subscription, resource group and location. Enter the storage account name you will use, with containers `device-state` and `device-detections` (or customize their names).
4. Review and create. The Logic App stays **disabled**.
5. Open its **Identity > System assigned** page. Copy the principal/object ID for the permissions step.

Alternatively, create a Consumption Logic App, disable it, enable its system-assigned identity, and paste `workflow.json` into **Logic app code view**. Set the three workflow parameters in the designer. The Code View file itself cannot enforce the resource's disabled state.

## Configure the rest

### Storage

Create a general-purpose v2 storage account with private containers named `device-state` and `device-detections`. They must be reachable from the Consumption Logic App. This package does not set up private networking.

Grant the Logic App identity **Storage Blob Data Contributor**, preferably scoped separately to those two containers. In the portal, use the container's **Access control (IAM) > Add role assignment > Managed identity**.

Upload `checkpoint.initial.json` to the state container as **checkpoint.json**, with content type **application/json**. Do this only before the first run. Never overwrite an active checkpoint with the initial file.

### Microsoft Graph

Have an Entra administrator assign the **Device.Read.All** application role on the Microsoft Graph service principal to the Logic App's managed identity. This is separate from Azure IAM. The workflow already requests a Graph token with audience `https://graph.microsoft.com`.

The Graph role assignment requires a one-time administrative Graph/PowerShell operation; switching the identity on in the GUI alone does not grant it. Follow [Microsoft's managed-identity API permission procedure](https://learn.microsoft.com/en-us/powershell/entra-powershell/grant-api-permissions-managed-identity), selecting `Device.Read.All` and identifying the managed identity by its principal/object ID.

### Alerts and enrichment

Each output blob contains `schemaVersion`, `eventType`, `eventId`, `detectedAtUtc`, and the Graph `device` object. The event ID and filename use the Entra **object ID** (`id`), not the hardware registration's `deviceId`.

Later, connect a separate workflow or consumer to these durable detection files to create Sentinel incidents or notifications. Deduplicate downstream processing by event ID. Retain the files or maintain a processed-event ledger; deleting files can permit recreation when a delta round is replayed.

Owner lookup and incident creation are deliberately extension work. For owner enrichment use `GET /devices/{object-id}/registeredOwners`, follow its pagination, and assess additional permissions if you need full owner profile fields.

The event represents **first observation after baseline**, not a guaranteed device-creation timestamp. Delta payloads can contain partial properties; a downstream consumer can retrieve the full device if required. `trustType` values classify joins as described in the main README.

## Enable and check

1. Confirm the three parameters, Graph role, storage RBAC, and initial checkpoint.
2. Enable the Logic App and run the trigger.
3. Inspect run history. The first successful run should populate the checkpoint's delta link and known IDs without creating detection files.
4. Register a test device after baseline completes. A subsequent successful run should create its detection file.
5. Verify that a property update for the same device does not create a second file.

The workflow is structurally checked locally; it has not been deployed or executed against a tenant. Azure runtime validation and the above integration checks are still required.

## Failure and recovery behavior

- HTTP actions have bounded exponential retries. Authentication/permission failures must be corrected before retrying.
- Missing or malformed state fails the run instead of silently starting over.
- Page or detection failures stop processing and prevent checkpoint advancement. A later run replays from the last committed delta link.
- Detection writes use `If-None-Match: *`. A 412 response means that deterministic file already exists and is accepted as a replay.
- Checkpoint writes use the ETag loaded at the start. A changed checkpoint causes the save to fail rather than overwrite another writer.
- Until-loop exhaustion fails instead of committing a partial round. The limits are 5,000 pages and one hour per run.
- An expired/invalid Graph delta token requires operator recovery. Preserve known IDs, reconcile changes during the gap, and establish a fresh baseline deliberately. Emptying `deltaLink` suppresses all detections during that re-baseline, including devices registered during the gap.
- Use one state and detection container pair per tenant/monitor. Do not run multiple independent monitors against the same checkpoint.

## Scope and limits

The package targets public-cloud endpoints and a single tenant. It keeps all known IDs in a single checkpoint array, so checkpoint size and processing costs grow with the tenant. For large fleets, use a partitioned state store or a worker instead. Deleted IDs are retained to avoid rediscovery alerts.

A one-minute schedule is a polling interval, not a detection-time guarantee. Graph propagation, throttling and run duration affect latency. Devices first appearing during initial baselining are included in the baseline. Existing README cost estimates are illustrative and have not been calculated for this workflow's action count.

References: [Graph delta](https://learn.microsoft.com/en-us/graph/api/device-delta?view=graph-rest-1.0), [loop behavior](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-control-flow-loops), [Blob writes](https://learn.microsoft.com/en-us/rest/api/storageservices/put-blob).
