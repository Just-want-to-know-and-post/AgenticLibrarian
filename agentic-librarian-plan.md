# The Agentic Librarian — Implementation Plan

*A tool-agnostic template for human editorial review of a knowledge-curation pipeline, and a buildable specification for a verified agentic librarian.*

Version 1.0 · Derived from Callum ("Waterloots"), *How to build a trusted LLM wiki*. Intended to be executed by a human engineer or a capable generative coding tool.

---

## 0. How to read this document

This plan has three jobs, stacked from most general to most concrete:

1. **The template (Sections 2–4).** A generic, tool-agnostic pattern for putting a *human editorial gate* in front of any knowledge-curation pipeline. This is the reusable heart of the project — it is not tied to any particular agent, editor, or storage format.
2. **The buildable specification (Sections 5–9).** A concrete design for the strongest form of that template — the *verified agentic librarian* — with data models, state machines, gate policies, and the split between deterministic code and semantic agent work.
3. **The integration and delivery plan (Sections 10–12).** How the curated knowledge base is exposed to downstream agents as a stable contract, how the build is phased, and how to know when each phase is done.

A reader who only wants the pattern can stop after Section 4. A reader who is building the system should read through Section 9. A reader wiring this into other agents needs Section 10.

Throughout, **Hermes + Obsidian appears only as one worked example** of an otherwise portable design. Wherever a concept has a tool-specific name in the source material, this plan names the *generic role* first and the tool-specific instance second.

---

## 1. Problem statement and design goals

### 1.1 The failure mode we are designing against

Giving an agent a durable, self-organizing memory is powerful, but the real hazard is not that the agent forgets something. It is that it **confidently retains something that was never true** — a hallucination, a piece of stale information, or a deliberately injected falsehood — and then *reasons on top of it*. Once a bad note enters a linked knowledge base, an agent can repeat it, connect it to other ideas, and build new conclusions on it. One corrupted note triggers a chain reaction that quietly taints the whole store. The probability of this rises as the surrounding information environment degrades.

The naive architecture makes this worse. When an agent is allowed to move a raw source *directly* into the trusted knowledge base — extracting concepts, writing pages, and interlinking them in a single automated pass — there is no point at which a human can intervene *before* the damage is woven into the graph. Even systems that are smart enough to *detect* a contradiction still often resolve or record it autonomously, meaning the questionable material has already been admitted and connected by the time anyone sees it.

### 1.2 The core insight

We cannot rely on an AI to reliably detect its own hallucinations, spot stale facts, or catch injected poison — **but we do not have to.** The fix is not smarter automated detection; it is an **authority boundary**: the agent may propose, synthesize, and interlink, but a *human decision* is what admits knowledge into the trusted store. The agent types the final words; human judgment is what lets them into the library.

### 1.3 Design goals

- **G1 — Trust by construction.** No content reaches the trusted store without passing an explicit editorial gate (with narrow, deliberately configured exceptions).
- **G2 — Separation of concerns.** *Semantic* work (reading, summarizing, cross-referencing, drafting) is done by an intelligence engine; *trust and admission* decisions are made by a human; *enforcement* of the process is done by deterministic code, not by trusting the agent to behave.
- **G3 — Portability.** The pattern must apply to any agent ("brain"), any editor/interface, and any markdown-or-similar store. The brain is replaceable.
- **G4 — Progressive adoption.** A user should be able to start with the simplest useful version and add rigor only as needed, without rebuilding.
- **G5 — Auditability and reversibility.** Every admitted change is traceable to a source and a human decision, and any run can be rolled back.
- **G6 — Consumability.** The trusted store must expose a stable, documented contract so *other* agents can build on it as ground truth.

### 1.4 Two distinctions to hold onto

Two conceptual separations run through the entire design:

**Where knowledge lives vs. how it earns trust.** These are orthogonal. The first is a *storage pipeline* (Section 2). The second is an *operating path* over that pipeline (Section 3). Conflating them is the most common way these systems go wrong.

