# 🎙️ voice-to-email-n8n

> **Speak. Send. Done.**
> A browser-based voice recorder paired with an n8n automation workflow that transcribes your voice using Google Gemini, auto-generates a professionally formatted HTML email, fetches mentioned attachments from Google Drive, and sends everything via Gmail — hands-free.

---

## 📌 What It Does

1. **Record** a voice message in the browser (`recorder.html`)
2. **Send** the raw audio binary to an n8n webhook with one click
3. n8n **transcribes** the audio using **Gemini 2.0 Flash**
4. Gemini **extracts** the recipient email, file name, and subject from the transcript
5. Gemini **generates** a professional email body matching the intent of the voice message
6. A second Gemini pass **reformats** the body into clean, styled HTML
7. The workflow **searches Google Drive** for any file mentioned in the recording
8. The file is **downloaded** and attached to the email
9. Gmail **sends** the final email with attachment — fully automated

---

## 🗂️ Repository Structure

```
voice-to-email-n8n/
├── recorder.html                                    # Browser-based mic recorder UI
├── Email_voice_workflow_Drive_Attachments.json      # n8n workflow export
└── README.md
```

---

## 🧩 Components

### `recorder.html` — Browser Mic Recorder

A standalone single-file HTML app with no external dependencies.

| Feature | Detail |
|---|---|
| Audio capture | `MediaRecorder` API (`audio/webm;codecs=opus` or `audio/ogg`) |
| Controls | Record, Pause/Resume, Stop, Send |
| Live meter | Real-time amplitude visualizer via `AnalyserNode` |
| Timer | Elapsed recording time display |
| Delivery | POSTs raw binary audio to a configurable n8n webhook URL |
| Compatibility | Works on Chrome, Edge, Firefox (HTTPS or localhost) |

> **Configure:** Edit the `value` attribute of the `#webhook` input field in `recorder.html` to point to your n8n webhook URL.

---

### `Email_voice_workflow_Drive_Attachments.json` — n8n Workflow

#### Node Pipeline

```
[Webhook]
    └─▶ [Transcribe a recording]  ← Gemini 2.0 Flash (audio → text)
            ├─▶ [Code4] → [AI Agent1 / Gemini] ─────────────────┐
            │      (extract: recipient email, fileName)          │
            └─▶ [Merge2] ◀────────────────────────────────────── ┘
                    └─▶ [Code]  (combine email + fileName + transcript)
                            ├─▶ [Code1] → [AI Agent / Gemini]
                            │      (generate email body from voice intent)
                            │        └─▶ [Code3]  (detect subject by keyword)
                            │              └─▶ [Code2]  (build HTML body)
                            │                    └─▶ [Code5] → [AI Agent2 / Gemini]
                            │                           (reformat HTML professionally)
                            │                             └─▶ [Merge3] → [Code6]
                            │                                    └─▶ [Merge] ─────┐
                            │                                                      ▼
                            └─▶ [Search files and folders]  ← Google Drive      [Send a message] → Gmail
                                    └─▶ [Download file]
                                            └─▶ [Merge] ──────────────────────────┘
```

#### Node Descriptions

| Node | Type | Purpose |
|---|---|---|
| **Webhook** | Webhook (POST) | Receives raw audio binary from recorder.html |
| **Transcribe a recording** | Google Gemini | Transcribes audio to text using `models/gemini-2.0-flash` |
| **Code4** | Code | Builds extraction prompt (email, fileName, documentName) |
| **AI Agent1** | LangChain Agent + Gemini | Extracts structured JSON from transcript |
| **Merge2** | Merge | Combines AI entity data + raw transcript |
| **Code** | Code | Merges recipient, fileName, and transcript text into unified item |
| **Code1** | Code | Builds Gemini prompt to generate email body from intent |
| **AI Agent** | LangChain Agent + Gemini | Generates contextual email message text |
| **Code3** | Code | Keyword-based subject detection (cover letter, report, reminder, etc.) |
| **Code2** | Code | Wraps message in professional HTML email template |
| **Code5** | Code | Builds prompt to professionally reformat the HTML body |
| **AI Agent2** | LangChain Agent + Gemini | Rewrites and polishes the HTML email body |
| **Merge3 + Code6** | Merge + Code | Assembles final email object (email, subject, body) |
| **Search files and folders** | Google Drive | Searches Drive for PDF/image by filename extracted from voice |
| **Download file** | Google Drive | Downloads the matched file as binary |
| **Merge** | Merge (by position) | Joins formatted email data with downloaded file binary |
| **Send a message** | Gmail | Sends the email with the attachment |

