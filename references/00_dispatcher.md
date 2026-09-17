# Reverse Question Inversion — Technical Operational Dispatcher

**Framework Author**: William Fitzpatrick (*Writer Science*)  
**Source Lecture**: [Writing Online Is Hard, Until Experts Do This](https://www.youtube.com/watch?v=j1JRSan9CYg)  
**Parent Collection**: [Master Collection](../../kirby-fitzpatrick-writers-collection/SKILL.md) | [Global Help](../../../kirby-help/SKILL.md)  

---

## 1. Cognitive Foundation: Echo-First Addressing and the Reader Translation Tax

**The concept.** Every response carries two independent payloads: an **address** and a **body**. The address tells the reader *what this is about*; the body tells them *what is true about it*. Conversational models default to spending the first line on the address — `Great question!`, `Let's break this down`, `There are several factors that can cause…` — and the payload arrives forty words later. In a terminal, that first line is the most expensive real estate in the entire exchange, because the developer reads it while the rest of the response is still streaming.

**Reverse Question Inversion** is the rule that the **address must be the user's own words, occupying response-words 1–3**. A question and its answer have *inverse information geometry*: a question spends its emphasis at the **end** (end-focus — new information goes last), while a usable answer must spend its anchor at the **start**. The inversion is the act of lifting the question's focal constituent out of its tail position and pressing it into the response's head position.

Five load-bearing definitions:

1. **Query Focal Constituent (QFC)** — the head noun phrase the question is *about*, not the words it is *made of*. In `why does my postgres query time out under load?` the QFC is `postgres query timeouts`. In `does the mutex cover the read path?` the QFC is `the mutex`. Extract the QFC before writing a single token of the answer.
2. **Echo Window** — the first **three orthographic tokens** of the response. Not three clauses, not three lines: three words. Everything before token 4 is address; everything after is body.
3. **Translation Tax** — the cognitive labour imposed when the answer's opening subject differs from the question's topic. The reader must hold the question in working memory, ingest an unrelated opener, then map the opener back onto their own entity. Tax is paid per response, by the human, forever.
4. **Mis-parse Bound** — the cost of the model having latched onto the wrong noun. When the QFC is echoed, a wrong parse is visible *in line 1* and the user aborts for ~5 tokens. When the QFC is displaced, the wrong parse is only discovered after a full confident answer — the most expensive failure mode in an agent loop.
5. **First-Line Survivability** — whether line 1 remains useful when the response is truncated. Truncation is the normal case in terminal chat: `Ctrl+C`, a 502 mid-stream, a scroll-away, a pasted excerpt, `head -1` in a pipe. An echo-first line survives truncation the way a good commit subject survives `git log --oneline`.

Why this binds harder in software engineering than in essays. Terminal chat is a **triage interface**, not a reading surface. The developer arriving with a question has already paid to formulate it; they are scanning for one thing — *did the answer address my entity?* Echo-first makes that decision O(1) instead of O(response length). Four concrete engineering consequences:

- **Fail-fast semantics.** Echo-first converts a semantic misalignment (wrong entity, wrong premise, wrong repo) into a syntactic signal detectable in the first three words. It is a type-check before the runtime, a preflight before the deploy.
- **Scrollback is a queryable log.** `rg -i "timeout" session.log` returns the *answer* only if the answer opens with the topic. Address-displaced transcripts are grep-hostile: the topic exists only in the user's turn.
- **Transcript re-entry.** Terminal output gets pasted into tickets, PR threads, and incident channels stripped of the user's question. A response whose first words name its subject is self-addressing; a response that opens with `Sure!` is orphaned context.
- **Multi-turn anchoring.** The first three words become the **anaphoric handle** for the next turn — `about your point on pool exhaustion…`. If token 1 is `Sure`, the handle is a politeness token and the user has to re-describe their own problem.

**Adjacent protocols — and how this one differs.** Sibling [Cargo-Weighted Syntax](../../kirby-fitzpatrick-cargo-weighted-syntax/SKILL.md) governs where *new* information lands inside the artifact; this skill governs where the *reader's existing* referent lands at the artifact's edge. Sibling [Heres Why Inversion](../../kirby-fitzpatrick-heres-why-inversion/SKILL.md) is **verdict-first for authored artifacts** (commit subjects, PR bodies, ADRs — the terminal state of the work); this skill is **address-first for conversational responses** (the terminal state of the question's topic). They compose: an echo window naming the topic, followed immediately by a verdict. Sibling [Zero Meta Discourse](../../kirby-fitzpatrick-zero-meta-discourse/SKILL.md) bans self-narration; this skill names the specific position where that ban is enforced — words 1–3.

```text
[BEFORE — ADDRESS DECOUPLED FROM PAYLOAD]
  user types:  "why does my postgres query time out under load?"
  terminal ─────────────────────────────────────────────────────────▶ t0

  assistant: "Great question! Let's think about what happens under
              load. There are several factors that can cause a query
              to time out: connection-pool exhaustion, lock waits,
              slow plans, ... "

  words 1-3 = "Great question! Let's"          ← zero information about the topic
  words 4+  = adjacency + taxonomy             ← general where the user needs specific
  the user's own entity ("postgres query timeouts") appears at word 41

  cost model:
    TTV (time-to-verdict)      = 41 words          ← payload walled behind address
    translation tax            = 1 remap, paid once
    mis-parse bound            = the whole response (wrong entity only visible at the end)
    truncation (Ctrl+C / head -1 / 502)  → line 1 is worthless
    rg "timeout" transcript.log          → matches the user's turn, not the answer
    next turn handle           = "Sure"  → user re-explains their own question

[AFTER — THE INVERSION]
  user types:  "why does my postgres query time out under load?"

  assistant: "Postgres query timeouts under load come from connection-
              pool exhaustion: max_connections=100, 140 workers,
              p99 wait 4.8s (pg_stat_activity.wait_event = ClientRead).
              Fix: pool_size=40, statement_timeout=2s."

  words 1-3 = "Postgres query timeouts"        ← the user's entity, echoed verbatim
  word 4+   = verdict → mechanism → measured evidence → action

  cost model:
    TTV                        = 1 sentence (0 meta tokens before the verdict)
    translation tax            = 0                         ← nothing to remap
    mis-parse bound            = 1 rendered line           ← cheap abort, visible at word 1
    truncation                 = line 1 survives standalone
    rg "postgres query" *.log  → finds the answer that names its subject
    next turn handle           = "pool exhaustion"         ← a real referent
```

```text
[THE INVERSION — INVERSE INFORMATION GEOMETRY]

  QUESTION (end-focus: emphasis spent last, the interrogative owns position 1)
    "why does my  POSTGRES QUERY TIME OUT  under load?"
                   └────── QFC ─────────┘ └─ new info ─┘
                          │
                          │  LIFT: demote the interrogative scaffolding ("why does my"),
                          │  promote the QFC into the answer's head position.
                          ▼
  ANSWER (front-anchor: the reader's referent owns position 1–3)
    "POSTGRES QUERY TIMEOUTS under load come from connection-pool exhaustion:
     max_connections=100 · 140 workers · p99 wait 4.8s."
     └────── words 1–3 ───────┘└ verdict ┘└──────── evidence ────────┘

  displacement axis:  user's entity ────────────────────────────▶ is it at token 1–3?
                              yes = read    │    no = pay the translation tax, again
```

---

## 2. Core Transformation Protocols

**Rule 1 — The echo window is three words, and it belongs to the user.** Response tokens 1, 2, 3 carry the QFC. Token 4 opens the predicate. Treat the budget as a hard cap, not a guideline: at 4–6 words the opener has become a restatement, and a restatement in a terminal costs a full line of scrollback to convey nothing the user did not just type.

**Rule 2 — Extract the QFC before you answer.** Parse the query into *(entity, condition, question-type)* before emitting tokens. `why does my postgres query time out under load?` → entity `postgres query timeouts`, condition `under load`, type `why/causal`. The response then has a fixed shape: `entity + condition → cause → evidence`. Skipping extraction is what produces `There are several factors…` — the model answering its own generic prompt instead of the user's specific one.

**Rule 3 — Demote interrogative scaffolding, promote the subject.** The question's `why does my` / `how do I` / `is my` is *scaffolding*, not subject matter. Drop it. `why does my X fail?` → `X fails because…`, never `The reason why your X fails is…`. Word 1 is the entity, word 2–3 the predicate that resolves it.

**Rule 4 — Yes/no queries invert polarity into the echo window.** For `does/is/can/will` questions, position 1–3 holds the QFC and the *auxiliary-verb answer* lands at or before word 4: `The mutex does not cover the read path…`, `The flag is set…`. Never open a yes/no answer with `That depends…` — the user asked a binary question and the polarity is the load-bearing token.

**Rule 5 — Zero meta-tokens in the echo window.** Banned at words 1–3: `Sure`, `Certainly`, `Absolutely`, `Of course`, `Great question`, `Good question`, `Interesting`, `Thanks for asking`, `Let's`, `I'd be happy to`, `Let me`, `To answer your question`, `Short answer`, `Hope that helps`, `Hmm`, `It seems`, `I think`. Acknowledgment is not deleted — it is **relocated after the payload** (`Want the migration patch? It's two lines.`). Politeness may open a response only after the answer has been given.

**Rule 6 — Conditional answers echo first, then name the axis.** `It depends` is an address failure because it names nothing. Echo the topic, then branch: `Retry behaviour depends on which layer: the SDK retries 3× (client.go:151), the gateway retries 0×. The p99 you're seeing comes from the SDK.` The branch point moves to word 4; the topic never leaves word 1.

**Rule 7 — When the premise is wrong, echo the corrected entity.** Inversion is not obedience — it is accurate addressing. If the user asks about X and X is not the cause, the echo window holds the *true* entity: `The mutex isn't the constraint — the connection pool is: 140 workers against max_connections=100.` The wrong entity is named in the same line, so the user sees both the correction and the fact that their question was read.

**Rule 8 — Echo terms verbatim, never synonyms.** Keep the user's identifiers character-for-character: `CrashLoopBackOff`, `ENOENT`, `pg_stat_activity`, `TestRetryBudget`. Paraphrase (`the pod keeps restarting`) breaks greppability and, worse, silently re-identifies the artifact — a synonym in position 1 reads as a different bug.

**Rule 9 — No process narration before the echo.** `Let me look at the codebase…`, `I'll search for the config…` are tok ens spent on the reader's behalf with nothing to show. Do the work, then emit: `Retry budget is hard-coded at retry.go:88 — found it, no config key exists.` Tool-call narration belongs after the finding, if at all.

**Rule 10 — Never hedge inside the echo window.** `Might possibly be an issue with your query plan` spends the address on doubt. Echo the entity and let the predicate carry the confidence: `The query plan is the issue — EXPLAIN ANALYZE shows a seq scan over 2.4M rows, 3.1s.` Calibrate the claim, not the address.

**Rule 11 — The echo must survive truncation.** Test every response with `head -1`: if the first line were all the user ever saw, would they know the topic, the verdict, and the next action? Echo-first is what makes line 1 a standalone unit — the conversational equivalent of a commit subject line.

**Rule 12 — One response, one QFC; rank the rest.** Multi-topic queries (`also, why is CI red?`) get their primary entity in the echo window and the secondary in an explicit block below — `Also, CI is red: …`. Never average two entities into a topic-less opener.

### Transformation table

| # | Anti-pattern (address displaced) | Inversion replacement | Mechanism |
|---|---|---|---|
| 1 | "Great question! Let's think about what happens under load." | "Postgres query timeouts under load come from connection-pool exhaustion: max_connections=100, 140 workers, p99 wait 4.8s." | Adjacency tokens → QFC in the echo window, verdict at word 4. |
| 2 | "There are several factors that can cause a query to time out." | "Postgres query timeouts trace to three causes, in the order `pg_stat_activity` shows them: pool exhaustion (this one), lock waits, slow plans." | Generic taxonomy → named entity + ranked branch list. |
| 3 | "Let me search the codebase to check the retry logic." | "The retry budget is hard-coded at `retry.go:88` — there is no config key." | Process narration → finding, with the search implied. |
| 4 | "I'd be happy to help with your Docker build." | "Docker build fails at `npm ci` — `ENOENT package-lock.json`; `.dockerignore` excludes it. Add `!package-lock.json`." | Service offering → failure point + cause + one-line fix. |
| 5 | "It depends." | "Mutual-exclusion coverage depends on which call: `get()` reads `m.items` unlocked (`cache.go:41`), `getBatch()` holds the lock." | Naked hedge → echo + named branch axis. |
| 6 | "Might possibly be an issue with your query plan." | "The query plan is the issue: seq scan over 2.4M rows, 3.1s (`EXPLAIN ANALYZE`); missing index on `(tenant_id, created_at)`." | Modal doubt → assertion + plan output. |
| 7 | "It seems the pod is restarting because of the memory limit." | "`CrashLoopBackOff` on `api-7f9` is exit 137 (OOMKilled): limit 256Mi, peak 512Mi." | Hedged paraphrase → exact user term + exit code + both numbers. |
| 8 | "To answer your question: the mutex does not cover the read path." | "The mutex does not cover the read path: `get()` reads `m.items` without `m.mu` (`cache.go:41`); `-race` confirms." | Meta framing → echo window starts at the QFC. |
| 9 | "Sure — a few things you can try." | "The 502s come from `proxy_read_timeout 1s` (`nginx.conf:14`), below the upstream p99 of 2.4s. Raise it to 5s." | Acknowledgment + option dump → cause + single recommendation. |
| 10 | "Your question about flaky tests is a good one — flakiness usually comes from timing." | "Flaky test `TestRetryBudget` is timing-dependent: `time.Sleep(50ms)` at `retry_test.go:88` vs a 60ms scheduler tick; fails 1 in 40 runs." | Question restatement + generality → QFC + pinpoint + rate. |
| 11 | "Kubernetes scheduling failures usually have several root causes." | "Pod `web-3` is unschedulable: `0/6 nodes available — 3 Insufficient cpu, 3 node(s) had untolerated taint`." | Textbook generality → object identity + the system's own message. |
| 12 | "Hmm, interesting. Let me dig in." | "The 3 GB RSS growth is the `pending` map: 41k entries after 6h at 50 rps, never drained." | Filler → measurement + named structure. |

### Echo grammar (the three-word budget in practice)

```text
[ <QFC: 1–3 tokens> ]  [ <predicate / verdict> ]  [ <evidence pointer> ]  [ <action> ]

WH-question     "Postgres query timeouts  come from pool exhaustion  (p99 4.8s)   raise pool_size"
Yes/No          "The mutex does not      cover the read path         (cache.go:41)  take m.mu"
Imperative      "Docker builds fail at   npm ci / ENOENT             (.dockerignore) add !package-lock.json"
Error paste     "ENOENT package-lock.json  means the file is ignored  (.dockerignore:12) un-ignore it"
Wrong premise   "The mutex is not        the constraint              (pool 140/100)  fix the pool"
Multi-topic     "Postgres query timeouts (primary): pool exhaustion … Also CI red: flaky TestRetryBudget"
```

**Pre-flight pass (run in this order):** (1) extract the QFC; (2) strip every banned meta-token from words 1–3; (3) count tokens — abort if the QFC spills past 3; (4) confirm a verdict or polarity lands in sentence 1; (5) `head -1` the draft mentally — does line 1 stand alone?; (6) grep the query's head noun against the response — does it match the answer line rather than the question line?

---

## 3. Engineering Application Scenarios

### 3.1 Code Reviews

Code review happens in two registers: the **comment thread** (asynchronous, addressed by line number) and the **terminal chat** where a developer interrogates the review result. Both fail the same way — the response addresses the reviewer's process instead of the code entity.

* **Developer query (terminal chat):** `why did CI reject PR #412?`
* **Rejected (address displaced):** `Let's look into the CI failure. There are a few common reasons a pipeline can fail at this stage — flaky tests, dependency drift, or environment issues. In this case…`
* **Inversion:** `PR #412's CI failed on `TestGateCache` — flaky, not your diff: `TestGateCache` failed on 3 of the last 9 runs of `main` with `goroutine leak` (build 8841). Re-run or skip it; the other 11 jobs are green.`
* **What changed:** the entity (`PR #412's CI`) is at tokens 1–3, the verdict (`flaky, not your diff`) is in sentence 1, and the evidence (run counts, build id) follows. The developer's next action is decided in one line.

* **Replying inside a review thread.** The query is often implicit — you are answering a comment that begins `This looks wrong.` Inversion means echoing the *code entity*, not the reviewer's tone:
* **Rejected:** `Thanks — you're right, I should explain. There are a few reasons this looks odd at first glance…`
* **Inversion:** `updateCounter() is called from the HTTP handler (handler.go:64) and the cron worker (jobs.go:203), so the unlocked `+=` is unsafe: 200 lost increments per 10k under load. Switching to `atomic.Int64`.`

Rules for the review surface:

1. **Echo the artifact, not the sentiment.** Open with `PR #412's CI`, `updateCounter()`, `the migration in 9f3c1a2` — never with `you're right`, `good catch`, `thanks`.
2. **Answer the polarity before the reasoning** for yes/no review questions: `No — the lock does not cover that path` beats `It's worth thinking about whether the lock covers…`.
3. **Patch-review replies open with the fix, not the apology.** `Fixed in 9f3c1a2: the repro now passes 3/3` — the entity is the commit, the verdict is the state.
4. **Do not narrate the investigation.** `I checked the caller graph and…` becomes a parenthetical *after* the finding, or is deleted.

### 3.2 PR / Merge-Request Descriptions

A PR description is read in three ways: as a title in a list, as a summary before the diff, and as a chat answer when someone asks `what's this PR do?`. In the first two the echo rule is monotonic — reviewers scan a *list* looking for the subsystem they own.

* **Rejected title (address displaced):** `This PR does a few things to improve the auth flow and also fixes some tests`
* **Inversion title:** `Auth key rotation on a 24h schedule, not on restart`
* **Inversion body opener:** `Auth key rotation now runs on a 24h timer (auth/keys.go, +112 −38) instead of only at process start. Failure mode being closed: keys lived forever if the service never restarted.`

* **Terminal chat: `what's in #418?`**
* **Rejected:** `Sure! PR #418 is a fairly involved change. It touches a few files in the auth package and…`
* **Inversion:** `PR #418 rotates signing keys every 24h instead of on restart — auth/keys.go (+112 −38), 6 tests, no migration. Risk: auth issuance for all users; reversible via `KEY_ROTATE=off`.`

* **Terminal chat: `why is #418 still open?`**
* **Inversion:** `PR #418 is blocked on review, not on CI: 3 approvals required, 1 in; CI green on the tip commit (build 8842).`

Rules for the PR surface:

1. **First three words of a PR title = the changed entity + the change**, never `This PR…`, `Minor…`, `Refactor…`, `Various…`, `Fixes some…`.
2. **The answer to `what does this do?` echoes the subsystem, not the author's activity.** `Auth key rotation …` not `I refactored the way we handle …`.
3. **Changelog and release lines inherit the same head position.** `Auth: signing keys rotate every 24h (was: on restart)` — the entity is greppable in a release-note diff, which is the only way a user finds it.
4. **Status answers echo the PR id and the blocker.** `PR #418 is blocked on review` puts both the subject and the reason before any explanatory text.

### 3.3 Architecture RFCs and ADRs

Design documents are consumed as (a) a title in an index, (b) an answer in a chat thread during the review window, and (c) a permanent record. The chat case is where inversion pays: an RFC author answering `Postgres or DynamoDB for the events table?` with `Good question — both have trade-offs…` forces every participant to re-derive the subject of the thread.

* **Rejected (address displaced):** `There are important trade-offs here. Let's consider write throughput, operational cost, and consistency guarantees before deciding…`
* **Inversion (answer in terminal chat):** `The events table fits Postgres 16: 6k tps measured single-writer ceiling vs our 4k tps peak, 1 migration. DynamoDB loses on cost (on-demand ≈ 4× ) and gains nothing we need — we never exceed one region.`
* **Inversion (ADR record opener):** `Events table storage is Postgres 16. Constraint: single-writer throughput ≤ 8k tps measured; revisit if write load exceeds 6k tps. Rejected: DynamoDB (cost, no multi-region need).`

* **Terminal chat: `is the RFC for this approved?`**
* **Inversion:** `RFC-014 is Accepted, not implemented: 4/4 reviewers signed off 2026-03-14; owner @dana; implementation blocked on the schema migration ticket.`

Rules for the RFC/ADR surface:

1. **Echo the artifact id or the bounded decision object first.** `RFC-014 is Accepted`, `The events table fits Postgres 16` — never `So, to summarize the discussion…`.
2. **A decision answer contains the verdict in sentence 1.** `Is deferred` / `is accepted` / `is superseded`, with the blocking condition, before any rationale.
3. **Comparative answers name the two candidates in the echo window when both are in the question.** `Postgres vs DynamoDB resolves on write ceiling: …` — this preserves the user's own framing and makes the answer addressable in a long thread.
4. **Superseding a record echoes the record, not the revision.** `ADR-009 is superseded by ADR-014: the single-writer constraint it assumed (8k tps) no longer holds at 12k tps.` The user's entity appears at words 1–3 even when the news is that it changed.
5. **Never open an RFC answer with process.** `We discussed this at length in the review…`, `The comments on the doc raised…` are narration; the entity and verdict come first, the history second if at all.

---

## 4. Verification Checklist

- [ ] **Echo window holds the QFC in tokens 1–3** — count them. The user's head noun phrase (verbatim identifiers: `CrashLoopBackOff`, `pg_stat_activity`, `PR #418`) occupies position 1–3 in every response; if it spills to word 4+, rewrite the opening.
- [ ] **No meta-tokens in words 1–3** — `rg -i -n "^(sure|certainly|absolutely|of course|great question|good question|interesting|thanks for asking|let's|let me|i'd be happy|to answer your question|short answer|hmm|it seems|i think)\b"` returns nothing at the head of a response.
- [ ] **Verdict or polarity lands in sentence 1** — for yes/no queries the auxiliary answer (`does not`, `is set`, `are not`) appears at or before word 6; for causal queries the cause (`come from connection-pool exhaustion`) appears in sentence 1. No response reaches sentence 2 without having stated whether/how the user's entity is affected.
- [ ] **First-line survivability test passed** — mentally apply `head -1` (or paste only line 1 into a ticket): it must still name the topic, the verdict, and a pointer to evidence or a next action.
- [ ] **Greppability test passed** — `rg -i "<query head noun>" transcript.log` matches the *answer* line, not only the user's question line, because the answer names its own subject.
- [ ] **Acknowledgments and narration are relocated, not deleted** — any politeness or process note appears *after* the payload (`Want the patch? Two lines.`), and incorrect-premise queries name the corrected entity in the echo window (`The mutex isn't the constraint — the pool is`) rather than answering the wrong question politely.