**Curated knowledge vs. agent memory.** They serve different purposes and must not be merged. *Agent memory* (the automatic, high-volume record of how the user works and what happened in past sessions — tens of thousands of entries accrue on their own) answers "how do I work with this person?" *Curated knowledge* (this system's Gold store) answers "what have we deliberately chosen to trust about the world?" Memory is operational and cheap; the wiki is a deliberately small, human-vetted ground truth. Memory-generated pages may become *candidate sources* for the wiki, but they never auto-promote to truth.

---

## 2. The storage pipeline — a medallion architecture

Borrowed from data-lake practice, knowledge flows through three tiers. Each tier is a distinct location with distinct guarantees.

| Tier | Name | Contains | Trust guarantee | Who writes it |
|------|------|----------|-----------------|---------------|
| **Bronze** | Raw / ingestion | Unmodified source material — articles, transcripts, papers, notes, clippings | None. Assumed dirty. | Human (drops sources) or agent (fetches) |
| **Silver** | Review / proposals | Structured, cleaned, schema-conformant *proposals* derived from Bronze, awaiting a decision | Candidate only. Not yet trusted. | Intelligence engine, under policy |
| **Gold** | Trusted wiki | Approved, interlinked knowledge pages | Trusted ground truth | Promotion process, only after human approval |

The essential move is the **insertion of Silver between Bronze and Gold.** The failure mode in Section 1 is precisely what happens when a pipeline runs Bronze → Gold with no Silver. Silver is where cleaning, schema enforcement, contradiction surfacing, and — above all — *human editorial review* happen, before anything is admitted to Gold.

Around these three tiers sit two governance artifacts that are not themselves knowledge tiers:

- **The schema / operating manual** — a document (or set of documents) that governs how the system behaves: naming conventions, page templates, required metadata, link rules, and which workflows run how. The agent reads this as its rulebook. It is authored once and refined rarely.
- **The index and the log** — a machine-maintained catalog of every page and source (the index), and an append-only record of every action the system took (the log). Together they make the store navigable and auditable.

---

## 3. The trust ladder — three operating paths

Over the same storage pipeline, three operating paths deliver increasing trust. They are a **maturity model**: each is independently useful, and each higher rung adds one guarantee the rung below cannot make. The guiding advice is to **adopt the simplest path that meets the need and climb only when a real limitation forces it.**

### Path 1 — Intelligence (built-in curation skill)
The agent runs a capable, self-contained curation routine: it reads a raw source, applies the schema, extracts concepts, writes interlinked pages, hashes the source for change-detection, tags it, and even surfaces contradictions between sources. **What it adds:** intelligence and organization. **What it cannot make:** a trust guarantee — it runs Bronze → Gold and admits everything it processes, contradictions included. A hallucinated "opposing truth" can be woven through the whole graph before a human sees it, and there is no way to stop a specific change in advance.

### Path 2 — Trust (human-in-the-loop review wrapper)
A **wrapper** around Path 1 that intercepts its output: instead of writing compiled pages, it emits *proposals* into Silver and stops for human approval. It changes no Gold page until a human decides. **What it adds:** the editorial gate — the Silver layer, approval options, a feedback loop. **What it cannot make:** an *enforcement* guarantee. A skill/instruction is a *soft boundary*; a sufficiently eager agent can be asked to review and instead bypass the wrapper and run the underlying Path 1 routine directly. It can *request* good behavior; it cannot *prove* the exact approved change is what actually landed.

### Path 3 — Verification (the agentic librarian) — *the build target*
The review process is enforced by **deterministic tooling**, not by trusting the agent to follow instructions. A "librarian tool" — real code — performs a preflight check, hands only the *semantic* work to the agent, then validates the agent's output in code, routes it into a review inbox, applies the human decision, and checkpoints the result. **What it adds:** enforcement and verification — the process *cannot* be skipped, and what lands is provably what was approved and conforms to the schema. It also decouples the system from any one agent and creates a clean base for later expansion.

> **This plan targets Path 3** while documenting Paths 1 and 2 as the rungs beneath it, so a builder can ship an early rung and climb.

---

## 4. The generic Human Editorial Review template

This is the reusable core. It is the pattern that turns *any* Bronze → Gold pipeline into a trusted one, independent of tools. Sections 5+ are one rigorous realization of it.

### 4.1 Roles and the division of labor

| Role | Responsibility | Explicitly *not* responsible for |
|------|----------------|----------------------------------|
| **Curator (human)** | Chooses which sources enter Bronze; directs what analysis matters; makes every admission decision; supplies feedback | Writing pages, maintaining links, enforcing format |
| **Intelligence engine (agent / "the brain")** | Reads sources; summarizes; cross-references; drafts proposed pages and links; maintains internal consistency; flags contradictions | Deciding what is *true* or what gets *admitted* |
| **Control plane (deterministic code)** | Detects new/changed sources; enforces schema and link validity; routes proposals; applies approved changes exactly; checkpoints and logs | Any semantic judgment about meaning |
| **Reviewer surface (interface)** | Presents proposals, diffs, and rationale for human decisions; captures decisions and feedback | Making decisions on its own |

The one rule that makes the whole thing safe: **the agent proposes; the human admits; the code enforces.** No role may take another's job.

### 4.2 The unit of review: the *proposal*

Nothing is edited in place during review. The atom of the workflow is a **proposal** — a self-contained, inspectable description of a single intended change to Gold. A proposal carries:

- **Target** — the exact Gold page it would create or modify (path + title).
- **Operation** — `create`, `update`, `merge`, or `link`.
- **Rationale** — why the intelligence engine believes this belongs, in plain language.
- **Provenance** — the Bronze source(s) it derives from, by stable identifier/hash.
- **Proposed content** — the full page body or the precise diff, including all outgoing links (rendered so a human can read exactly what would land).
- **Connections** — which existing Gold pages it would link to or alter.
- **Open questions** — anything the engine is unsure about and wants the human to resolve.
- **Human feedback slot** — a dedicated field the curator writes into to request a revision.
- **Status** — see the state machine below.
- **Revision number** — proposals are versioned; feedback produces a new revision, not an overwrite.

Batches of proposals are grouped per source/run so a curator reviews a coherent set, and each source is captured **without changing any compiled page** until approval.

### 4.3 The proposal state machine

```
                 ┌─────────────┐
   new source →  │ needs_review│  ← (revision N)
                 └──────┬──────┘
        ┌───────────────┼───────────────┬───────────────┐
   approve           revise           defer            reject
        │               │               │               │
        ▼               ▼               ▼               ▼
   ┌─────────┐   ┌──────────────┐  ┌─────────┐    ┌──────────┐
   │approved │   │needs_review  │  │deferred │    │ rejected │
   │(pending │   │(revision N+1)│  │(paused) │    │(archived)│
   │ apply)  │   └──────────────┘  └─────────┘    └──────────┘
        │            ↑ engine re-drafts from feedback
        ▼
   ┌─────────┐        The apply step is SEPARATE and
   │ applied │  ←──   runs only after an explicit
   │ (in     │        "proceed" — decision and
   │  Gold)  │        application are two phases.
   └─────────┘
```

**Decision options presented to the curator** (individually or in bulk): **approve**, **reject**, **revise** (with feedback), **defer**, or **skip for now** (leave the whole batch for later). Bulk actions (`approve all`, `reject all`) exist for routine batches.

**The two-phase rule (critical).** Recording a decision and mutating Gold are *separate steps*. When the curator approves, the system **records the decision and stops** — it has changed nothing. Only an explicit *"proceed with the approved revisions"* triggers application to Gold. This makes the decision reviewable before it is irreversible and gives the curator a last look.

**The feedback loop.** A `revise` decision does not discard the proposal; the curator's feedback (written into the proposal's feedback slot or as inline edits on the draft) is fed back to the intelligence engine, which produces **revision N+1** for another round. Review is iterative, not one-shot.

