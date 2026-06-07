# MedScribe — AI-Powered Voice Medical Scribe

> **Voice-first, AI-driven clinical documentation that learns each doctor's style.**
> Speak naturally during a consultation and MedScribe transcribes it in real-time, structures it into a SOAP note, extracts a prescription, and lets patients follow along on their own device — all without touching a keyboard. Over time, [Hindsight agent memory](https://github.com/vectorize-io/hindsight) makes every note sound more like the doctor who wrote it.

---

## Table of Contents

1. [Overview](#overview)
2. [Key Features](#key-features)
3. [Tech Stack](#tech-stack)
4. [Architecture](#architecture)
5. [User Flows](#user-flows)
6. [Hindsight AI — Agent Memory](#hindsight-ai--agent-memory)
7. [Project Structure](#project-structure)
8. [API Reference](#api-reference)
9. [Database Schema](#database-schema)
10. [Environment Variables](#environment-variables)
11. [Getting Started](#getting-started)
12. [Scripts](#scripts)
13. [Notable Implementation Details](#notable-implementation-details)

---

## Overview

MedScribe solves one of the biggest pain points for doctors in India: **clinical documentation overhead**. The average GP spends 30–40% of their time writing notes instead of focusing on the patient. MedScribe eliminates that friction.

A doctor opens a session on their phone or laptop, presses the mic, and talks naturally with the patient. MedScribe:

- **Transcribes** the conversation every ~4 seconds using Groq Whisper
- **Structures** the full transcript into a SOAP note + prescription list using LLaMA 3.3 70B
- **Personalizes** the output over time using [Hindsight agent memory](https://hindsight.vectorize.io/) — learning each doctor's drug notation, plan verbosity, follow-up cadence, and language mix
- **Shares** a live session link with the patient so they can follow the transcript and read their own summary

---

## Key Features

### For Doctors

| Feature | Description |
|---------|-------------|
| **Live Session Recording** | One-tap mic starts a live consultation. The full accumulated audio blob is re-sent each ~4s interval for complete transcript accuracy (see [WebM Chunking Fix](#webm-chunking-fix)). |
| **Real-time Transcription** | Live transcript panel updates as speech is recognized; auto-follows new lines. |
| **AI SOAP Generation** | On stop, the full transcript goes to LLaMA 3.3 70B → structured SOAP JSON (Subjective → Objective → Assessment → Plan). |
| **Inline SOAP Editing** | Every field is an editable textarea. An `Auto-generated` badge shows for 3s, then becomes `Edited`. |
| **Prescription Extraction** | The LLM extracts a prescription list: drug, dose, frequency, duration. |
| **Printable Prescription Card** | Stylized, printable card with doctor name, date, and medicine table (via `react-to-print`). |
| **Share with Patient** | Modal shows the full session URL and formatted session ID; patient follows along live. |
| **Personalized Notes (Hindsight)** | Notes adapt to the doctor's style across sessions. A Brain badge shows `Memory learning` / `Memory saved`. |
| **Doctor Dashboard** | Stats (unique patients, sessions today, total sessions) + full session history with expandable SOAP summaries. |
| **Patient Autocomplete** | Debounced (250ms) typeahead in the New Session modal; links sessions to existing patient records by email or selection. Walk-ins fully supported. |
| **Patient Profiles** | `/patients` and `/patients/[id]` show full patient histories. |

### For Patients

| Feature | Description |
|---------|-------------|
| **Live Session View** | Open the session URL on any device; transcript polls every second. |
| **Visit Summary** | After the doctor saves, see a structured SOAP + prescription summary. |
| **Visit Timeline & Stats** | `/home` shows past visits, total medicines, and distinct doctors seen. |
| **Prescription Photo Upload** | Drag-drop, file picker, or phone camera. PNG/JPG/WEBP up to 10MB. |
| **OCR Medicine Extraction** | Uploaded images go to Groq vision (LLaMA 4 Scout) to extract doctor name + medicines. |
| **Cloudinary Storage** | Images stored on Cloudinary, with base64 fallback. |
| **Smart Session Join** | Accepts a full session URL or a bare UUID (regex-extracted). |

### Auth & Onboarding

| Feature | Description |
|---------|-------------|
| **Clerk Auth** | Email + password (and social if configured). |
| **Role Selection** | `/choose-role` → Doctor or Patient. |
| **Role Propagation** | `POST /api/auth/setup` writes role to Clerk `publicMetadata` + creates a MongoDB `User`. |
| **Role-based Redirect** | `/auth/ready` routes doctors → `/dashboard`, patients → `/home`. |
| **Auth Guards** | Server-side via `proxy.js` (Next.js 16's Clerk file) + client-side `useUser()` checks. |

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js (App Router) | 16.2.7 |
| UI Library | React | 19.2.4 |
| Styling | Tailwind CSS v4 | ^4.0 |
| Auth | Clerk | ^7.4.3 |
| Database | MongoDB Atlas + Mongoose | ^9.6.3 |
| STT | Groq Whisper (`whisper-large-v3-turbo`) | groq-sdk ^1.2.1 |
| LLM (SOAP) | Groq LLaMA 3.3 70B Versatile | groq-sdk ^1.2.1 |
| LLM (OCR) | Groq LLaMA 4 Scout 17B | groq-sdk ^1.2.1 |
| Agent Memory | [Hindsight by Vectorize](https://github.com/vectorize-io/hindsight) | hindsight-client ^0.7.2 |
| Image Storage | Cloudinary | ^2.10.0 |
| Fonts | Barlow Semi Condensed + Instrument Serif | next/font/google |
| Animations | Framer Motion + GSAP | ^12 / ^3.15 |
| Icons | Lucide React | ^1.17.0 |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT (Browser)                         │
│   Doctor Side                        Patient Side               │
│   ┌──────────────────────┐          ┌─────────────────────┐     │
│   │  DoctorSessionView   │          │  PatientSessionView  │     │
│   │   RecordButton       │          │  Live transcript     │     │
│   │   LiveTranscript     │          │  Visit summary       │     │
│   │   SOAPNote           │          │  Prescription Rx     │     │
│   │   PrescriptionCard   │          └─────────────────────┘     │
│   └──────────┬───────────┘                                      │
│              │ MediaRecorder API (WebM / OGG / MP4)              │
└──────────────┼───────────────────────────────────────────────────┘
               ▼
┌──────────────────────────────────────────────────────────────────┐
│                      NEXT.JS 16 APP ROUTER                        │
│  POST /api/transcribe  →  Groq Whisper STT                        │
│  POST /api/structure   →  Hindsight recall                        │
│                           → LLaMA 3.3 70B (SOAP JSON)            │
│                           → Hindsight retain (fire-and-forget)   │
│  POST /api/ocr         →  Groq vision (LLaMA 4 Scout)            │
│  GET/POST /api/visits  →  MongoDB CRUD                           │
│  GET /api/patients/match → Patient autocomplete                  │
│  GET /api/session/:id  →  In-memory session store (transcript)   │
│  POST /api/auth/setup  →  Clerk metadata + MongoDB user create   │
└──────────────────────────┬───────────────────────────────────────┘
         ┌─────────────────┼──────────────────────┐
         ▼                 ▼                      ▼
  MongoDB Atlas       Groq Cloud           Hindsight Cloud
 (Users, Visits,    (Whisper STT,         (Per-doctor
  Prescriptions)    LLaMA 3.3 / 4)         memory banks)
```

### In-Memory Session Store

Live transcripts are stored in a Node.js in-process `Map` (`src/lib/sessionStore.js`) keyed by `sessionId` — no Redis dependency, zero extra latency. The patient's live view polls `GET /api/session/:sessionId` every second.

> **Production note:** For multi-instance deployments, replace `sessionStore.js` with Redis or a DB-backed store.

---

## User Flows

### Doctor Flow

```
Sign Up → /choose-role (Doctor) → POST /api/auth/setup → /auth/ready → /dashboard
/dashboard → "New Session" (typeahead patient search) → POST /api/visits → /session/:id
/session/:id (DoctorSessionView)
  → mic → every ~4s: POST /api/transcribe → Groq Whisper → setTranscript (replace)
  → stop → POST /api/structure
      → Hindsight recall (≤6 doctor-style memories)
      → buildPersonalizedPrompt → Groq LLaMA 3.3 70B → SOAP JSON
      → Hindsight retain (fire-and-forget)
  → edit SOAP / prescription → "End & Save" → POST /api/session/:id/save → /dashboard
```

### Patient Flow

```
Sign Up → /choose-role (Patient) → /auth/ready → /home
/home → timeline, stats, join session (URL or UUID) → /session/:id
/session/:id (PatientSessionView) → polls GET /api/session/:id every 1s → summary after save
/prescriptions/upload → Cloudinary → POST /api/ocr → LLaMA 4 Scout → review → POST /api/prescriptions
```

---

## Hindsight AI — Agent Memory

MedScribe uses **[Hindsight by Vectorize](https://hindsight.vectorize.io/)** — open-source [agent memory](https://vectorize.io/what-is-agent-memory) — to give each doctor a persistent memory bank that improves SOAP quality over time, **without per-user fine-tuning**.

### How it works

**After each session** (`retainSessionPatterns`), MedScribe stores a natural-language summary of the visit. Hindsight's own extraction layer decides what's worth remembering — drug notation (generic vs. brand), plan verbosity, follow-up intervals, and language mix (English / Hinglish / Hindi).

**Before each generation** (`recallDoctorMemory`), MedScribe recalls the most relevant memories (`budget: "mid"`, `maxTokens: 800`) and injects up to 6 into the LLM system prompt as a `DOCTOR PREFERENCES` block. The LLM applies them subtly — and always defers to the current transcript when it contradicts a stored preference.

### Memory bank scoping

Each doctor gets a dedicated bank named `medscribe-doctor-{clerkUserId}`. Patients never get memory banks — the `isDoctor` role check (from `sessionClaims?.metadata?.role`) gates all Hindsight calls. One doctor, one bank, no cross-contamination.

### Graceful degradation

Every recall/retain is wrapped so a memory outage **never blocks a clinical note**. If Hindsight is unavailable, `recallDoctorMemory` returns `[]` and the scribe falls back to its generic-but-correct behavior. Memory is an enhancement, never a critical-path dependency.

### Configuration

```env
HINDSIGHT_BASE_URL=https://api.hindsight.vectorize.io
HINDSIGHT_API_KEY=hsk_...
```

Get your API key at [ui.hindsight.vectorize.io → Settings → API Keys](https://ui.hindsight.vectorize.io).

> **Important:** The API endpoint is `https://api.hindsight.vectorize.io`. Do **not** use `https://hindsight.vectorize.io` (the docs site) — that returns 405 errors for API calls.

---

## Project Structure

```
soapmaker/
├── src/
│   ├── app/
│   │   ├── layout.js                      # Root layout: fonts, Clerk provider, dark mode
│   │   ├── globals.css                    # Tailwind v4 base + CSS variables
│   │   ├── page.js                        # Marketing landing page
│   │   ├── choose-role/page.js            # Role picker: Doctor / Patient
│   │   ├── auth/ready/page.js             # Post-auth role-based redirect
│   │   ├── (auth)/sign-in | sign-up/      # Clerk components
│   │   ├── (doctor)/
│   │   │   ├── dashboard/page.js          # Dashboard + NewSessionModal
│   │   │   └── patients/[patientId]/      # Patient profiles
│   │   ├── (patient)/
│   │   │   ├── home/page.js               # Stats, join session, timeline
│   │   │   └── prescriptions/upload/      # Photo upload + OCR
│   │   ├── session/[sessionId]/
│   │   │   ├── page.js                    # Role splitter
│   │   │   ├── DoctorSessionView.jsx      # Record, SOAP, share, save
│   │   │   └── PatientSessionView.jsx     # Live view + summary
│   │   └── api/
│   │       ├── transcribe/route.js        # audio → Groq Whisper
│   │       ├── structure/route.js         # transcript → Hindsight → LLaMA → SOAP
│   │       ├── ocr/route.js               # image → Groq vision
│   │       ├── visits/route.js            # session CRUD
│   │       ├── session/[sessionId]/       # live transcript + save
│   │       ├── patients/                  # CRUD + match (autocomplete)
│   │       ├── prescriptions/route.js     # prescription CRUD
│   │       └── auth/setup/route.js        # role + MongoDB user
│   ├── components/
│   │   ├── header.jsx                     # Fixed glass-pill navbar
│   │   ├── scribe/                        # RecordButton, LiveTranscript, SOAPNote, PrescriptionCard
│   │   └── patient/                       # VisitTimeline, PrescriptionUpload, ExtractedMeds
│   └── lib/
│       ├── db/                            # connect.js + models (User, Visit, Prescription)
│       ├── groq.js                        # Groq client singleton
│       ├── hindsight.js                   # recallDoctorMemory, retainSessionPatterns, doctorBank
│       ├── prompts.js                     # SOAP_SYSTEM_PROMPT, buildPersonalizedPrompt, OCR_PROMPT
│       ├── sessionStore.js                # In-memory transcript store
│       └── cloudinary.js                  # Cloudinary upload helper
├── proxy.js                               # Clerk auth proxy (Next.js 16 — NOT middleware.js)
├── scripts/seed.mjs                       # DB seed
└── package.json
```

---

## API Reference

### `POST /api/transcribe`
Transcribes audio via Groq Whisper. **Request:** `multipart/form-data` — `audio` (File, ≥1KB), `sessionId`, `mimeType`. **Response:** `{ text: string }`

### `POST /api/structure`
Generates a SOAP note. Doctors get Hindsight personalization. **Request:** `{ transcript, sessionId }`. **Response:**
```json
{
  "soap": {
    "subjective": { "chiefComplaint": "...", "history": "..." },
    "objective":  { "vitals": "...", "examination": "..." },
    "assessment": "...",
    "plan": "..."
  },
  "prescriptions": [
    { "drug": "Amoxicillin", "dose": "500 mg", "frequency": "TDS", "duration": "5 days" }
  ]
}
```

### `POST /api/ocr`
Extracts medicines from a prescription image via Groq vision. **Request:** `{ imageBase64, mimeType }`. **Response:** `{ doctorName, medicines[], rawText }`

### `GET /api/session/:sessionId`
Returns the live transcript from the in-memory store. **Response:** `{ transcript, sessionId }`

### `POST /api/session/:sessionId/save`
Persists `{ soap, prescriptions, rawTranscript }` to MongoDB and marks the session `completed`.

### `GET /api/visits` / `POST /api/visits`
List visits (by role) / create a session. POST: `{ patientName, patientEmail?, patientId? }` → `{ sessionId, visitId }`

### `GET /api/patients/match`
Patient autocomplete. **Query:** `email` (exact) or `q` (partial name, ≥2 chars). **Response:** `{ matchType, patient, visits[], matches[] }`

### `POST /api/auth/setup`
Writes role to Clerk `publicMetadata` + upserts MongoDB `User`. **Request:** `{ role: "doctor" | "patient" }`

### `GET /api/prescriptions` / `POST /api/prescriptions`
List or create patient prescriptions (incl. OCR-extracted medicines).

---

## Database Schema

### User
```js
{ clerkId: String, name: String, email: String, role: "doctor" | "patient", avatarUrl: String }
```

### Visit
```js
{
  sessionId: String,                 // UUID — live session key
  doctorId: ObjectId → User,
  patientId: ObjectId → User,
  status: "live" | "completed",
  rawTranscript: String,
  soap: {
    subjective: { chiefComplaint, history },
    objective:  { vitals, examination },
    assessment: String,
    plan:       String,
  },
  prescriptions: [{ drug, dose, frequency, duration }],
}
```

### Prescription
```js
{
  patientId: ObjectId → User,
  visitId:   ObjectId → Visit,       // optional (null for manual uploads)
  imageUrl:  String,                 // Cloudinary URL or base64 fallback
  doctorName: String,
  extractedMeds: [{ drug, dose, frequency, duration }],
  rawOcrText: String,
  uploadedAt: Date,
}
```

---

## Environment Variables

Create `.env.local` in the project root:

```env
# ── Clerk Auth ──────────────────────────────────────────────────
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/

# ── Groq (STT + LLM) ─────────────────────────────────────────────
GROQ_API_KEY=gsk_...                # https://console.groq.com

# ── MongoDB Atlas ────────────────────────────────────────────────
MONGODB_URI=mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/medscribe

# ── Hindsight Cloud (Agent Memory) ───────────────────────────────
HINDSIGHT_BASE_URL=https://api.hindsight.vectorize.io
HINDSIGHT_API_KEY=hsk_...           # https://ui.hindsight.vectorize.io → Settings → API Keys

# ── Cloudinary (Prescription Image Storage) ──────────────────────
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=...
CLOUDINARY_API_SECRET=...
```

---

## Getting Started

### Prerequisites
- Node.js 20+
- [MongoDB Atlas](https://cloud.mongodb.com) cluster (free M0 works)
- [Groq](https://console.groq.com) API key
- [Clerk](https://clerk.com) application
- [Hindsight Cloud](https://ui.hindsight.vectorize.io) account
- [Cloudinary](https://cloudinary.com) account

### Installation
```bash
git clone <your-repo-url>
cd soapmaker
npm install
cp .env.local.example .env.local   # then fill in all values
npm run dev
```
Open [http://localhost:3000](http://localhost:3000).

### First Run Walkthrough
1. `/sign-up` → create a **Doctor** account → `/choose-role` → **Doctor**.
2. `/dashboard` → **New Session** → enter a patient name → **Start Session**.
3. Tap the mic, speak about symptoms for ~15s, then stop.
4. Watch the SOAP note + prescription appear in ~2s.
5. **Share with Patient** → copy the link → open in an incognito window to see the patient live view.
6. **End & Save** → the session appears in dashboard history.

---

## Scripts

| Command | Description |
|---------|-------------|
| `npm run dev` | Start dev server |
| `npm run build` | Production build |
| `npm run start` | Start production server |
| `npm run lint` | Run ESLint |
| `npm run seed` | Seed the database (`scripts/seed.mjs`) |

---

## Notable Implementation Details

### WebM Chunking Fix
The Web `MediaRecorder` API stores EBML container headers **only in the first chunk**; sending later chunks alone returns a "not a valid media file" 400. **Fix (`RecordButton.jsx`):** all chunks accumulate in a ref, and each interval re-sends the full audio as `new Blob(chunksRef.current, { type: mimeType })`. Groq Whisper always receives a complete file, so the store uses `setTranscript` (replace), not append.

### MIME Type Detection
Browsers vary in codec support. `RecordButton.jsx` probes in order: `audio/webm;codecs=opus` → `audio/webm` → `audio/ogg;codecs=opus` → `audio/mp4`. The filename sent to Groq is derived from the chosen MIME type.

### Tailwind v4 Arbitrary Values
Tailwind v4 omits the `8` step in its default opacity scale — use bracket syntax: `bg-white/[0.08]` ✓ (not `bg-white/8` ✗, which is silently ignored).

### Google Fonts in Next.js 16
CSS `@import url(...)` is silently dropped by Tailwind v4's pipeline. Fonts load via `next/font/google` in `layout.js`, exposing CSS variables (`--font-barlow`, `--font-instrument`).

### Next.js 16 Breaking Changes
- Clerk's integration file is **`proxy.js`**, not `middleware.js`.
- Dynamic route `params` are Promises — `await` in server components, or `use(params)` in client components.
- All Groq LLM calls use `response_format: { type: "json_object" }` for parseable output.

---

## License

MIT
