 Personal AI Assistant — Self-Hosted Agent Stack

A private, always-on AI assistant running on my own cloud server, reachable from my phone
over WhatsApp. Built and maintained by me over three months (June–September 2026) —
607 sessions and counting.

I'm an operations manager, not a software engineer. I built this by describing what I
wanted in plain language to AI coding assistants, then reading, testing, and debugging
the results until they worked. Everything here was earned the hard way.

**This repo is an architecture and engineering-decisions write-up.** It is not a
deployable copy — no hostnames, addresses, ports, credentials, or personal data.

---

## Contents
- [What it does](#what-it-does)
- [How I built it](#how-i-built-it)
- [Architecture](#architecture)
- [From "claw" to Hermes](#from-claw-to-hermes)
- [Tools and stack](#tools-and-stack)
- [Monthly cost](#monthly-cost)
- [Planned vs actual](#planned-vs-actual--the-most-useful-section)
- [What I had to fix myself](#what-i-had-to-fix-myself)
- [Engineering decisions](#engineering-decisions)
- [Current state — honest](#current-state--honest)
- [What I learned](#what-i-learned)

---

## What It Does

- Answers questions and runs tasks from a phone message — including reading photos and
  documents (bills, scans) and extracting the content
- Organises incoming mail, tracks a stock portfolio and recurring expenses, and pushes
  scheduled digests unprompted (morning brief, email watch, bill scans)
- Searches the web privately through a self-hosted, non-tracking search instance; works
  in multiple languages
- Takes delegated jobs from a second, cloud-based scheduler agent through a shared task
  mailbox

---

## How I Built It

I'm an operations manager. I don't write production code — I write precise plain-language
specifications, hand them to AI coding assistants, then read every diff, test every
behaviour, and debug every failure myself. The agent runs the code; I run the agent.

That loop — specify, read, test, break, fix — is the whole methodology. It works because
I treat the AI's output like a junior engineer's pull request: never merged blind.

---

## Architecture

```
Phone (WhatsApp)
        │
        ▼
  Agent runtime (Hermes, Nous Research)
  containerised on a cloud VPS
        │
        ├─►  Model router ────────────────► Primary : hosted API, cheap & fast
        │                                  ├─ Fallback: second hosted provider
        │                                  └─ Offline : local model (Ollama)
        │
        ├─►  Vector database (Qdrant)        — semantic recall over notes and mail
        ├─►  Structured database (NocoDB)    — portfolio, expenses, bills, health logs
        ├─►  Self-hosted metasearch         — private web search, no tracking
        ├─►  Connector layer (MCP)          — filesystem, cloud docs, web search
        ├─►  Scheduled jobs (crons)         — morning brief, watchers, bill scans
        │
        └─►  Task mailbox (cloud drive)  ◄──►  Cloud scheduler agent
             queue/ + done/ folders            (does the heavy thinking on a free tier;
             30-second poller, API-key auth     currently paused — see Current state)
             PII approval gate built in
```

**Design principle:** it must keep working when any single provider fails, and must never
be *only* dependent on something I don't control. That's why a local model sits at the
bottom of the chain, and why the server is reachable only over a private VPN — never the
open internet.

---

## From "claw" to Hermes

I started on a different agent framework (my "claw" setup) and later migrated everything
to the Hermes agent, keeping the same phone number and chat interface. Config format
changed, the plugin model became a skills model, and two legacy skills were carried
across rather than rewritten.

The test of a migration: from my phone, nothing changed.

---

## Tools and Stack

| Layer | Tool |
|---|---|
| Agent runtime | Hermes (Nous Research), containerised |
| Server | Cloud VPS, ~12 GB RAM (downgraded from 24 GB after measuring real usage) |
| Primary model | Hosted API (cheap, fast) |
| Fallback model | Second hosted provider |
| Local model | qwen3.5:9b via Ollama (research, summarisation — free) |
| Vector memory | Qdrant |
| Structured data | NocoDB |
| Web search | Self-hosted metasearch + DuckDuckGo MCP |
| Phone interface | WhatsApp |
| Network | Private VPN only; firewall locked to VPN + localhost |
| Task bridge | Cloud-drive mailbox + Python poller (30 s loop, file lock, API-key auth) |
| Behaviour | "Skills" — plain-language instruction files, not code |

---

## Monthly Cost

Rough, as of September 2026:

| Item | Cost |
|---|---|
| Cloud VPS | ~€13/mo (about ₹1,200) — was ~€44/mo before downsizing |
| Primary LLM API | Pay-as-you-go, hard-capped at ~$0.05 per session |
| Fallback LLM | Free tier |
| Local model | €0 |
| Search / databases / connectors | Self-hosted, €0 |
| **Total** | **~€13–15/mo (about ₹1,200–1,400)** |

The local model does all research and summarisation for free; the paid model only handles
live conversation. That single rule is most of the cost control.

---

## Planned vs Actual — The Most Useful Section

I wrote a full plan before buying anything. Three months later, most of it was wrong.
Keeping both versions is the single most useful thing in this repo.

| | Planned (month 0) | Actual (month 3) | Why it changed |
|---|---|---|---|
| **Server** | 24 GB RAM, 3 dedicated cores, 180 GB | ~12 GB RAM, 96 GB | Measured steady-state usage at a fraction of capacity. Downsized a tier: **~€44/mo → ~€13/mo**, no capability loss. |
| **Main model** | 27B open model, local, "free" | Hosted API model as primary | 27B on CPU took ~2 minutes per reply. Unusable for chat. A cheap hosted model answers in seconds. "Free" cost more than it saved. |
| **Local models** | Four (general, vision, multilingual, embeddings) | One 9B model | One model that does all three beats four that swap in and out of RAM. |
| **Model selection basis** | Raw capability | **Tool-calling reliability** | For an agent, reliably calling the right tool matters more than raw intelligence. |
| **Agent platform** | First framework ("claw") | Migrated to Hermes | Better fit; migration kept the phone interface identical. |
| **Voice input** | Day-1 feature | Still not installed | Kept getting deprioritised. Listed as an honest gap rather than quietly dropped. |
| **Cost of "free" tiers** | Assumed viable | Mostly not | Free API tiers have per-minute token caps too low for an agent's fixed per-request overhead. |

**The lesson:** every assumption I made about performance was wrong in the same direction
— I underestimated how slow local inference is and overestimated how usable free tiers are.

---

## What I Had to Fix Myself

The real content. Each of these cost me hours. The dated ones trace to the session
record (607 sessions) — nothing reconstructed from memory.

### 1. The entire fallback chain was dead on arrival (Jun 8)
**Symptom:** a scheduled job failed — and investigation showed *every* provider in the
chain was broken simultaneously: stale connection on the primary, dead key on the
second, a typo'd key on the third, model not loaded on the local one. Tens of thousands
of tokens burned for zero output.
**Fix:** keys wired into every provider via environment config (they existed but weren't
referenced); silent-provider timeout cut from 180s to 30s; local model preloaded with an
infinite keep-alive.
**Lesson:** a fallback chain is only as good as its key wiring. Verify every hop, not
just the primary.

### 2. A ten-cent token burn exposed the cost model (Jul 24)
**Symptom:** a routine email-classification task burned $0.10 of paid tokens.
**Cause:** the free local classifier failed *silently*, so the agent fell back to dumping
40+ full emails into paid context. The two-model design was correct; the failure path
wasn't.
**Fix:** hard cap of ~$0.05 per session; new rule — if the free model fails, *narrow the
input*, never bulk-dump into the paid model; all research and summarisation moved to the
free local model permanently. A purpose-built email-scan pipeline cut one job's token
cost by ~93%.
**Lesson:** "free" is a cost model, not a price — and silent failures bill you the most.

### 3. The nightly reader took four iterations to get right (Jul 7–15)
**Symptom:** a script meant to deliver one authentic text excerpt per night sent footnote
fragments, then nothing (the local model couldn't meet the agent's context minimum),
then commentary, then repeats.
**Fix:** rewrote the extractor to pull one clean unit at a time, and — the actual fix —
**reordered the sources by data quality instead of prestige**, starting with the cleanest
corpus.
**Lesson:** order your sources by data quality, not by prestige. OCR text with mixed
footnotes and page numbers defeats naive parsers.

### 4. The task bridge shipped with four real bugs (Aug 2)
**Symptom:** the poller connecting my cloud scheduler to the home agent would have wedged
jobs forever, sent malformed payloads, and double-pulled.
**Fix:** acknowledgements nested correctly, payload shape corrected, a real API key
(rotated afterward), and a file lock so a second instance waits instead of crash-looping.
Measured honestly afterwards: ~2 minutes from scheduler write to execution.
**Lesson:** test the contract between two systems, not just each system alone.

### 5. "The agent is down" — it wasn't (Aug 3)
**Symptom:** the scheduler reported the home agent down with jobs piling up.
**Truth:** the poller had processed everything overnight. The stuck job referenced a
script that didn't exist, under a name the bridge didn't recognise, with parameters in
the wrong field.
**Fix:** built the missing script, taught the poller to translate job names and
parameter shapes, and made the result-delivery path a universal contract.
**Lesson:** the most expensive mistake in operations is believing a status report instead
of checking the log.

### 6. The privacy gate flagged its own prohibition (Aug 3–5)
**Symptom:** a nightly job that explicitly *forbade* account numbers and UPI IDs kept
getting held for containing the words "account number" and "UPI".
**Cause:** the keyword filter matched terms *inside the prohibition clause itself*.
**Fix:** a negation-aware, clause-scoped filter — and the gate still fails *closed*,
because false positives are the safe failure mode at 3 AM.
**Lesson:** when you automate a rule, test it against the rule's own wording.

### 7. One corrupt file crash-looped the whole queue (Aug 18)
**Symptom:** a malformed job file restarted the poller ~14 times while new jobs piled up
behind it — 31 files deep before I called a halt.
**Fix:** stopped everything, quarantined by **copying, never deleting**, reset the state
file. Nothing destroyed; everything recoverable.
**Lesson:** copy before you delete, always — and a queue needs a dead-letter path, not
just retries.

### 8. Replies got slower over weeks
**Symptom:** response time degraded from seconds to minutes over a fortnight.
**Cause:** session history accumulated until requests crossed the primary provider's
limit and silently fell through to a much slower fallback. Nothing errored — it just
got worse.
**Fix:** an hourly cleanup job with an enforced size ceiling.
**Lesson:** any system that accumulates state needs an enforced ceiling, not a cleanup
you remember to run.

### 9. Semantic search silently returned nothing
**Symptom:** memory search came back empty. No error, no exception.
**Cause:** the local embedding model unloads itself after idle; background jobs needing
it got nothing back.
**Fix:** a scheduled keep-alive ping plus an explicit keep-alive setting.
**Lesson:** empty results are a *symptom* — treat them as a failure mode, not as "no data".

### 10. Scheduled jobs fired at the wrong times
**Symptom:** a job set for 08:00 delivered at 02:30.
**Cause:** server clock in UTC, my expectations in IST.
**Fix:** verified the timezone and corrected all jobs **in one pass** instead of
discovering each over weeks.
**Lesson:** when several things share one misconfiguration, fix them together.

### 11. The messaging link expires regularly
**Not a bug — a design constraint** of the messaging bridge. Requires a manual re-link
roughly fortnightly.
**Handled by:** accepting it and setting a recurring reminder. Some constraints are
cheaper to accept than to engineer around.

---

## Engineering Decisions

**Multi-tier routing with graceful degradation.** Cheap fast hosted provider first, second
hosted provider as fallback, local model if the network is gone. The assistant degrades
in quality rather than going dark.

**Config over code.** Behaviour lives in dozens of "skills" — mostly plain-language
instruction files. Changing what the assistant does usually means editing a file, not
shipping code. This is the main reason a non-engineer can maintain it.

**Right-sized from measured data.** Ran deliberately oversized for the first weeks,
measured real utilisation, then downsized. **~€44/mo → ~€13/mo.** I resisted doing this
on day one precisely because I had no data yet.

**Migration without changing the interface.** When I moved frameworks, the config format
changed and the plugin model became a skills model — but from my phone, nothing changed:
same number, same behaviour. Two legacy skills were carried across rather than rewritten.
*A migration is successful when the user cannot tell it happened.*

**Self-hosted search over an API.** Unlimited, private, no per-query cost, no third party
seeing my queries.

**Token budget as architecture.** A hard per-session spend cap plus a standing rule —
free/local models do the bulk work, the paid model only converses. Cost control designed
in, not bolted on.

**The privacy gate fails closed.** Anything touching sensitive identifiers is held for my
explicit approval, day or night. False positives are annoying; false negatives are
unacceptable.

**Quarantine by copying, never by deleting.** Anything removed from the queue is copied
aside first. Recoverability beats tidiness.

---

## Current State — Honest

**Working:** agent runtime · 3-tier model routing · local model · vector database ·
structured database · self-hosted search · connector layer · dozens of skills ·
off-box backups · private-VPN-only networking

**Paused:** the cloud-scheduler task bridge has been dark since a queue flood in
mid-August (services stopped deliberately during cleanup; job files quarantined, nothing
deleted — resumable). Its scheduled reports are not generating until it resumes.

**Degraded or missing:** mail body extraction (partially broken) · voice input (never
installed) · a pending agent-platform update (needs a few minutes of downtime, timing not
yet chosen)

I keep a dated audit of what's actually running and treat *that* as the source of truth,
not my own documentation. A README claiming everything works is a README nobody should
trust.

---

## What I Learned

- **Every major failure was a silent failure at a boundary** — between providers, between
  two systems, between a rule and its own wording.
- **Believing a status report instead of checking the log** is the most expensive
  mistake. ("The agent is down" — it wasn't.)
- **Measure, don't inherit.** Every performance figure I'd written down was wrong when I
  re-measured it. A latency number has a half-life.
- **A workaround applied every run is still an open defect.** If the fix isn't written
  back to the source, you pay the cost forever.
- **Silent failures are the expensive ones.** Empty results are harder to find than crashes.
- **Design for the unhappy path.** Almost every decision above is about what happens when
  something fails.
- **"Free" is a cost model, not a price.** Free tiers cost time, and time is the scarcer
  resource.
- **Some constraints are cheaper to accept than to engineer around.**

---