### 4.4 Dual-surface principle

The curator must be able to act on proposals from **either** the raw store (editing proposal files directly in the editor) **or** a dedicated review interface — and both must be *views of the same underlying decisions*, never separate inboxes that can drift. A visual "decision studio" makes comparison and approval easier; the file view keeps everything inside the vault. Whichever the curator uses, the next run reads the same state.

### 4.5 Contradiction handling

When new material conflicts with existing knowledge, the system must **preserve the contradiction rather than silently resolve it.** The pattern: create a *comparison* artifact stating both positions, the precise question in dispute, and their provenance; mark every affected page `contested = true`; and surface it for human clarification (is this a changed position, a deliberate counter-argument, or a test case?). In a trusted pipeline this happens *inside Silver*, as a proposal — so the contested material waits for a human before it can touch Gold, closing the gap where a merely-detected contradiction is nonetheless admitted.

### 4.6 Provenance and change detection

Every Bronze source is fingerprinted (a content **hash**) and marked `processed` once handled. If a source's content changes, its hash changes, so the system knows to re-ingest and re-review *only what changed*. Every Gold page traces to the source(s) and the human decision that admitted it. This is what makes the store auditable and lets the system do incremental, idempotent runs.

### 4.7 The template as a checklist

A pipeline implements this template if and only if it has: (1) three distinct storage tiers with Silver between Bronze and Gold; (2) proposals as the review atom, carrying provenance and rationale; (3) the decision state machine with a *separate* apply phase; (4) an iterative feedback loop; (5) dual, non-diverging review surfaces; (6) non-destructive contradiction preservation; (7) provenance + change detection; and (8) a clear role split where the agent proposes, the human admits, and (at Path 3) code enforces.

