# Enrich Incident - Cloudflare Security Events

## Summary

When a new Microsoft Sentinel incident is created, this playbook is triggered and performs the following actions:

1. Fetches the `IP` entities attached to the incident.
2. If no IP entities are found, the run terminates with a clear error so the failure is visible in the Logic App run history.
3. For each IP address, queries the **Cloudflare GraphQL Analytics API** (`firewallEventsAdaptive`) for firewall/WAF events tied to that IP within a configurable lookback window (default 24 hours).
4. Builds an HTML table of every event found — action taken, matching rule source, country, ASN, request path, and user agent — and adds it as a comment on the incident. If no events are found, it posts a comment saying so.

This playbook is **read-only** — it does not block, challenge, or modify anything in Cloudflare or the incident's status. It exists to give analysts the context needed to decide whether to escalate, run [`Block-IP-Cloudflare`](../Block-IP-Cloudflare/README.md), or dismiss the incident.

## Prerequisites for deployment

1. A Cloudflare **API Token** scoped with `Zone > Analytics > Read` permission for the target zone.
   Create one under **Cloudflare dashboard > My Profile > API Tokens > Create Token**. [Refer here](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
2. The **Zone ID** (`zoneTag`) of the Cloudflare zone to inspect, found on the zone's Overview page in the Cloudflare dashboard. [Refer here](https://developers.cloudflare.com/fundamentals/setup/find-account-and-zone-ids/)
3. Analytic rules mapping the source to an `IP` entity, so incidents carry an IP address for the playbook to enrich — this playbook is most useful on rules where the detection alone doesn't tell the full story, such as `CloudflareCCFWafThreatAllowed`, `CloudflareCCFXSSProbingPattern`, `CloudflareCCFUnexpectedPost`, `CloudflareCCFUnexpectedRequest`, or `CloudflareCCFUnexpectedUrl`.

## Deployment instructions

1. Deploy the playbook by clicking on the "Deploy to Azure" button. This will take you to a Deploy an ARM Template wizard.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FEnrich-Incident-Cloudflare-SecurityEvents%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FEnrich-Incident-Cloudflare-SecurityEvents%2Fazuredeploy.json)

2. Fill in the required parameters for deploying the playbook.

| Parameter | Description |
| --- | --- |
| **Playbook Name** | Name for the Logic App, without spaces (default `Enrich-Incident-Cloudflare-SecurityEvents`). |
| **Cloudflare Zone Id** | The Cloudflare Zone ID (`zoneTag`) for the domain to inspect. |
| **Cloudflare API Token** | The Cloudflare API Token with `Zone > Analytics > Read` permission for the target zone. Stored as a secure (`securestring`) Logic App parameter. |
| **Lookback Hours** | How many hours of Cloudflare firewall event history to query per IP address (default `24`). |

## Post-deployment instructions

### a. Authorize API connection

Once deployment is complete, go under deployment details and authorize the Microsoft Sentinel connection.

1. Click the Microsoft Sentinel connection (`azuresentinel-<PlaybookName>`).
2. Click **Edit API connection**.
3. Sign in with an account that has access to the Sentinel workspace.
4. Click **Save**.

### b. Configurations in Sentinel

- In Microsoft Sentinel, analytic rules should be configured to trigger an incident with `IP` entities.
- Configure an automation rule to trigger this playbook from the analytic rules that would benefit from extra Cloudflare context before an analyst decides on a response.

## Playbook steps explained

### When Azure Sentinel incident creation rule was triggered

Captures the triggering Microsoft Sentinel incident, including its related entities.

### Entities - Get IPs

Retrieves the list of `IP` entities attached to the incident.

### Initialize ConsolidatedResults variable

Creates an empty array used to accumulate every Cloudflare firewall event found, across all IPs, later rendered as the incident comment table.

### Condition - Check if IP entities exist

- If no `IP` entities were found on the incident, the run terminates (`Terminate_if_no_IP_found`) with a `404` error so the missing-entity condition is visible in run history.
- If one or more `IP` entities were found, the playbook proceeds to query Cloudflare for each one.

### For each IP Address

For every IP entity on the incident:

1. **Query Cloudflare firewall events** – sends a GraphQL query to `https://api.cloudflare.com/client/v4/graphql` for `firewallEventsAdaptive`, filtered by `clientIP` and a `datetime_geq`/`datetime_leq` window derived from **Lookback Hours**:
   ```graphql
   query ListFirewallEvents($zoneTag: string, $filter: FirewallEventsAdaptiveFilter_InputObject) {
     viewer {
       zones(filter: { zoneTag: $zoneTag }) {
         firewallEventsAdaptive(filter: $filter, limit: 25, orderBy: [datetime_DESC]) {
           action
           clientAsn
           clientCountryName
           clientIP
           clientRequestPath
           clientRequestQuery
           datetime
           source
           userAgent
         }
       }
     }
   }
   ```
2. **Parse Cloudflare GraphQL response** – parses the JSON response, tolerating an error/empty result so a failed lookup for one IP doesn't abort the loop.
3. **For each firewall event** – appends every returned event (datetime, action, source, country, ASN, request path, user agent) to `ConsolidatedResults`, tagged with the IP address it came from.

### Condition - Check if any events found

- If `ConsolidatedResults` has entries, builds an HTML table (**Create incident HTML table**) and posts it to the incident (**Add comment to incident - events found**).
- If no events were found for any IP within the lookback window, posts a short comment saying so (**Add comment to incident - no events found**), so analysts know the enrichment ran rather than silently doing nothing.

## Notes

- This playbook only reads Cloudflare analytics data — it never modifies firewall rules, WAF settings, or the incident's status/classification.
- Pair it with [`Block-IP-Cloudflare`](../Block-IP-Cloudflare/README.md): run this one first (or in parallel) to give the analyst context, and use the block playbook once a decision is made.
- The GraphQL Analytics API has its own rate limits and data retention window; see the [Cloudflare GraphQL Analytics API docs](https://developers.cloudflare.com/analytics/graphql-api/) for current limits.
