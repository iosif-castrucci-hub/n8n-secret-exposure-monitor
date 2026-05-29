# Secret Exposure Monitor

A cybersecurity automation project built with **n8n**, **Webhook**, **Google Sheets**, and **Telegram**.

This project detects possible exposed secrets such as API keys, tokens, passwords, private keys, and `.env`-style credentials inside submitted text, logs the scan result in Google Sheets, and sends a Telegram alert when exposed secrets are detected.

---

![n8n Workflow Overview](trueFlow.png)
![n8n Workflow Overview](falseFlow.png)
![n8n Workflow Overview](DB_google.png)

## Project Overview

The goal of this project is to create a lightweight secret exposure monitoring workflow.

The workflow receives text or logs through an n8n Webhook, scans the content using regex-based detection rules, identifies possible exposed secrets, calculates a risk score, saves the scan report in Google Sheets, sends a Telegram alert if secrets are found, and updates the scan status after notification.

This project is designed as a practical demo of **secret detection**, **security automation**, **data leakage prevention**, and **alerting**.

---

## Tech Stack

* **n8n Cloud** — workflow automation platform
* **Webhook Trigger** — receives text/log content for scanning
* **Code Node** — detects exposed secrets and calculates risk
* **Google Sheets** — stores scan reports
* **IF Node** — checks if secrets were found
* **Telegram Bot API** — sends secret exposure alerts

No paid external API is required for this project.

---

## Main Features

### Text and Log Intake

The workflow starts with a Webhook that receives content to scan.

Example payload:

```json
{
  "source": "GitHub Commit",
  "submittedBy": "developer@example.com",
  "environment": "production",
  "content": "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE\nSTRIPE_SECRET_KEY=sk_live_1234567890abcdef\nDATABASE_PASSWORD=mySuperSecretPassword"
}
```

---

### Secret Detection

The workflow uses a Code node to scan the submitted content for common secret patterns.

Detected secret types include:

```txt
AWS Access Key
Stripe Secret Key
GitHub Token
GitHub Fine-grained Token
Bearer Token
JWT Token
Private Key Block
Generic API Key
Generic Secret
Generic Token
Password Variable
```

The workflow masks detected secret values before saving or sending alerts.

Example masked output:

```txt
AWS Access Key: AKIAIO...AMPLE
Stripe Secret Key: sk_liv...cdef
Password Variable: DATABA...word
```

---

### Risk Scoring

The workflow calculates a risk score based on:

* number of detected secrets
* severity of detected secret types
* whether the content belongs to production or staging
* presence of `.env`-style content
* database-related indicators
* cloud/payment/developer platform indicators

The final risk score is capped at `100`.

---

### Risk Level Classification

The workflow classifies scan results into four levels:

```txt
0 - 34     Low
35 - 64    Medium
65 - 84    High
85 - 100   Critical
```

---

### Google Sheets Logging

Every scan is saved in Google Sheets.

Logged fields include:

```txt
Scan ID
Timestamp
Source
Submitted By
Environment
Secrets Found
Detected Types
Risk Score
Risk Level
Recommendation
Action Taken
Status
Notes
```

This creates a simple history of scanned content and findings.

---

### Telegram Alerting

If one or more secrets are detected, the workflow sends a Telegram alert.

Example alert:

```txt
🚨 Secret Exposure Alert

Potential exposed secrets were detected.

Risk Level: Critical
Risk Score: 100/100

Source: GitHub Commit
Submitted By: developer@example.com
Environment: production

Secrets Found: 3

Detected Types:
AWS Access Key, Stripe Secret Key, Password Variable

Masked Findings:
AWS Access Key: AKIAIO...AMPLE | Stripe Secret Key: sk_liv...cdef | Password Variable: DATABA...word

Recommendation:
Immediately rotate exposed credentials, remove them from the source, revoke active sessions if needed, and investigate possible unauthorized access.

Status: Open
```

---

### Status Update After Notification

After the Telegram alert is sent, the workflow updates the same Google Sheets row:

```txt
Action Taken = Telegram alert sent
Status = Notified
```

This confirms that the exposure was escalated.

---

## Workflow Structure

The workflow contains 6 nodes:

```txt
01 Webhook - Receive Text for Secret Scan
02 Code - Detect Exposed Secrets
03 Google Sheets - Save Scan Report
04 IF - Secrets Found?
05 Telegram - Send Secret Exposure Alert
06 Google Sheets - Update Action Taken
```

---

## Workflow Flow

```txt
01 Webhook - Receive Text for Secret Scan
↓
02 Code - Detect Exposed Secrets
↓
03 Google Sheets - Save Scan Report
↓
04 IF - Secrets Found?
   ├── true
   │   ↓
   │   05 Telegram - Send Secret Exposure Alert
   │   ↓
   │   06 Google Sheets - Update Action Taken
   └── false
       end
```