---

## 5. Verified agentic librarian — architecture

Sections 5–9 specify the Path 3 realization of the template.

### 5.1 The deterministic/semantic split

The defining property of Path 3 is that **the entire workflow is deterministic code except for the one step that genuinely needs intelligence.** The agent is never trusted to *run the process*; it is invoked as a bounded function to do semantic drafting, and its output is validated by code before it is allowed to proceed.

```
  ┌──────────────────────────── LIBRARIAN CONTROL PLANE (code) ───────────────────────────┐
  │                                                                                        │
  │  ①TRIGGER ─▶ ②PREFLIGHT ─▶ ③SEMANTIC ─▶ ④VALIDATE ─▶ ⑤ROUTE ─▶ ⑥DECISION ─▶ ⑦APPLY ─▶ ⑧CHECKPOINT
  │  (manual/    (code:        (AGENT:       (code:       (code:     (HUMAN:     (code:      (code:
  │   auto)      diff sources  draft         schema,      routine    approve/    exact       git
  │              vs store;     proposals,    links,       vs         revise/     approved    commit;
  │              stop if       summaries,    citations,   complex;   defer/      change      log;
  │              nothing new)  links)        format)      auto-      reject)     only)       index)
  │                            └─ only step   └─ reject                promote?)                        │
  │                               that is        back to                                               │
  │                               non-code       ③ on fail                                              │
  └────────────────────────────────────────────────────────────────────────────────────────┘
```

Only step ③ is semantic. Everything else — detecting what changed, enforcing the schema, checking that every link resolves and every claim cites a source, deciding routine-vs-complex routing, applying exactly the approved change, and committing a checkpoint — is code. This is what upgrades Path 2's *soft* boundary into a *hard* one: the agent literally cannot write to Gold, because only the apply step (code) can, and it applies only what a human approved.

### 5.2 Component inventory

1. **Librarian tool (CLI/library).** The control plane. A single bounded tool exposing sub-commands (preflight, ingest/propose, validate, apply, status, checkpoint). The agent invokes *these commands*; it does not free-form manipulate the store. This is what keeps the agent "on rails."
2. **Policy / settings.** Human-editable configuration (see 6.3) that governs routing, auto-promotion, which source types go where, memory on/off, and automation cadence.
3. **Schema / operating manual.** Page templates, required metadata, naming and linking rules — the contract the validator enforces and the agent drafts against.
4. **Intelligence-engine adapter.** A thin interface so the "brain" is swappable (hosted frontier agent or local model). The control plane calls the adapter for step ③ only.
5. **Store (Bronze/Silver/Gold + index + log).** The medallion tiers plus catalog and audit trail, as plain files.
6. **Review surface(s).** The in-editor view and a visual decision studio, both bound to the same Silver state.
7. **Version control.** A checkpoint per run for rollback.
8. **Dashboard.** A generated status view (coverage %, counts per tier, recently updated, contested items).

### 5.3 Why a *tool* and not just a skill

A skill/instruction can be ignored or misapplied by the agent; a tool is executed. By having the schill/skill *tell the agent to call specific tools*, the agent is bounded inside the librarian's operations rather than defaulting to unconstrained behavior. This is the single change that makes the difference between "please review" and "review is enforced," and it is also what makes later expansion safe: new capabilities plug into typed tool calls and code contracts rather than a fragile web of skills that may or may not cooperate.

---

## 6. Data model and artifacts

All artifacts are plain text (markdown with structured frontmatter) so the store is human-readable, diffable, portable, and editor-agnostic. Formats below are illustrative, not mandatory — the *fields* are the contract.

### 6.1 Bronze — a raw source