---

## ⚙️ Setup & Configuration

### Prerequisites

- **n8n** instance (self-hosted or cloud), v1.x+
- **Google Gemini API** key (PaLM API credential in n8n)
- **Gmail OAuth2** credential configured in n8n
- **Google Drive OAuth2** credential configured in n8n
- A browser that supports `MediaRecorder` (Chrome/Edge recommended)

### Step 1 — Import the Workflow

1. Open your n8n instance
2. Go to **Workflows → Import**
3. Upload `Email_voice_workflow_Drive_Attachments.json`
4. Activate the workflow

### Step 2 — Configure Credentials

After importing, update the following nodes with your credentials:

| Node | Credential Needed |
|---|---|
| Transcribe a recording | Google Gemini (PaLM) API |
| Google Gemini Chat Model | Google Gemini (PaLM) API |
| Google Gemini Chat Model1 | Google Gemini (PaLM) API |
| Google Gemini Chat Model2 | Google Gemini (PaLM) API |
| AI Agent / AI Agent1 / AI Agent2 | (uses Gemini model above, no separate cred) |
| Search files and folders | Google Drive OAuth2 |
| Download file | Google Drive OAuth2 |
| Send a message | Gmail OAuth2 |

### Step 3 — Get the Webhook URL

1. Open the **Webhook** node in the workflow
2. Copy the **Production** webhook URL  
   (format: `https://your-n8n-instance/webhook/f2265fc3-ebf0-4d02-810a-baf84ee96af4`)

### Step 4 — Configure the Recorder

Open `recorder.html` in a text editor and update the webhook input's `value`:

```html
<input id="webhook" ...
  value="https://your-n8n-instance/webhook/f2265fc3-ebf0-4d02-810a-baf84ee96af4" />
```

Or simply paste the URL directly into the input field in the browser after opening the file.

### Step 5 — Use It

1. Open `recorder.html` in Chrome or Edge (HTTPS or localhost)
2. Click **● Record** — allow microphone access
3. Say something like:

   > *"Send an email to john.doe@example.com with the sales_report.pdf attached. Let him know the Q3 numbers are ready for review."*

4. Click **■ Stop**, then **⬆ Send**
5. Check the Gmail outbox — the email will arrive formatted with the attachment

---

## 🎤 Example Voice Commands

| What you say | What happens |
|---|---|
| *"Send a greeting to alice@company.com"* | Sends a friendly hello email |
| *"Email bob@work.com the report.pdf"* | Finds `report.pdf` in Drive, attaches it |
| *"Send a reminder to team@org.com about the meeting"* | Subject: "Reminder", body generated by Gemini |
| *"Cover letter to hr@company.com with cv.docx"* | Subject: "Cover Letter", attaches `cv.docx` from Drive |

---

## 🔒 Notes & Limitations

- **HTTPS required** for microphone access in production (localhost exempted)
- **Google Drive search** is based on filename keywords — ensure files in Drive are named clearly
- **Gmail attachment** works for PDFs and images; other MIME types may need Drive node adjustment
- **Gemini extraction accuracy** depends on clear speech and a valid email address being spoken
- The webhook URL contains a UUID path — keep it private or add n8n authentication headers

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend recorder | Vanilla HTML/CSS/JS, MediaRecorder API, Web Audio API |
| Automation engine | [n8n](https://n8n.io) |
| AI / Transcription | Google Gemini 2.0 Flash (via n8n LangChain nodes) |
| Email | Gmail via OAuth2 |
| File storage | Google Drive via OAuth2 |

---

## 📄 License

MIT — free to use and modify.
