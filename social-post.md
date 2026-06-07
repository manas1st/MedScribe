# LinkedIn Post

> Publish after your article is live. Put your **GitHub repo link in the main post**, then add the Hindsight repo as the **first comment**. Replace `[ARTICLE_URL]` and `[GITHUB_REPO_URL]`.

---

Most "personalized" AI agents aren't personalized. They're one prompt pretending to know you.

I built a voice medical scribe that turns a doctor-patient conversation into a structured clinical note. The first version worked — and wrote the exact same note for every doctor. Generic. Forgettable.

The fix wasn't a bigger model or per-user fine-tuning. It was memory.

Here's what changed once I added Hindsight agent memory:

→ Before: every doctor got the same verbose, house-style plan.
→ After: a doctor who writes terse plans gets terse plans. One who prescribes by salt name gets salt names. Nobody configured this.

What I'd copy into your next agent:

1. Recall before you generate, retain after. Two function calls bracketing one LLM request.

2. Store prose, not schemas. I hand Hindsight a plain-language session summary and let its extraction decide what matters — so it surfaces patterns I never coded for.

3. Scope memory per user. One bank ID per doctor. No cross-contamination.

4. Make memory an enhancement, never a dependency. Recall fails → fall back to generic. A memory outage should never block real work.

5. Treat preferences as a prior, not a law. If the current input contradicts a stored preference, the input wins.

Choosing Hindsight for the memory layer meant I skipped building a preferences table, an extraction pipeline, and a relevance ranker. Three lines and a bank name got me there.

Personalization is a memory problem, not a training problem.

Full write-up + code in the comments 👇

#AIAgents #AgentMemory #Hindsight #LLM #AI

---

## First comment (post immediately after)

Full article: [ARTICLE_URL]

Project repo: [GITHUB_REPO_URL]

And the agent memory layer that made it work — Hindsight is open source: https://github.com/vectorize-io/hindsight