```yaml
---
id: src_2026_0830_a1b2          # stable id
type: transcript                 # article | paper | note | clipping | transcript | memory_page
source_url: https://…            # if applicable
ingested_at: 2026-08-30T18:04:00Z
content_hash: sha256:…           # fingerprint for change detection
processed: false                 # flipped true after handling
tags: []                         # applied by the engine on ingest
---
<verbatim source content>
```

### 6.2 Silver — a proposal

```yaml
---
id: prop_…                       # stable id
revision: 1
status: needs_review             # needs_review | approved | applied | deferred | rejected
target: concepts/agentic-memory.md
operation: create                # create | update | merge | link
provenance: [src_2026_0830_a1b2] # one or more Bronze ids
classification: routine          # routine | complex   (set by validator/policy)
connections: [[LLM Wiki]], [[Retrieval-Augmented Generation]]
open_questions:
  - Does this supersede the prior position on persistence, or contrast it?
human_feedback: ""               # curator writes revision requests here
decided_by: null                 # human id + timestamp once decided
---
## Proposed content
<full page body or precise diff, links rendered>
```

### 6.3 Policy / settings

A human-editable table drives system behavior without code changes. Representative keys:

| Setting | Effect |
|---------|--------|
| `default_route` | Where new proposals land: `review_inbox` (Silver) or, per type, `direct_to_gold` |
| `auto_promote_types` | Source/proposal types allowed to skip the gate (deliberately narrow) |
| `routine_criteria` | Rules by which the validator marks a proposal `routine` vs `complex` |
| `memory_provider` | Optional link to an agent-memory store as a *candidate-source* feed (default off) |
| `intelligence_engine` | Which brain adapter to use |
| `automation` | Cadence for unattended runs; and "stop before calling a model if nothing changed" |
| `git_checkpoints` | On/off for per-run commits |

### 6.4 Index, log, and dashboard

- **Index** — machine-maintained catalog of every source and page with status; updated every run.
- **Log** — append-only record of every action (ingested, proposed, approved, applied, committed) for audit.
- **Dashboard** — generated view: counts per tier, **source→wiki coverage %**, recently updated pages, and a contested-items list. Purely a read model over the store.

### 6.5 Store layout (illustrative)

```
vault/
  schema/              # operating manual, page templates, link rules
  raw/                 # BRONZE — by type: articles/ papers/ transcripts/ notes/
  review/              # SILVER — proposals + comparison artifacts
  wiki/                # GOLD — approved interlinked pages: concepts/ …
  index.md             # catalog
  log.md               # audit trail
  home.md              # dashboard
  .git/                # checkpoints
```

---

## 7. Gate policies — routing and triage

Not every proposal deserves the same scrutiny, and the system must let the curator tune this without touching code.

- **Routine vs complex triage.** During validation the control plane classifies each proposal. *Routine* = source-grounded, schema-clean, low-connectivity, non-contested — safe to approve in bulk. *Complex* = introduces new top-level concepts, alters many existing pages, or is contested — demands individual review. The curator is told the counts ("validated five source-grounded proposals, all routine — approve all, or review individually?").
- **Auto-promotion (use sparingly).** Policy may allow specific *types* to bypass the gate straight to Gold (e.g., a trusted internal note type). This is a deliberate, narrow exception to G1, configured by the human, never a default.
- **Contested override.** Anything marked `contested = true` is forced to `complex` and can never auto-promote, regardless of type.
- **Idempotence.** Preflight compares source hashes to the index; if nothing changed, the run stops *before* invoking the engine (saving cost and avoiding spurious proposals). Re-running is always safe.

---

## 8. Versioning, rollback, and safety

- **Checkpoint per run.** Initialize version control in the store. Every librarian run that mutates the vault commits automatically, creating a save point. A mistaken ingest, an accidental file, or a bad apply is recovered by rolling back — the store is never a one-way door.
- **Non-destructive review.** Because review happens in Silver and Gold is only ever touched by the code apply-step acting on an approved proposal, a bad proposal costs nothing but a rejection.
- **Skill/tool supply-chain caution.** Any externally supplied skill or tool must be **read and reviewed before installation** — a malicious one can carry prompt injection. Run the intelligence engine and its file/terminal access **inside a sandbox/container** with scoped read/write permissions, so a hostile source or skill has a limited blast radius. (This also matters because Bronze is assumed dirty by design.)
- **Least privilege.** The engine adapter gets only the vault paths it needs; the apply-step is the only component with Gold write access.

---

## 9. Automation and the replaceable brain

