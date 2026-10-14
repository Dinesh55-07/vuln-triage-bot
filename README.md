# vuln-triage-bot

An **n8n** workflow that takes a vulnerability scan result, removes duplicates, uses an **AI agent** to rank the top 5 most exploitable findings (with a one-line fix each), and sends them to **Telegram** for human approval before the final report is released.

**Platform:** n8n Cloud | **Domain:** Security Automation

---

## Features

- Accepts a scan result as JSON through an HTTP POST (webhook)
- Removes duplicate findings (same host + port + type + CVE/title)
- AI agent ranks the top 5 findings by exploitability with a one-line fix each
- Low-severity findings are summarised in a single line
- Human approval on Telegram (Approve / Reject) before the final report is sent
- Access control: the bot only talks to the owner's chat ID; any other sender is ignored and logged
- Every message carries a status line:
  `🟣 exec #<execution id> | <IST time> | <verdict / severity> | <one-line summary>`

## Nodes

### Flow 1: triage and approval

| Node | Type | Purpose |
|---|---|---|
| Receive Scan (POST) | Webhook | Entry point; receives the scan JSON |
| Parse & Dedupe | Code | Removes duplicates, separates low/info findings into a one-line summary |
| AI Agent - Rank Top 5 | AI Agent | Ranks the top 5 findings by exploitability |
| Gemini Chat Model | Chat model | Language model used by the agent |
| Structured Output Parser | Output parser | Forces the AI answer into a fixed JSON shape |
| Format Message | Code | Builds the Telegram message and the status line |
| Telegram - Send and Wait (Approval) | Telegram | Sends the ranked list with Approve / Reject buttons and pauses the workflow |
| Approved? | IF | Routes on the button pressed |
| Telegram - Final Report | Telegram | Sends the final report after approval |
| Telegram - Rejected Notice | Telegram | Sends a rejection notice |

### Flow 2: access control

| Node | Type | Purpose |
|---|---|---|
| Telegram Trigger (inbound) | Telegram Trigger | Fires on every message sent to the bot |
| Allowlist: is owner? | IF | Compares the sender's chat ID with the owner's |
| Owner message (no action) | No Operation | Owner messages pass with no action |
| Log & Ignore Intruder | Code | Logs time, chat ID, username and text of other senders; sends no reply |

## Repository contents

```
vuln-triage-bot/
├── README.md
├── Vulnerability Triage.json    # exported n8n workflow
├── sample_scan_NGP-055.json     # sample scan (14 raw findings, 3 duplicates)
└── screenshots/                 # canvas, Telegram messages, execution logs
```

## Sample data

`sample_scan_NGP-055.json` contains 14 raw findings: open ports, outdated versions and weak ciphers.

| Stage | Count |
|---|---|
| Raw findings | 14 |
| Duplicates removed | 3 |
| Unique findings | 11 |
| Low / info (summarised in one line) | 3 |
| Candidates ranked by the AI | 8 |
| Shown as top ranked | 5 |

## Setup

1. **Create a Telegram bot.** Message `@BotFather`, send `/newbot`, and copy the token.
2. **Get your chat ID.** Open `@userinfobot` and press Start.
3. **Start your bot.** Open your bot and press Start so it can message you.
4. **Import the workflow.** In n8n, use the menu (⋯) → *Import from file* and choose `NGP-055_workflow.json`.
5. **Add credentials.**
   - Telegram API credential with your bot token
   - Google Gemini API credential (key from Google AI Studio). Pick any currently available Flash model in the *Gemini Chat Model* node; model names change over time.
6. **Replace the placeholder.** Search for `YOUR_CHAT_ID` and replace it with your chat ID in these nodes:
   - Telegram - Send and Wait (Approval)
   - Telegram - Final Report
   - Telegram - Rejected Notice
   - Allowlist: is owner?
7. **Publish** the workflow so the Telegram Trigger and webhook run continuously.

## Usage

Send the sample scan to the webhook:

```bash
curl -X POST "https://<your-n8n-host>/webhook/ngp055-scan" \
     -H "Content-Type: application/json" \
     -d @sample_scan_NGP-055.json
```

Use the **Test URL** (`/webhook-test/ngp055-scan`) while the editor is listening, and the **Production URL** (`/webhook/ngp055-scan`) once the workflow is published.

Then:

1. The ranked list arrives on Telegram with Approve / Reject buttons.
2. Tap **Approve**: the final report is sent (verdict shows `APPROVED`).
3. Tap **Reject**: a rejection notice is sent (verdict shows `REJECTED`).

### Example output

```
🟣 NGP-055 | exec #6 | 14 Oct 2026 08:51 IST | Critical / AWAITING APPROVAL | Critical remote code execution and memory disclosure vulnerabilities present high exploitability risk.

Top 5 by exploitability (11 unique findings from 14 raw, 3 duplicates removed):

1. Apache httpd 2.4.49
   Why: Critical path traversal and RCE vulnerability with public exploits.
   Fix: Upgrade Apache httpd to 2.4.50 or later.

2. OpenSSL 1.0.1f (Heartbleed)
   Why: Critical Heartbleed memory disclosure flaw with trivial exploitation.
   Fix: Upgrade OpenSSL to a non-vulnerable version.

3. Telnet service exposed
   Why: Exposed cleartext remote login service allowing credential interception.
   Fix: Disable Telnet and enforce SSH.

4. SMB exposed to LAN
   Why: Outdated SMBv1 protocol enabled on network-exposed port 445.
   Fix: Disable SMBv1 across the network.

5. TLS RC4 cipher suite enabled
   Why: Weak TLS RC4 cipher suite enabled on HTTPS service.
   Fix: Disable RC4 cipher suites in TLS configuration.

Low severity (3): Self-signed certificate; Server banner discloses version; ICMP timestamp response
```

The AI ranking can vary slightly between runs.

## Security

- **Allowlist (access control).** The workflow only talks to the owner's chat ID. Messages from any other Telegram account are ignored with no reply and logged (time, chat ID, username, text). Logs can be viewed in the n8n *Executions* tab.
- **Human approval.** The final report is only released after the owner presses Approve.
- **Secrets.** The bot token and API keys are stored as n8n credentials and are not included in the exported workflow. Do not commit real tokens or keys.

## Tech stack

- n8n (Cloud)
- Webhook and Code (JavaScript) nodes
- AI Agent with Google Gemini and a Structured Output Parser
- Telegram Bot API

## Future work

- Create tickets (Jira, GitHub Issues) after approval
- Email or save the final report (Gmail, Google Sheets)
- Store intruder logs in a persistent table
- Connect to a live scanner instead of a sample file


