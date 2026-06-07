# How I taught an AI scribe to write like each doctor with Hindsight

The first version of MedScribe wrote a perfectly competent SOAP note. The problem was that it wrote the *same* competent SOAP note for everyone.

A pediatrician in Pune and a cardiologist in Chennai do not document the same way. One writes a three-line plan; the other writes a paragraph. One prescribes generics by their salt name, the other reaches for brand names. One dictates in clean English, the other slips into Hinglish halfway through the consultation. My model flattened all of that into one house style, and every doctor who tried it had the same reaction: *this is close, but it doesn't sound like me.*

This is the story of how I fixed that without fine-tuning a single model, by giving every doctor a private memory that the scribe reads before it writes and updates after every visit. The memory layer is [Hindsight, an open-source agent memory system from Vectorize](https://github.com/vectorize-io/hindsight), and adding it changed the product from "an LLM that writes notes" into "an LLM that writes *your* notes."

## What MedScribe does

MedScribe is a voice-first clinical scribe. A doctor opens a session, taps the mic, and just talks to the patient. The audio streams to a speech-to-text model in short intervals; when the doctor stops, the full transcript goes to an LLM that returns a structured SOAP note (Subjective, Objective, Assessment, Plan) plus a parsed prescription list. The doctor can edit any field inline and save. The patient, meanwhile, can open a shared link and watch the transcript and summary on their own phone.

The stack is deliberately boring where it can afford to be: Next.js on the front and back, Groq for both transcription and the SOAP generation (it's fast enough that the note appears a second or two after the doctor stops talking), MongoDB for persistence. The interesting part — the part that took the product from a clever demo to something a doctor would actually keep using — is the memory layer.

## The core problem: personalization without fine-tuning

The obvious way to make the scribe sound like a specific doctor is to fine-tune a model per doctor. That's a non-starter. You'd need a labeled corpus of each doctor's notes, a training pipeline per user, and a deployment story for thousands of small models. None of that survives contact with a real clinic.

The cheaper idea is to put the doctor's preferences in the prompt. But where do those preferences come from? Nobody is going to fill out a "my documentation style" form, and even if they did, they couldn't articulate that they always write follow-up intervals as "review after 5 days" or that they abbreviate three-times-daily as TDS, not "8 hourly." Those patterns live in their behavior, not in their self-description.

So the real requirement is: *observe* how a doctor documents, *remember* it, and *apply* it later — automatically, per user, and without me hand-writing rules. That is exactly the shape of an [agent memory problem](https://vectorize.io/what-is-agent-memory), and it's why I reached for Hindsight instead of building a bespoke "preferences" table.

## How Hindsight fits

Hindsight gives you two primitives that map cleanly onto the problem: `retain` (store something worth remembering) and `recall` (fetch what's relevant right now). Crucially, you don't store rigid key-value preferences — you hand it natural language, and its own extraction layer turns prose into retrievable facts. That detail ended up being the whole trick, and I'll come back to it.

The first decision was scoping. Memory has to be per-doctor, never shared, so I key every bank to the doctor's user ID:

```js
/** Bank ID scoped to a single doctor */
export function doctorBank(userId) {
  return `medscribe-doctor-${userId}`;
}
```

One doctor, one bank. There's no path by which one doctor's style can leak into another's notes, which matters a lot when the data is downstream of medical consultations.

### Recall before you generate

Before the LLM writes anything, I ask Hindsight what it knows about this doctor's style:

```js
export async function recallDoctorMemory(userId) {
  const client = getClient();
  if (!client) return [];

  try {
    const bank = doctorBank(userId);
    const result = await client.recall(
      bank,
      "SOAP note writing style, prescription preferences, drug notation, follow-up intervals, language style",
      { budget: "mid", maxTokens: 800 }
    );
    return (result?.results ?? []).map((r) => r.text).filter(Boolean);
  } catch (err) {
    console.warn("[Hindsight] recall failed (non-fatal):", err?.message ?? err);
    return [];
  }
}
```

Two things I'd call out here. First, the query is a natural-language description of *what kind* of memory I want, not an exact-match key — Hindsight does the relevance work. Second, the whole thing is wrapped so that a memory outage can never block a clinical note. If Hindsight is slow or down, `recallDoctorMemory` returns `[]` and the scribe falls back to its generic-but-correct behavior. Memory is an enhancement, never a dependency on the critical path.

The recalled memories get folded into the system prompt as an explicit preferences block:

```js
export function buildPersonalizedPrompt(memories = []) {
  if (!memories.length) return SOAP_SYSTEM_PROMPT;

  const memoryBlock = memories
    .slice(0, 6)
    .map((m, i) => `  ${i + 1}. ${m}`)
    .join("\n");

  return `${SOAP_SYSTEM_PROMPT}

DOCTOR PREFERENCES (learned from past sessions — apply these when generating the note):
${memoryBlock}
Apply these preferences subtly. If the transcript contradicts a preference, follow the transcript.`;
}
```

That last line is load-bearing. Preferences are a prior, not an override — if the doctor clearly says something different in *this* consultation, the transcript wins. Memory should bias the output, not fight the present.

### Retain after you generate

After the note is generated, I summarize the session and hand it to Hindsight. The key design choice: I store **prose**, not a struct.

```js
export function retainSessionPatterns(userId, { soap, prescriptions, transcript }) {
  const client = getClient();
  if (!client) return;

  const date  = new Date().toISOString().slice(0, 10);
  const drugs = (prescriptions ?? [])
    .map((p) => `${p.drug} ${p.dose} ${p.frequency}`)
    .join("; ") || "none";
  const plan  = soap?.soap?.plan ?? "";

  const content = `
MedScribe session on ${date}.
Plan: ${plan}
Prescriptions: ${drugs}
Transcript language mix: ${detectLanguageMix(transcript)}
SOAP plan length: ${plan.split(" ").length} words (${plan.split(" ").length < 30 ? "concise" : "detailed"})
Drug style: ${detectDrugStyle(prescriptions)}
  `.trim();

  client
    .retain(bank, content, {
      context: "doctor session pattern",
      tags: [`doctor:${userId}`, "session"],
    })
    .catch((err) => console.warn("[Hindsight] retain failed (non-fatal):", err));
}
```

I went back and forth on whether to store structured facts — a JSON blob of `{ planLength: "concise", drugStyle: "brand" }`. Storing prose won, for one reason: I don't actually know in advance which signals matter. By describing the session in plain language and letting Hindsight's extraction decide what's worth keeping and surfacing, I get to discover patterns I never explicitly coded for. The light heuristics here (language mix, plan length, brand-vs-generic) are just hints to make the prose richer; Hindsight does the remembering. And like recall, retain is fire-and-forget — it runs off the critical path, so storing a memory never adds latency to the note the doctor is waiting on.

## What it actually does, before and after

Here's the behavior change that made the difference real.

**Before any memory (a brand-new doctor, empty bank):**

> **Plan:** Advise rest and adequate hydration. Prescribe paracetamol for fever. Recommend follow-up if symptoms persist.
>
> **Prescription:** Paracetamol, 500mg, three times daily, 5 days.

Correct, generic, slightly verbose.

**After ten-odd sessions with a doctor who writes terse plans, uses standard abbreviations, and prescribes by salt name:**

> **Plan:** Symptomatic care. Review after 3 days.
>
> **Prescription:** Paracetamol 500mg TDS x 5d.

Nobody told the model "this doctor writes short plans" or "use TDS." It watched, Hindsight remembered, and the next note bent toward that style. When that same doctor's transcript is half in Hindi, the recalled language-mix memory nudges the note's terminology choices too — without ever changing the rule that the final SOAP note is always written in English.

The flow end to end is just: recall → inject → generate → retain.

```js
const memories     = isDoctor ? await recallDoctorMemory(userId) : [];
const systemPrompt = buildPersonalizedPrompt(memories);
const completion   = await groq.chat.completions.create({
  model: "llama-3.3-70b-versatile",
  messages: [
    { role: "system", content: systemPrompt },
    { role: "user",   content: `Transcript:\n${transcript}` },
  ],
  response_format: { type: "json_object" },
});
// ...parse the SOAP JSON, return it, then:
if (isDoctor) retainSessionPatterns(userId, { soap, prescriptions, transcript });
```

The personalization lives in two function calls bracketing one LLM request. That's the entire mechanism.

## Lessons I'd reuse on the next project

**Store prose, not schemas, when you don't yet know what matters.** My instinct was to extract structured preferences myself. Handing Hindsight natural-language summaries and letting its extraction layer decide what's salient meant the system could surface patterns I never anticipated. You can always tighten later; you can't recover signal you threw away at write time.

**Make memory an enhancement, never a dependency.** Every recall and retain is wrapped so a failure degrades gracefully to the generic behavior. A memory service hiccup should never be the reason a doctor can't get a note. Designing for "memory is down" from day one kept the critical path honest.

**Recall before you generate; retain after.** This two-beat rhythm is the reusable pattern. Fetch relevant context, fold it into the prompt as a clearly-labeled prior, do the work, then write back what you learned. It generalizes far past medical notes — any agent that should get better at serving a specific user can use the same loop.

**Personalization is a memory problem, not a training problem.** I avoided per-user fine-tuning entirely and got something better: a system that adapts within a handful of sessions, requires no labeled data, and improves continuously. For most "make it sound like me" requirements, [agent memory](https://hindsight.vectorize.io/) is the cheaper and faster answer than touching model weights.

**Treat preferences as a prior, not a law.** The instruction to follow the transcript when it contradicts memory prevents the classic failure where a system becomes a caricature of its own history. The present should always be able to override the past.

MedScribe still does the obvious thing — turn a conversation into a clean clinical note. But the version doctors keep using is the one that, after a week, stops sounding like a generic AI and starts sounding like them. That difference is one recall call, one retain call, and a memory bank named after the person it serves.