- **Unattended runs.** The librarian can run on a schedule. The loop: on trigger, preflight; **if nothing changed, stop before calling a model or memory provider**; else produce proposals under the review policy exactly as in an interactive run. Automation changes *when* the librarian checks — never the *quality bar*, because the same gate and validation apply.
- **The brain is a swappable part.** Because everything is deterministic except step ③, the intelligence engine is decoupled and replaceable — a hosted frontier agent or a local model. Caveat: a local model must be good enough at long context, structured output, exact filenames, and citations; sub-par models will fail validation. Test before trusting.
- **Memory as candidate sources, not truth.** If an agent-memory provider is connected, its generated pages enter as Bronze *candidates* and go through the same gate. They never become Gold automatically — preserving the curated/operational distinction from §1.4.
- **Optional knowledge expansions.** Once the deterministic core exists, further layers can operate *only on Gold* (e.g., decomposing approved concepts into atomic notes and recombining them for learning/teaching). These are additive and out of scope for the core build, but the tool-based core is what makes them safe to add.

---

## 10. Downstream consumption contract (for other agents)

> **Integration hook.** The Librarian exists to build knowledge bases *for other agents* whose capabilities are described in two separate work sessions (`cse_01L5wLJz…` and `cse_015mttV1…`), which are **not readable from this session.** Per the agreed approach, this plan defines a clean, tool-agnostic **consumption contract** so any downstream agent can build on the Gold store as ground truth, and marks the exact place to bind those two agents' specifics once their capability descriptions are provided. **→ See TODO-INTEGRATION at the end of this section.**

A downstream agent should never read Bronze or Silver. It consumes **Gold only**, through a stable contract:

- **Trust boundary.** Only `wiki/` (Gold) is ground truth. Bronze is dirty; Silver is unapproved. Consumers must not read from them.
- **Page contract.** Every Gold page exposes: a stable id/title, typed frontmatter (concept type, tags, `contested` flag, provenance source ids, last-approved timestamp), a body, and typed outgoing links. Consumers can rely on these fields existing.
- **Contested signal.** A consumer must check `contested`; contested pages are usable but must be treated as disputed, not settled.
- **Provenance access.** Each page's source ids let a consumer trace a claim back to raw evidence and to the human decision that admitted it.
- **Query surface.** Expose the store to consumers by (at minimum) the index (catalog + status) and the link graph. Optionally provide a read-only retrieval endpoint over Gold. Consumers get *retrieval over vetted knowledge*, not raw-document RAG.
- **Change feed.** The log/index lets a consumer detect what Gold pages changed since it last synced, so downstream caches stay fresh without re-reading everything.
- **Read-only.** Downstream agents never write to the store. New knowledge they generate re-enters as Bronze candidates and goes through the gate like anything else.

**TODO-INTEGRATION (to complete once the two sessions are available):**
1. Enumerate each downstream agent's required knowledge *domains* → map to Gold page types/schema.
2. Determine each agent's access mode (bulk read, retrieval endpoint, or change-feed subscription).
3. Decide whether any agent needs a *scoped subset* of Gold (namespacing/tags for access control).
4. Confirm whether any agent's outputs should feed back as Bronze candidates, and under what policy.
5. Specify freshness/sync expectations per agent.

---

## 11. Build roadmap

Phased so each phase is independently shippable and maps to a rung of the trust ladder.

**Phase 0 — Foundations.** Stand up the store layout (§6.5), author the schema/operating manual, and choose the intelligence-engine adapter. *Exit:* an empty, initialized store with schema, index, and log; the brain can read and write the vault under scoped permissions.

**Phase 1 — Path 1 (Intelligence).** Implement or adopt automated Bronze→Gold curation: schema application, concept extraction, interlinking, source hashing/`processed`, tagging, and contradiction surfacing. *Exit:* dropping a source produces interlinked Gold pages with provenance; contradictions produce comparison artifacts. *(Known gap: no human gate — this is expected and motivates Phase 2.)*

**Phase 2 — Path 2 (Trust).** Wrap Phase 1 so output becomes **proposals in Silver** instead of Gold writes. Implement the proposal artifact (§6.2), the state machine and two-phase decide/apply (§4.3), the feedback loop, and a basic in-editor review view. *Exit:* no Gold page changes without an explicit approve-then-proceed; revisions iterate; nothing is admitted un-reviewed.