---

## Node Details

### 01 Webhook - Receive Text for Secret Scan

This node receives the content to scan.

Configuration:

```txt
HTTP Method: POST
Path: secret-scan
Authentication: None
Respond: Immediately
```

For production usage, the Webhook should be protected with authentication or a secret header.

---

### 02 Code - Detect Exposed Secrets

This node scans the submitted content and detects possible exposed secrets.

It produces:

```txt
scanId
timestamp
source
submittedBy
environment
secretsFound
detectedTypes
maskedSecrets
riskScore
riskLevel
recommendation
actionTaken
status
notes
```

Detection patterns include:

```txt
AWS Access Key
Stripe Secret Key
GitHub Token
GitHub Fine-grained Token
Bearer Token
JWT Token
Private Key Block
Generic API Key
Generic Secret
Generic Token
Password Variable
```

The node masks detected values to avoid exposing full secrets in logs or alerts.

---

### 03 Google Sheets - Save Scan Report

This node saves the scan report in Google Sheets.

Google Sheet name:

```txt
Secret Exposure Monitor
```

Tab name:

```txt
Scans
```

Columns:

```txt
Scan ID
Timestamp
Source
Submitted By
Environment
Secrets Found
Detected Types
Risk Score
Risk Level
Recommendation
Action Taken
Status
Notes
```

Example mappings:

```txt
Scan ID: {{ $json.scanId }}
Timestamp: {{ $json.timestamp }}
Source: {{ $json.source }}
Submitted By: {{ $json.submittedBy }}
Environment: {{ $json.environment }}
Secrets Found: {{ $json.secretsFound }}
Detected Types: {{ $json.detectedTypes }}
Risk Score: {{ $json.riskScore }}
Risk Level: {{ $json.riskLevel }}
Recommendation: {{ $json.recommendation }}
Action Taken: {{ $json.actionTaken }}
Status: {{ $json.status }}
Notes: {{ $json.notes }}
```

---

### 04 IF - Secrets Found?

This node checks if exposed secrets were detected.

Condition:

```txt
{{ $('02 Code - Detect Exposed Secrets').item.json.secretsFound }} > 0
```

If true, the workflow sends a Telegram alert.

If false, the workflow ends after saving the report.

---

### 05 Telegram - Send Secret Exposure Alert

This node sends a Telegram alert when secrets are detected.

Recommended message:

```txt
🚨 Secret Exposure Alert

Potential exposed secrets were detected.

Risk Level: {{ $('02 Code - Detect Exposed Secrets').item.json.riskLevel }}
Risk Score: {{ $('02 Code - Detect Exposed Secrets').item.json.riskScore }}/100

Source: {{ $('02 Code - Detect Exposed Secrets').item.json.source }}
Submitted By: {{ $('02 Code - Detect Exposed Secrets').item.json.submittedBy }}
Environment: {{ $('02 Code - Detect Exposed Secrets').item.json.environment }}

Secrets Found: {{ $('02 Code - Detect Exposed Secrets').item.json.secretsFound }}

Detected Types:
{{ $('02 Code - Detect Exposed Secrets').item.json.detectedTypes }}

Masked Findings:
{{ $('02 Code - Detect Exposed Secrets').item.json.maskedSecrets }}

Recommendation:
{{ $('02 Code - Detect Exposed Secrets').item.json.recommendation }}

Status: {{ $('02 Code - Detect Exposed Secrets').item.json.status }}
```

---

### 06 Google Sheets - Update Action Taken

This node updates the same scan row after the Telegram alert is sent.

Column to match:

```txt
Scan ID
```

Match value:

```txt
{{ $('02 Code - Detect Exposed Secrets').item.json.scanId }}
```

Updated fields:

```txt
Action Taken: Telegram alert sent
Status: Notified
Notes: {{ $('02 Code - Detect Exposed Secrets').item.json.notes }} | Telegram alert sent
```

---

## Google Sheets Structure

Google Sheet:

```txt
Secret Exposure Monitor
```

Tab:

```txt
Scans
```

Columns:

```txt
Scan ID
Timestamp
Source
Submitted By
Environment
Secrets Found
Detected Types
Risk Score
Risk Level
Recommendation
Action Taken
Status
Notes
```

Example row:

| Scan ID | Timestamp            | Source        | Submitted By                                          | Environment | Secrets Found | Detected Types                                       | Risk Score | Risk Level | Recommendation                          | Action Taken        | Status   | Notes                           |
| ------- | -------------------- | ------------- | ----------------------------------------------------- | ----------- | ------------: | ---------------------------------------------------- | ---------: | ---------- | --------------------------------------- | ------------------- | -------- | ------------------------------- |
| 74      | 29/05/2026, 10:30:00 | GitHub Commit | [developer@example.com](mailto:developer@example.com) | production  |             3 | AWS Access Key, Stripe Secret Key, Password Variable |        100 | Critical   | Immediately rotate exposed credentials. | Telegram alert sent | Notified | Production environment detected |

