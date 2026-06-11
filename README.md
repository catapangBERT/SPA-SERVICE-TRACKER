# Spa Service Tracker Bot

An n8n automation that lets spa staff update client session statuses by simply sending a Telegram message. The bot validates the input, cross-checks against a schedule, prevents duplicate updates, and writes everything to Google Sheets automatically.

---

## The Problem

Spa staff had to manually open a Google Sheet after every session and update each client's status one by one, every single day. It was repetitive, time-consuming, and easy to forget.

## The Solution

Staff now just send a message on Telegram in a simple format and the bot handles everything automatically.

---

## How It Works

**Message format:**
```
Client Name, Service, Status, Time
```

**Example:**
```
Jennifer, Swedish Massage, Done, 7:30
```

**Valid statuses:** `Done`, `Cancelled`, `Rescheduled`

### Flow

```
Telegram Message Received
        ↓
Parse and validate message format
        ↓
Has error? → Send error reply and stop
        ↓ (no error)
Check SCHEDULES sheet — does this client + service exist?
        ↓
Not found? → Reply "No schedule found" and stop
        ↓ (found)
Check TRACKER sheet — already updated with same status?
        ↓
Already updated? → Reply "Already updated" and stop
        ↓ (not yet updated)
Existing row in TRACKER? → Update row
New client in TRACKER? → Append new row
        ↓
Send success reply via Telegram
```

---

## Google Sheets Setup

You need **one Google Spreadsheet with two tabs:**

### Tab 1: `SCHEDULES`
Contains all booked appointments. Filled in manually by staff or admin.

| Client Name | Service | Time | Date | Location |
|---|---|---|---|---|
| Jennifer | Swedish Massage | 7:30 | 2026-06-11 | Branch A |

### Tab 2: `TRACKER`
Filled automatically by n8n. Starts empty.

| Client Name | Service | Status | Time | Last Updated |
|---|---|---|---|---|
| *(n8n writes here)* | | | | |

> Column names must match exactly including capitalization and spacing.

---

## Tech Stack

- [n8n](https://n8n.io/) — self-hosted via Docker
- Google Sheets API
- Telegram Bot API

---

## Setup Instructions

### 1. Prerequisites
- n8n running locally via Docker
- A Telegram Bot (create one via [@BotFather](https://t.me/BotFather))
- A Google account with Sheets API access
- ngrok or Cloudflare Tunnel (to expose your local n8n webhook to Telegram)

### 2. Google Sheets
- Create a new Google Spreadsheet
- Add two tabs named exactly `SCHEDULES` and `TRACKER`
- Add the column headers as shown above
- Copy your Spreadsheet ID from the URL:
  `https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit`

### 3. Import the Workflow
- Open your n8n instance
- Go to **Workflows → Import from file**
- Select `SPA_SERVICE_TRACKER_github.json`

### 4. Replace Placeholders
After importing, update the following in your n8n nodes:

| Placeholder | What to replace with |
|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Your actual Google Spreadsheet ID |
| `YOUR_TELEGRAM_CREDENTIAL_ID` | Your n8n Telegram credential ID |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Your n8n Google Sheets credential ID |
| `YOUR_INSTANCE_ID` | Your n8n instance ID |

Or simply re-link all credentials manually inside n8n after importing.

### 5. Activate the Workflow
- Open n8n via your **ngrok URL** (not localhost)
- Activate the workflow using the toggle
- Register your webhook with Telegram:
```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=https://YOUR_NGROK_URL/webhook/<WEBHOOK_ID>
```

---

## Nodes Overview

| Node | Type | Purpose |
|---|---|---|
| Telegram Trigger | Trigger | Receives incoming Telegram messages |
| Parse Message | Code | Splits and validates the message format |
| Has Error? | IF | Routes to error reply if format is invalid |
| Send Error Reply | Telegram | Sends format error message back to staff |
| Read SCHEDULES Sheet | Google Sheets | Fetches all rows from SCHEDULES tab |
| Check In SCHEDULES | Code | Checks if client + service exists in schedule |
| In SCHEDULES? | IF | Routes based on schedule match result |
| Reply Not In Schedules | Telegram | Notifies staff if no matching schedule found |
| Read TRACKER Sheet | Google Sheets | Fetches all rows from TRACKER tab |
| Check In TRACKER | Code | Checks if already updated with same status |
| Already Updated? | IF | Routes based on duplicate update check |
| Reply Already Updated | Telegram | Notifies staff if status already set |
| Find Client Row | Code | Finds row index in TRACKER for update |
| New or Existing? | IF | Routes to append or update |
| Update Status | Google Sheets | Updates existing row in TRACKER |
| Add New Row | Google Sheets | Appends new row to TRACKER |
| Send Success Reply | Telegram | Confirms successful update to staff |

---

## Notes

- Matching is done on **both Client Name and Service** together, so the same client with different services won't conflict.
- The workflow is case-insensitive for name and service matching.
- `Last Updated` is automatically set to Philippine time (Asia/Manila).