**Phase 3 — Path 3 (Verification) — the target.** Replace the skill-as-process with the **librarian tool** (§5.2): deterministic preflight, validate (schema/link/citation), route (routine/complex, auto-promote policy), apply-exact, and per-run git checkpoints. Add the settings/policy table and the dashboard. Bound the agent to tool calls. *Exit:* the process cannot be bypassed; applied Gold provably matches the approved proposal and the schema; every run is checkpointed and logged; dashboard shows 100%-traceable coverage.

**Phase 4 — Surfaces & automation.** Add the visual decision studio (bound to the same Silver state), scheduled unattended runs with the "stop if nothing changed" guard, and optional memory-provider candidate feed. *Exit:* curator can review from either surface interchangeably; automation runs safely without lowering the quality bar.

**Phase 5 — Downstream integration.** Implement the consumption contract (§10) and complete **TODO-INTEGRATION** against the two referenced sessions. *Exit:* at least one downstream agent consumes Gold as ground truth through the defined contract.

**Phase 6 — Optional expansions.** Gold-only knowledge patterns and multi-agent orchestration (specialized librarian/research/memory workers cooperating). Additive; only after the deterministic core is solid.

---

## 12. Acceptance criteria and verification

Each is a testable gate for "done," phrased so a human or an automated test can check it.

**Trust (G1).**
- Injecting a known-false source and running the pipeline produces a *proposal in Silver*, **never** a Gold change, until a human approves. *(This is the exact failure the whole system targets — test it explicitly.)*
- No path writes to Gold except the code apply-step acting on a proposal in `approved` status.

**Separation & enforcement (G2).**
- Asking the agent to "just update the wiki directly" cannot mutate Gold — the tool boundary prevents it (Path 3).
- Applied Gold content is byte-for-byte the approved proposal content (diff the two).

**Editorial workflow (§4).**
- The proposal state machine supports approve/reject/revise/defer/skip, individually and in bulk.
- Decision and application are two distinct steps; approving alone changes nothing.
- `revise` + feedback yields revision N+1 without losing history.
- The in-editor view and the decision studio show the *same* pending decisions (change one, the other reflects it).

**Integrity (§4.5–4.6, §8).**
- A contradiction yields a comparison artifact and `contested=true` on affected pages, held in Silver.
- Every Gold page traces to source id(s) and a human decision.
- Changing a source's content changes its hash and triggers re-review of only that item.
- Rolling back a run restores the prior vault state exactly.

**Portability (G3).**
- Swapping the intelligence-engine adapter (e.g., hosted → local) runs the full pipeline unchanged; validation catches a model too weak to meet the schema/citation contract.

**Idempotence & automation (§7, §9).**
- Re-running with no new sources makes no changes and does not invoke the model.

**Consumability (G6, §10).**
- A downstream consumer can read Gold, honor the `contested` flag, trace provenance, and detect changes since last sync — using only the documented contract, without reading Bronze or Silver.

**Suggested verification method.** Build a small fixture set — one clean source, one hallucinated/false source, one source that directly contradicts an existing page, and one unchanged re-run — and assert the expected tier transitions, flags, gate stops, and rollback behavior for each. This exercises every guarantee above end-to-end.

---

## Appendix A — Generic ↔ worked-example glossary

| Generic role in this plan | Worked example in the source (Hermes + Obsidian) |
|---|---|
| Store / vault | Obsidian vault |
| Intelligence engine ("brain") | Hermes agent (swappable: Claude Code, Codex, Gemini, local via Ollama) |
| Built-in curation skill (Path 1) | Hermes "LLM Wiki" skill |
| Review wrapper (Path 2) | "LLM Wiki Review" companion skill |
| Librarian tool / control plane (Path 3) | The "verified agentic librarian" kit + `librarian` tool script |
| Review surface | Obsidian review folder + "Decision Studio" |
| Dashboard | Obsidian "LLM Wiki Home" (via Obsidian Bases) |
| Schema / operating manual | The generated `schema` file |
| Checkpointing | Git initialized in the vault |
| Curated knowledge vs. agent memory | LLM wiki vs. Hindsight/Nemosity memory (~30k auto-memories) |

## Appendix B — One-line summary of the philosophy

*Start with intelligence, add trust, then verify it — and expand only when a real need appears.* The agent proposes, the human admits, and the code enforces; the wiki is the deliberately small, human-vetted ground truth that everything else is allowed to build on.