---

## Test Payloads

### Critical Secret Exposure Test

```json
{
  "source": "GitHub Commit",
  "submittedBy": "developer@example.com",
  "environment": "production",
  "content": "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE\nSTRIPE_SECRET_KEY=sk_live_1234567890abcdef\nDATABASE_PASSWORD=mySuperSecretPassword"
}
```

Expected result:

```txt
Secrets are detected
Risk Level is Critical
Report is saved in Google Sheets
Telegram alert is sent
Google Sheets row is updated to Notified
```

---

### Clean Content Test

```json
{
  "source": "Support Ticket",
  "submittedBy": "user@example.com",
  "environment": "staging",
  "content": "Hello, I need help configuring the dashboard. No credentials are included here."
}
```

Expected result:

```txt
Secrets Found = 0
Risk Level = Low
Report is saved in Google Sheets
No Telegram alert is sent
```

---

## cURL Test Examples

### Critical Test

Replace the URL with your own n8n test webhook URL.

```bash
curl -X POST "https://YOUR-N8N-DOMAIN.app.n8n.cloud/webhook-test/secret-scan" \
-H "Content-Type: application/json" \
-d '{
  "source": "GitHub Commit",
  "submittedBy": "developer@example.com",
  "environment": "production",
  "content": "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE\nSTRIPE_SECRET_KEY=sk_live_1234567890abcdef\nDATABASE_PASSWORD=mySuperSecretPassword"
}'
```

---

### Clean Test

Replace the URL with your own n8n test webhook URL.

```bash
curl -X POST "https://YOUR-N8N-DOMAIN.app.n8n.cloud/webhook-test/secret-scan" \
-H "Content-Type: application/json" \
-d '{
  "source": "Support Ticket",
  "submittedBy": "user@example.com",
  "environment": "staging",
  "content": "Hello, I need help configuring the dashboard. No credentials are included here."
}'
```

---

## Security and Business Value

This project demonstrates a basic secret detection and alerting workflow.

It helps identify potentially exposed credentials in:

* logs
* support tickets
* copied `.env` files
* GitHub commits
* internal messages
* configuration snippets
* developer notes
* deployment outputs

This type of workflow can support:

* security operations
* developer security awareness
* incident response
* data leakage prevention
* internal security reviews
* portfolio cybersecurity demos

---

## Possible Real-World Use Cases

This workflow can be adapted for:

* scanning support ticket content
* scanning webhook payloads from code review tools
* scanning GitHub commit messages or diffs
* monitoring internal security submissions
* checking pasted logs before escalation
* detecting accidental credential leakage
* alerting security teams about exposed secrets

---

## Future Improvements

Possible upgrades:

* Add GitHub API integration to scan commits or pull requests
* Add Slack or Microsoft Teams alerting
* Add Jira/Trello/Notion ticket creation
* Add automatic severity-based escalation
* Add allowlist for known false positives
* Add entropy-based secret detection
* Add file upload support
* Add scanning for full `.env` files
* Add automatic credential rotation workflows
* Add dashboard reporting
* Add daily or weekly exposure summary
* Add duplicate detection
* Add webhook authentication
* Add encryption or secure storage for reports
* Add audit trail for remediation status

---

## Security Notes

Before using this in production:

* Protect the Webhook with authentication or a secret token.
* Do not store full secret values in Google Sheets.
* Do not send full secret values through Telegram.
* Mask all detected secrets in alerts and logs.
* Restrict Google Sheets access.
* Protect Telegram bot credentials.
* Add false-positive handling.
* Add validation for incoming payloads.
* Review privacy and data retention requirements.
* Rotate any real credential detected by the workflow immediately.

---

## Project Status

Current version: working demo

Implemented:

* Webhook text/log intake
* Regex-based secret detection
* Secret masking
* Risk score calculation
* Risk level classification
* Google Sheets scan logging
* IF logic for detected secrets
* Telegram alerting
* Google Sheets status update after alert

---

## Author

Built by **Iosif Castrucci**

GitHub: `iosif-castrucci-hub`
Email: `contact.iosifcastrucci@gmail.com`

---

## Disclaimer

This project is a demo cybersecurity automation workflow created for portfolio and educational purposes.

It is not a complete enterprise-grade secret scanning platform and should not be used as the only method for detecting or managing exposed credentials in production environments.

For production use, add authentication, payload validation, false-positive handling, secure storage, audit logging, access control, alert deduplication, and formal incident response processes.

---

## License

This repository is intended for portfolio and educational purposes.
You may adapt the workflow structure for your own projects.
