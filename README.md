# Local AI Email Auto-Responder

A fully local, private AI email auto-responder using n8n and Ollama. No cloud APIs required. Every email sent to the bot account receives an automatic AI-generated reply powered by llama3:8b running locally on your machine.

---

## Architecture

```
Sender's Gmail
      │
      │  sends email
      ▼
┌─────────────────────────────────────────────┐
│           Docker Compose Environment         │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │           n8n Workflow               │   │
│  │                                      │   │
│  │  [IMAP Trigger] ──► [Loop Check IF]  │   │
│  │   polls every         │         │    │   │
│  │   minute           true       false  │   │
│  │                      │           │   │   │
│  │               [HTTP Request]  [Stop] │   │
│  │               POST to Ollama         │   │
│  │                      │              │   │
│  │               [Send Email]           │   │
│  │               SMTP reply             │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌─────────────────────┐                    │
│  │   Ollama Service    │                    │
│  │   llama3:8b (local) │◄── HTTP POST       │
│  └─────────────────────┘                    │
└─────────────────────────────────────────────┘
      │
      │  AI reply via SMTP
      ▼
Sender's inbox
```

### How it works

1. The **IMAP trigger** polls the bot Gmail inbox every minute for unread emails
2. The **IF node** filters out self-sent emails and auto-replies to prevent infinite loops
3. The **HTTP Request** node sends the email content as a prompt to the local Ollama API
4. **Ollama** generates a professional reply using llama3:8b (runs 100% locally — no cloud)
5. The **Send Email** node sends the AI reply back to the original sender via SMTP

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running
- A dedicated Gmail account for the bot (do not use your personal inbox)
- Gmail App Password (see setup below)

---

## Setup

### 1. Clone and configure environment

```bash
cp .env.example .env
```

Edit `.env` with your bot Gmail credentials:

```env
IMAP_HOST=imap.gmail.com
IMAP_PORT=993
IMAP_USER=yourbotaccount@gmail.com
IMAP_PASSWORD=your-16-char-app-password

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=yourbotaccount@gmail.com
SMTP_PASSWORD=your-16-char-app-password
```

### 2. Get a Gmail App Password

1. Enable 2-Step Verification: [myaccount.google.com/security](https://myaccount.google.com/security)
2. Generate App Password: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Name it `n8n` → copy the 16-character password (no spaces)
4. Enable IMAP in Gmail: **Settings → See all settings → Forwarding and POP/IMAP → Enable IMAP**

### 3. Start services

```bash
docker-compose up -d --build
```

Verify both containers are healthy (wait ~90 seconds):

```bash
docker-compose ps
```

Both `n8n_responder` and `ollama_responder` should show status `healthy`.

### 4. Pull the LLM model (one-time, ~5 GB download)

```bash
docker exec -it ollama_responder ollama pull llama3:8b
```

Verify the model downloaded:

```bash
docker exec -it ollama_responder ollama list
```

### 5. Import the workflow into n8n

1. Open [http://localhost:5678](http://localhost:5678) in your browser
2. Go to **Overview** → **Create workflow** dropdown → **Import from file**
3. Select `workflow.json` from this repository
4. Click the **Email Read (IMAP)** node → add your IMAP credential
5. Click the **Send Email** node → add your SMTP credential
6. Toggle the workflow to **Active** (top right)

---

## Verification

```bash
docker-compose ps
```

Expected output:
```
NAME               STATUS
n8n_responder      Up (healthy)
ollama_responder   Up (healthy)
```

Test the Ollama API directly:

```bash
curl http://localhost:11434/api/tags
```

Should return JSON containing `llama3:8b`.

---

## Testing

1. From your **personal Gmail**, send an email to the bot account
2. Wait **1–2 minutes** (workflow polls every minute)
3. Check your personal Gmail inbox for an AI-generated reply

The reply will be addressed using the sender's actual name and signed:
```
Warm regards,
Lahari
```

---

## Project structure

```
.
├── docker-compose.yml   # Defines n8n and Ollama services with healthchecks
├── workflow.json        # Complete n8n workflow export
├── .env                 # Your actual credentials (never commit this)
├── .env.example         # Credential template for reference
├── submission.json      # Email credentials for automated evaluation
└── README.md
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Containers not healthy | Run `docker logs n8n_responder` or `docker logs ollama_responder` |
| IMAP auth failure | Use App Password (not Gmail password), ensure IMAP is enabled |
| No reply received | Check **Executions** tab in n8n for error details |
| Ollama model missing | Re-run `docker exec -it ollama_responder ollama pull llama3:8b` |
| Infinite reply loop | Confirm Loop Prevention IF node is correctly configured |
| n8n can't reach Ollama | Ensure both services share `ai_responder_net` in docker-compose |