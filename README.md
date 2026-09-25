# Immediate Detection of New Entra Devices

Near-real-time monitoring for newly registered, Microsoft Entra joined, or hybrid joined devices without waiting for Microsoft Sentinel log ingestion.

## Recommended approach

Use a **Consumption Logic App** that runs every minute and polls the Microsoft Graph device delta endpoint:

```http
GET https://graph.microsoft.com/v1.0/devices/delta
```

- **Expected detection time:** Usually around 1-2 minutes.
- **Required permission:** `Device.Read.All` application permission with administrator consent.
- **Main benefit:** Detects creation of the Entra device object without waiting for `AuditLogs` to reach Log Analytics.

## How it works

1. A Logic App recurrence trigger runs every minute.
2. The workflow calls the saved Microsoft Graph `/devices/delta` URL.
3. It identifies newly created devices and ignores deleted objects.
4. It retrieves the registered owner and other device properties.
5. It creates an alert or Microsoft Sentinel incident when a new device is found.
6. It saves the latest `@odata.deltaLink` for the next run.

## Implementation requirements

- Run the initial `/devices/delta` request and follow every `@odata.nextLink` until Graph returns the final `@odata.deltaLink`.
- Store the final `@odata.deltaLink` in Azure Table Storage, Blob Storage, or another persistent location.
- During each scheduled run, call the previously saved delta URL rather than starting a full device query.
- Follow pagination if Graph returns another `@odata.nextLink`.
- Save the latest `@odata.deltaLink` only after the full response has been processed successfully.
- Objects containing `@removed` represent deleted devices and should not generate new-device alerts.
- Keep known device IDs or check `registrationDateTime` so device property updates are not mistaken for new registrations.
- Retrieve the registered owner using:

```http
GET https://graph.microsoft.com/v1.0/devices/{device-id}/registeredOwners
```

## Device classification

Use the device `trustType` property:

| `trustType` | Classification |
| --- | --- |
| `AzureAd` | Microsoft Entra joined |
| `ServerAd` | Microsoft Entra hybrid joined |
| `Workplace` | Microsoft Entra registered |

## Why Sentinel alone is not immediate

Microsoft Sentinel analytics rules can run only after the event reaches Log Analytics. A near-real-time analytics rule runs every minute with an approximately two-minute built-in delay, but it cannot remove the upstream Microsoft Entra reporting and ingestion delay.

Streaming Entra Audit Logs to Event Hub can reduce downstream processing time and trigger an Azure Function as soon as a record arrives. However, this method still depends on Entra generating and publishing the audit event.

Microsoft Graph currently does not support change-notification subscriptions for Entra `device` objects. Therefore, a direct webhook subscription to `/devices` is not available.

## Available options

| Method | Expected response | Best use |
| --- | --- | --- |
| Graph `/devices/delta` every minute | Around 1-2 minutes | Fastest practical detection of the device object |
| Entra Audit Logs -> Event Hub -> Function | Near real time, but variable | Immediate processing after the audit record is published |
| AuditLogs -> Sentinel NRT rule | Ingestion delay plus approximately 2 minutes | Native Sentinel incident creation and automation |
| Normal scheduled Sentinel rule | Ingestion delay plus rule schedule | Less urgent monitoring |

## Estimated monthly cost

For a **Consumption Logic App**, Microsoft meters trigger, action, and connector executions.

| Schedule | Approximate executions | Estimated cost per tenant |
| --- | --- | --- |
| Every 1 minute | 43,200 runs per month | **$6-$15 per month** |
| Every 5 minutes | 8,640 runs per month | **$2-$4 per month** |
| Ten tenants, separate one-minute workflows | 432,000 runs per month | **$60-$150 per month** |

The exact cost depends on the Azure region, workflow action count, and managed connectors. Azure Table Storage transactions for saving the delta link are negligible; the Logic Apps managed connector calls may cost more than the storage itself.

For many customer tenants, a centralized multitenant Azure Function or worker can be cheaper than deploying a separate polling Logic App for every tenant.

## Recommendation

For one tenant, use a **one-minute Consumption Logic App with `/devices/delta`**. For multiple customer tenants, use a **centralized Azure Function or multitenant worker** that maintains a separate delta link for each tenant and forwards new-device detections to the appropriate Sentinel workspace or automation workflow.

## References

- [Microsoft Graph - device delta API](https://learn.microsoft.com/en-us/graph/api/device-delta?view=graph-rest-1.0)
- [Microsoft Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview)
- [Microsoft Entra log latency](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-log-latency)
- [Microsoft Sentinel NRT analytics rules](https://learn.microsoft.com/en-us/azure/sentinel/near-real-time-rules)
- [Stream Microsoft Entra logs to Event Hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)
- [Azure Logic Apps pricing](https://azure.microsoft.com/en-us/pricing/details/logic-apps/)
