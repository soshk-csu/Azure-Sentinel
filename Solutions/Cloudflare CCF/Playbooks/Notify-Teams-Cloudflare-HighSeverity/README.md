# Notify Teams - Cloudflare High Severity

## Summary

When a new Microsoft Sentinel incident is created, this playbook is triggered and performs the following actions:

1. Fetches the `IP` entities attached to the incident.
2. Builds an adaptive card containing the incident's title, description, severity, status, incident number, affected IP address(es), and a button to open the incident.
3. Posts the card to a configured Microsoft Teams channel.
4. Adds a comment to the incident confirming the notification was sent.

This playbook runs independently of any blocking/challenge action — it exists purely to make sure the right people see high-signal Cloudflare incidents immediately, regardless of whether an automated remediation playbook also fires. Severity/rule targeting is done via the **automation rule** that triggers this playbook, not inside the playbook itself — this keeps the Logic App generic and lets you repoint it at any set of analytic rules without redeploying.

## Prerequisites for deployment

1. The **Team ID** (group ID) and **Channel ID** of the Microsoft Teams channel notifications should be posted to. [Guidance to get IDs](https://learn.microsoft.com/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)
2. Analytic rules mapping the offending source to an `IP` entity, and an automation rule scoping this playbook to the Cloudflare CCF rules that warrant immediate notification — typically higher-severity ones such as `CloudflareCCFWafThreatAllowed` (High).

## Deployment instructions

1. Deploy the playbook by clicking on the "Deploy to Azure" button. This will take you to a Deploy an ARM Template wizard.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FNotify-Teams-Cloudflare-HighSeverity%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FAzure%2FAzure-Sentinel%2Fmaster%2FSolutions%2FCloudflare%2520CCF%2FPlaybooks%2FNotify-Teams-Cloudflare-HighSeverity%2Fazuredeploy.json)

2. Fill in the required parameters for deploying the playbook.

| Parameter | Description |
| --- | --- |
| **Playbook Name** | Name for the Logic App, without spaces (default `Notify-Teams-Cloudflare-HighSeverity`). |
| **Teams Team Id** | The Microsoft Teams Team ID (group ID) that owns the target channel. |
| **Teams Channel Id** | The Microsoft Teams Channel ID to post notifications to. |

## Post-deployment instructions

### a. Authorize API connections

Once deployment is complete, go under deployment details and authorize both connections.

1. Click the Microsoft Sentinel connection (`azuresentinel-<PlaybookName>`), click **Edit API connection**, sign in, click Save.
2. Click the Microsoft Teams connection (`teams-<PlaybookName>`), click **Edit API connection**, sign in with an account that has access to the target Team, click Save.

### b. Configurations in Sentinel

- In Microsoft Sentinel, analytic rules should be configured to trigger an incident with `IP` entities.
- Configure an automation rule to trigger this playbook from the Cloudflare CCF analytic rules that should page or notify the team — this is where you control which severities/rules cause a notification, not the playbook.

## Playbook steps explained

### When Azure Sentinel incident creation rule was triggered

Captures the triggering Microsoft Sentinel incident, including its related entities, title, description, severity, status, and incident URL.

### Entities - Get IPs

Retrieves the list of `IP` entities attached to the incident.

### Select IP Addresses

Maps the IP entity objects down to a plain array of address strings.

### Compose - IP Address List

Joins the address array into a single comma-separated string (`N/A` if no IPs were found), used in the card's fact list.

### Compose - Teams adaptive card

Builds an Adaptive Card (schema 1.4) with:
- A title/header block.
- The incident title and description.
- A fact set: **Severity**, **Status**, **IP Address(es)**, **Incident Number**.
- An **Open Incident** button linking to the incident's URL in Sentinel.

### Post adaptive card in a chat or channel

Posts the card to the configured Team/Channel via the Microsoft Teams connector (`/v1.0/teams/conversation/adaptivecard/poster/Flow bot/location/Channel`).

### Add comment to incident

Adds a short comment to the incident confirming a Teams notification was sent, so analysts reviewing the incident later can see it was raised.

## Notes

- This playbook does not take any remediation action — pair it with [`Block-IP-Cloudflare`](../Block-IP-Cloudflare/README.md) or [`Challenge-IP-Cloudflare`](../Challenge-IP-Cloudflare/README.md) via separate automation rules if you also want an automated response.
- To notify on every Cloudflare CCF incident rather than just high-severity ones, simply broaden the automation rule's conditions — no playbook change needed.
