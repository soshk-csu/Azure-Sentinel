# Challenge IP - Cloudflare

## Summary

When a new Microsoft Sentinel incident is created, this playbook is triggered and performs the following actions:

1. Fetches the `IP` entities attached to the incident.
2. If no IP entities are found, the run terminates with a clear error so the failure is visible in the Logic App run history.
3. For each IP address, calls the Cloudflare API to create an **IP Access Rule** in a challenge mode (`managed_challenge` by default) on the configured Cloudflare zone — the visitor is presented with a challenge instead of being blocked outright.
4. Builds an HTML table of the results (IP address, whether it was challenged, the mode applied, the resulting Cloudflare rule ID, and any error detail) and adds it as a comment on the incident.

This is the softer sibling of [`Block-IP-Cloudflare`](../Block-IP-Cloudflare/README.md): use it for detections that are useful signals but prone to false positives (shared NAT gateways, corporate proxies, scanners), where an outright block could disrupt legitimate traffic.

## Prerequisites for deployment

1. A Cloudflare **API Token** scoped with `Zone > Firewall Services > Edit` permission for the target zone.
   Create one under **Cloudflare dashboard > My Profile > API Tokens > Create Token**. [Refer here](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
2. The **Zone ID** of the Cloudflare zone to protect, found on the zone's Overview page in the Cloudflare dashboard. [Refer here](https://developers.cloudflare.com/fundamentals/setup/find-account-and-zone-ids/)
3. Analytic rules mapping the offending source to an `IP` entity. This playbook is best suited to noisy/lower-confidence rules where a hard block risks false positives — for example `CloudflareCCFMultipleUAs`, `CloudflareCCFMultipleErrorsSource`, `CloudflareCCFEmptyUA`, or `CloudflareCCFUnexpectedCountry`.

## Deployment instructions

1. Deploy the playbook by clicking on the "Deploy to Azure" button. This will take you to a Deploy an ARM Template wizard.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FChallenge-IP-Cloudflare%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FChallenge-IP-Cloudflare%2Fazuredeploy.json)

2. Fill in the required parameters for deploying the playbook.

| Parameter | Description |
| --- | --- |
| **Playbook Name** | Name for the Logic App, without spaces (default `Challenge-IP-Cloudflare`). |
| **Cloudflare Zone Id** | The Cloudflare Zone ID for the domain to protect. |
| **Cloudflare API Token** | The Cloudflare API Token with `Zone > Firewall Services > Edit` permission for the target zone. Stored as a secure (`securestring`) Logic App parameter. |
| **Challenge Mode** | The Cloudflare IP Access Rule mode to apply: `managed_challenge` (recommended default), `js_challenge` (legacy JS challenge), or `challenge` (legacy CAPTCHA). |

## Post-deployment instructions

### a. Authorize API connection

Once deployment is complete, go under deployment details and authorize the Microsoft Sentinel connection.

1. Click the Microsoft Sentinel connection (`azuresentinel-<PlaybookName>`).
2. Click **Edit API connection**.
3. Sign in with an account that has access to the Sentinel workspace.
4. Click **Save**.

### b. Configurations in Sentinel

- In Microsoft Sentinel, analytic rules should be configured to trigger an incident with `IP` entities.
- Configure an automation rule to trigger this playbook from analytic rules where a challenge (not a hard block) is the appropriate automated response.

## Playbook steps explained

### When Azure Sentinel incident creation rule was triggered

Captures the triggering Microsoft Sentinel incident, including its related entities and title.

### Entities - Get IPs

Retrieves the list of `IP` entities attached to the incident.

### Initialize ConsolidatedResults variable

Creates an empty array used to accumulate the per-IP challenge result, later rendered as the incident comment table.

### Condition - Check if IP entities exist

- If no `IP` entities were found on the incident, the run terminates (`Terminate_if_no_IP_found`) with a `404` error so the missing-entity condition is visible in run history.
- If one or more `IP` entities were found, the playbook proceeds to challenge each one.

### For each IP Address

For every IP entity on the incident:

1. **Challenge IP in Cloudflare** – calls the Cloudflare API to create a challenge rule:
   ```
   POST https://api.cloudflare.com/client/v4/zones/{CloudflareZoneId}/firewall/access_rules/rules
   Authorization: Bearer {CloudflareAPIToken}

   {
     "mode": "managed_challenge",
     "configuration": { "target": "ip", "value": "<ip address>" },
     "notes": "Challenged automatically by Microsoft Sentinel - Incident: <incident title>"
   }
   ```
2. **Parse Cloudflare response** – parses the JSON response (`success`, `errors`, `result.id`), whether the call succeeded or returned an error status, so a failed challenge (e.g. IP already has a rule) doesn't abort the loop.
3. **Append result to ConsolidatedResults** – records the IP address, whether it was challenged, the mode applied, the Cloudflare rule ID, and any error details.

### Create incident HTML table

Builds an HTML table from `ConsolidatedResults` with columns **IP Address**, **Challenged**, **Mode**, **Rule ID**, and **Details**.

### Add comment to incident

Posts the HTML table to the incident as a comment, so analysts can see exactly which IPs were challenged in Cloudflare and the resulting rule IDs.

## Notes

- Cloudflare returns `success: false` if the IP already has an access rule; this is captured in the comment table under **Details** rather than failing the whole run.
- `managed_challenge` is Cloudflare's current recommended mode — it automatically selects the least intrusive challenge for the visitor. Use `js_challenge` or `challenge` only if you have a specific reason to pin the legacy behavior.
- To challenge IPs account-wide instead of on a single zone, swap the URI's `zones/{CloudflareZoneId}` path segment for `accounts/{account_id}` per the [Cloudflare IP Access Rules API](https://developers.cloudflare.com/waf/tools/ip-access-rules/).
