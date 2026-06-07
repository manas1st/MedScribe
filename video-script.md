# MedScribe — 3-Minute Demo Video Script

> Screen recording + voiceover. Talk conversationally, don't read verbatim. Replace `[YOUR NAME]`.
> Increase editor/terminal font size, close notifications, 1080p minimum.

---

## 5 YouTube Title Options

1. I taught an AI to write medical notes like a specific doctor
2. This AI scribe learns how each doctor writes — here's how
3. Whisper mangles medical Hindi — here's how I fixed it before the LLM sees it
4. My medical AI started prescribing in TDS and Hinglish on its own
5. Building an AI scribe that cleans, learns, and remembers how you work

---

## Script

### 1. Quick intro — 30 sec
**On screen:** MedScribe landing page (`/`), then the doctor dashboard (`/dashboard`).

> "Hi, I'm [YOUR NAME]. This is MedScribe — a voice scribe for doctors. A doctor just talks to their patient, and it writes a structured clinical note: symptoms, exam, assessment, plan, and the prescription. No typing. What I want to show you isn't the transcription, though — it's that this thing learns how each individual doctor writes, and it does it without fine-tuning anything."

### 2. Show the problem — 30 sec
**On screen:** Open `src/lib/prompts.js`, highlight `SOAP_SYSTEM_PROMPT`.

> "Here's the problem I started with. One system prompt, one house style. So every doctor got the exact same note — same length, same drug notation, same phrasing. A pediatrician and a cardiologist document completely differently, and my scribe flattened all of that into one voice. Doctors kept saying: 'this is close, but it doesn't sound like me.'"

### 3. Live demo — 2 min
**On screen:** Start a new session from the dashboard → `/session/:id`. The screen shows just the mic button and the live transcript area — nothing else.

> "So let me run a real consultation. The page is deliberately minimal while recording — just the mic, the waveform, and the live transcript. No distractions."

**Action:** Hit record, talk through a ~15-second Hindi/Hinglish sample consultation, then hit stop.

> "I stop the mic, and now watch what happens — there are two steps before the note appears."

**On screen:** The two-step progress indicator lights up: first "Cleaning transcript…" (bouncing dots), then "Generating SOAP note…".

> "First, the raw Whisper transcript goes through an AI cleanup pass. Whisper is great, but it phonetically mangles medical English when doctors speak it in Hindi — so 'ECG' comes out as 'एसी जी', 'Aspirin' as 'आस्प्रिेंट', 'ACS' as 'एसी एस'. The cleaner has a lookup table of these patterns and fixes them before the note ever gets written. Step two is the actual SOAP generation — from a clean transcript, not a garbled one."

**On screen:** SOAP note and prescription card slide up onto the screen with the animation.

> "And there's the note — it slid in from below rather than being there from the start, so the doctor knows exactly when it's ready. Every field is editable inline."

**On screen:** Edit one field in the SOAP note (e.g., the Plan section). Then click "End & Save". The post-save bottom sheet slides up showing two buttons: "Print SOAP" and "Dashboard".

> "After the doctor reviews and edits, they hit End & Save. Instead of immediately bouncing them to the dashboard, the app pauses and asks: do you want to print first? Clicking Print SOAP opens a clean document window — no navigation, no dark theme, no UI chrome — just the formatted note ready for the printer or a PDF."

**On screen:** Show the print window briefly — clean serif layout with S/O/A/P sections.

> "Now here's the interesting part — what happens behind that 'stop' button from the memory side."

**On screen:** Open `src/lib/hindsight.js`. Scroll to `recallDoctorMemory`.

> "Before the model writes anything, I ask Hindsight — that's the open-source agent memory layer — what it already knows about this doctor's style. Notice the query is just natural language: writing style, drug notation, follow-up intervals. Hindsight does the relevance work. And it's wrapped so if memory's ever down, it returns an empty list and we just fall back to the generic note. Memory never blocks a clinical note."

**On screen:** Open `src/lib/prompts.js`, show `buildPersonalizedPrompt` — the `DOCTOR PREFERENCES` block.

> "Those recalled memories get injected right here, as a preferences block in the prompt. And this last line matters — apply them subtly, but if the transcript contradicts a preference, follow the transcript. Memory is a prior, not a law."

**On screen:** Back to `hindsight.js`, show `retainSessionPatterns`.

> "Then after the note is generated, I summarize the session — in plain prose, not a struct — and hand it to Hindsight to remember. I deliberately store natural language and let Hindsight's extraction decide what's worth keeping, so it picks up patterns I never explicitly coded."

**On screen (the payoff):** Show two notes side by side — a day-one generic note vs. a note after several sessions.

> "And this is the before-and-after. Day one: 'Advise rest, paracetamol three times daily, follow up if needed.' After about ten sessions with a doctor who writes terse plans and uses standard abbreviations: 'Symptomatic care. Review after 3 days. Paracetamol 500 TDS.' I never told it to write short plans or use TDS. It watched, Hindsight remembered, and the next note bent toward that doctor's style."

### 4. One key takeaway — 30 sec
**On screen:** The full pipeline: `clean transcript → recall → inject → generate → retain`, shown across `src/app/api/clean-transcript/route.js` and `src/app/api/structure/route.js`.

> "Here's what surprised me building this: there were actually two garbage-in problems to solve, not one. The first was Whisper mangling medical terms — fixed with a targeted cleaning pass before the LLM ever sees the text. The second was personalization — fixed by storing plain prose and letting the memory layer decide what mattered, rather than me extracting structured preferences myself. Both problems had the same shape: clean the input, then let the model do what it's good at. Personalization turned out to be a memory problem, not a training problem. Thanks for watching."

---

## Thumbnail prompt (for Google Nano Banana, 16:9)

> Attach a photo of one or more team members, then use:

"Generate a viral 16:9 YouTube thumbnail for a video about an AI medical scribe that learns each doctor's writing style. Show a confident developer (use the attached photo) on one side, and on the other a glowing 'AI memory' / brain-network visual connecting to a clinical note. Bold large text reading 'AI THAT LEARNS HOW YOU WORK'. High contrast, punchy colors, attention-grabbing, the kind of thumbnail someone scrolling would want to click."
