#### four properties



1\. Next-token Prediction (How Claude "writes"):

generate token 1 at a time.

good at pattern, not great at precise facts

fix - use tools like web search, database lookups



2\. Knowledge (What Claude actually "knows")

only knows what was in its training data.

rare, changing fast topics it might be wrong.

fix - use web search, rag



3\. Working Memory (Claude's "context window")

context window.

two problems - Too much sent at once → Claude just rejects it

&#x09;     - It fits, but runs out mid-response → Claude starts answering, but gets cut off before finishing.

fix - small chunks, most important stuff first, summarize as you go



4\. Steerability (How well Claude follows instructions)

good at - short clear instruction

bad at - Vague requests

Fix - for precise math or logic, let Claude use a code execution tool instead of "thinking" the math out in plain text.





#### Three layers, three distinct decisions



1.Entry Point

How does a person/system talk to Claude?

Chosen for the user and the task

Claude.ai, Claude Code, a custom app



2.Build-time Interface

How does an engineer write the connection code?

Chosen by the engineering team

API, SDKs, MCP, Agent SDK



3.Delivery Route

Whose servers does it run on?

Chosen based on cloud contracts/compliance

Anthropic directly, AWS Bedrock, GCP Vertex AI, Microsoft Foundry







#### Seven primitives, seven jobs



1.tools(Act) - to do something or fetch information

2.MCP(connect) - expose a set of tools

3.subagents(Isolate / Parallelize)

4.hooks(Guarantee) - to enforce a rule that Claude cannot skip or forget

5.skills(Package a procedure) - A reusable, saved set of instructions

6.Agent Teams(Coordinate peers) - Multiple agents working together, each owning part of a larger goal.

7.dynamic workflow(Compose at runtime) - figures out the steps on the fly







#### Claude vs. Systems vs. Humans



The 3 Owners:

Claude - Understanding language, summarizing, planning, calling tools

Existing systems - databases, rule engines, policy tables

Humans - Judgment calls, exceptions, approvals



3 Questions to Ask:

Reversibility — If Claude gets this wrong, can we undo it easily?

Stakes — If Claude gets this wrong, how bad is the damage?

Accountability — If this goes wrong, who has to answer for it?



If the answers are "hard to undo," "very costly," and "a human will get blamed" — that task probably shouldn't be fully handed to Claude alone.







#### Three patterns for structuring Claude's involvement



2 axes:

predictability - how predictable the path through the work is

model autonomy - How much freedom are you giving Claude



1.Augmented LLM

One clean, bounded task.

You send one request. It use tool or look something, but it's one pass, one job.

High predictability, low autonomy.



2.Workflow

A fixed path, with some judgment inside each step

Step 1 (Claude reads the claim) → Step 2 (system checks priority rule) → Step 3 (system looks up policy) → Step 4 (Claude drafts email, human approves, system sends).

Predictable.



3.Agent

Claude decides its own path, step by step.

High autonomy, low predictability.

The model owns the trajectory.



Sub-patterns within workflows:

Chaining - Steps happen one after another

Routing - A classifier decides which path to take

Parallelization - Multiple things happen at the same time, independently

Evaluator-optimizer - Draft it, grade it, redo it if needed



five factors in sequence: (low/medium/high risk)

&#x09;

|Factor|Augmented LLM|Workflow|Agent|
|-|-|-|-|
|Predictability|low|low|high|
|Error cost|medium|low|high|
|Observability|medium|low|high|
|Latency budget|low|medium|high|
|Cost|low|medium|high|





Before You Fine-Tune: Try This Order First:

If Claude isn't behaving reliably, don't jump to fine-tuning. Instead, work through these steps in order:

1.Fix the prompt first

2.Add tools or retrieval

3.Upgrade to a stronger pattern

4.Only then, consider fine-tuning







#### Multi-agent systems and orchestration



2 Roles:

The Orchestrator - Breaks the big job into smaller pieces

&#x09;         - Decides who (which subagent) handles which piece

&#x20;      		 - Never does the actual sub-task work itself

The Subagents - gets one specific, scoped piece of the work

&#x09;      - Works in its own separate context



fan-out:

Split one big job into many independent pieces, hand each piece to its own subagent, run them at the same time (in parallel), then combine everything.



3 Things You MUST Design (Not Just Assume):

How the work gets split up?

How each subagent's answer is formatted?

How the orchestrator handles conflicts or missing pieces?



Where Failures Happen — and Whether You Can Recover:

|Failure|Can you recover?|What to do about it|
|-|-|-|
|One subagent returns a broken or empty result|yes|Retry that one unit, send it elsewhere, or just skip it and note the gap. The rest of the work keeps going fine.|
|Two subagents disagree with each other|yes|Give the orchestrator a clear rule for resolving conflicts, or send it to a human to decide.|
|The orchestrator itself loses track of the goal or messes up combining everything|no|The whole run can fail, and all the good work done by subagents so far may be wasted. Protect this carefully — save progress checkpoints so a failed run can pick back up instead of starting over.|
|You can't trace what happened across the orchestrator and all its subagents|Cross-cutting issue|Give every run a shared "trace ID" so you can piece together the full story after the fact.|





Human-in-the-loop checkpoint patterns for agent workflows:

Before a subagent takes an action that's irreversible or high-stakes, pause and let a human review it first, rather than letting the agent act completely on its own.

High-stakes/irreversible action → always gate it with human review

Lower-stakes actions → just spot-check some of them occasionally, don't review every single one







#### Reference Architectures



Reference architectures are references, not blueprints to adhere to.



The 5 Common Reference Architectures:

||Good Design|Common Mistake|
|-|-|-|
|Agent|Model picks its own tool calls toward a goal; kept in check with limited tools + turn limit. Use for unscriptable work (codebase exploration, research, complex triage).|Unbounded autonomy — no turn limit, no human review, no way to check if the goal was met.|
|RAG|Stable knowledge (docs, manuals) is indexed; relevant chunks retrieved as context.|Used for live/changing data (order status, inventory) — the index is just a stale snapshot.|
|Document Pipeline → Evaluator-Optimizer|Extract fields from documents (claims, invoices) against a schema, validate, route exceptions. Extra check-loop for unreliable edge cases.|No exception path — bad extractions flow through untouched, with no human catching them|
|Ticket Triage → Routing|Classify the request, then route: docs → retrieval, live data → direct API call, high-stakes → human.|Using retrieval for live data instead of an API call; no human escalation path.|
|Coding Agent|Two phases: agentic investigation (unscriptable) + deterministic edit/test/review steps|Letting it commit code with no human review; no regression tracking.|



When retrieval got reached for instead of a tool call:

\#1 most common mistake - using RAG (retrieval) for live, changing data instead of calling the actual system directly.



Retrieval (RAG) — pull relevant text chunks from a pre-built index

Tool call — directly ask a live system for its current, real-time value



What broke and why:

|The Category Error|Retrieval is for knowledge. It's wrong for live state. The order-status question was fundamentally asked of the wrong tool.|
|-|-|
|A Data-Architecture Failure, Not a Retrieval Failure|Retrieval did its job correctly. The real mistake happened earlier: someone stored live, changing order data as static text snapshots in the first place.|
|Similarity Is Not Truth|A high similarity score just means the text sounds close to the question — not that it's currently accurate.|
|The Fix Is a Tool Call|Just call the live order-status API directly.|





#### 

#### Chunking and indexing



Chunking: How You Break Up the Documents

|Approach|How it works|Best for|
|-|-|-|
|Fixed-size|Cut into equal-sized pieces (with some overlap)|Plain, unstructured text with no clear natural breaks — simplest option|
|Semantic|Cut where the meaning shifts (topic changes, idea boundaries)|Prose where each chunk needs to make sense on its own|
|Hierarchical|Keep the document's actual structure (sections/subsections)|Structured docs like contracts, manuals, policies — where section context matters|





Indexing: How You Make Chunks Searchable

|Strategy|Matches on|Best for|
|-|-|-|
|Dense (embeddings)|Semantic similarity, meaning, not words.|Paraphrased or loosely-worded questions|
|Sparse (keyword/BM25)|Exact terms, identifiers, codes, names.|Queries with specific tokens — part numbers, error codes, statute citations|
|Hybrid|Both, with the results combined.|Mixed real-world queries — the common production choice|



The 3-Way Trade-off:

Quality — does the right chunk actually come back?

Latency — how much time does retrieval add?

Maintenance — how much upkeep as the corpus grows/changes?



The pattern: Smaller chunks + hybrid indexing = better quality, but slower and more to maintain. Larger chunks + dense-only = faster and simpler, but misses exact-match queries.







#### Model, Context Window \& Context Strategy



4 Terms People Mix Up (Don't!):

Context window - The model's active "attention space" right now

Retrieval - Fetched knowledge added into the context window

Persistent app state - 	Live data (order status, balances) your system owns — model needs a tool call to see it.

summaries/memory - Continuity across turns — the model has no native memory; your app has to store and re-feed it.



1\. Model Selection: Start with Sonnet

Default: Sonnet.

Move up to Opus only when an eval proves Sonnet isn't good enough.

Move down to Haiku only when an eval proves the quality drop is acceptable.

Never switch models based on gut feeling — always based on measured data.



2\. Context Window Sizing: "The Working-Memory Cliff"

Everything inside the window = usable.

Measured in tokens, not characters.

Never budget the full window. Budget for: biggest realistic conversation + retrieved context + system prompt + scratch space + growth margin.



3\. Context Strategy: Progressive vs. Monolithic

Monolithic - Dump everything into one prompt (full doc, full history)

&#x09;   - Fills up silently over many turns — hits the "cliff" in production.

Progressive - Only carry forward what's needed for the next step

&#x09;    - Most production systems should use this

Retrieval - Fetch relevant chunks just-in-time

&#x09;  - Good for large corpora too big to preload

Compaction - Periodically summarize/compress old context



Worked Example: Coding Agent

Load initial task + files	    - Monolithic (small, stable, cached)

Ongoing tool calls (read/test/edit) - Progressive (latest state)

Needs a file it didn't load	    - Retrieval (search codebase, pull on demand)

Context filling up after many turns - Compaction (summarize what was learned)



Extended Thinking: When to Turn It On:

Extended thinking is an optional mode where, before giving you the final answer, Claude first works through the problem in a separate "scratchpad" of reasoning — like showing its work — and then gives you the polished final answer.

Extended thinking = an extra, billed step. You pay for the tokens used in that "scratchpad" reasoning, AND it makes the response take longer (added latency).



Always Gate Model Changes with an Eval:

Swapping models = a real behavior change, treat it like a code release. You need,

A test set with known-good answers

A grading method (rubric or code-based check)

A rollback threshold set in advance — before you see the results







#### System Prompts, Templates \& Guardrails



1.System-prompt architecture for enterprise reuse

A one-off chat prompt and a prompt that hundreds of requests a day depend on are completely different things.

A production-grade system prompt needs actual structure:

Role and scope — what is Claude here to do?

Constraints — what must it always do / never do?

Output contract — the exact shape the response must take



2.Templates: consistency and safety, enforced

What a template is: A system prompt with fixed scaffolding (guardrails, structure) + variable slots (the parts that change per request).

Think of a template like a fill-in-the-blank contract form.



3.Description: the 4D competency applied to prompt design

precisely specifying what you want.

Scope	    - What's in-bounds vs. out-of-bounds?

Format	    - Exactly what shape must the output take?

Constraints - What rules must never be broken?

Underspecification: Whatever the prompt doesn't explicitly say, the model fills in with its own guess



4.Diagnosing underspecification gaps

"Does the prompt actually state this — or am I just hoping the model figures it out?"

The fix: Make the implicit explicit — restate the goal, name the exact format, spell out the constraints clearly. Don't assume Claude will "just know" what you meant.







#### Caching, Modular Prompts \& Skills



Caching:

your prompt stays the same across many requests, the system can "remember" it instead of re-processing it every time.

fixed instructions first, unique/changing content last.



Caching mechanics:

1.Cache breakpoints — where exactly the "stable" part ends and the "dynamic" part begins

2.Content ordering — fixed stuff first, changing stuff last (like the story above)

3.TTL selection — how long a cached prefix stays valid before it needs refreshing (TTL = "time to live")

4.When the write overhead isn't worth it — sometimes writing to cache costs more than it saves



Modular Prompt Library vs. Skill:

||Prompt Library|**Skill**|
|-|-|-|
|What it is|A shared collection of prompt fragments/templates engineers assemble themselves in code|A formal, versioned, self-contained package (SKILL.md + optional scripts) — the whole procedure travels as one unit|
|Best for|A prompt that gets tweaked/assembled differently each time|A stable procedure that runs the exact same way every time|
|Who uses it|Shared within one team/codebase|Distributed across multiple teams/products|
|Governance|Lightweight — engineers just own their pieces|Formal — versioning, approval, rollback built in|



How to Decide: Library or Skill?

1.Repeatability: Does this get tweaked per use, or run the exact same way every time?

→ Tweaked = library. Same every time = Skill.

2.Distribution: Is this just for my team, or does it need to be used the same way across multiple teams/products?

→ One team = library. Multiple teams = Skill.

3.Governance: Do we need formal versioning, approval, and rollback?

→ Light oversight = library. Formal control needed = Skill.







#### Entry Points, Routes \& Governance



1\. Entry Point — "How does a person/system reach Claude?"

Claude.ai       - Knowledge workers using Claude as a chat/thinking partner

Claude Code     - Engineers doing real development

Claude Cowork   - Non-developers automating files/tasks on their own computer

Claude in Chrome- Browser-based tasks — navigating and acting on web pages

Claude for Excel- Analysts/finance roles working mainly in spreadsheets



2\. Build-time Interface — "How does an engineer connect code to Claude?"

Direct API   - You need full control or an SDK hasn't exposed a feature yet

SDKs	     - Default choice — same as API but with less boilerplate

MCP	     - The same tools need to be reachable from multiple Claude clients (Claude.ai, Claude Code, your app, etc.)

Agent SDK    - You need a multi-turn agent loop (like Claude Code's) running inside your own product



3\. Delivery Route — "Whose servers does it run on?"

Anthropic first-party - No cloud commitment, want newest features immediately

AWS Bedrock           - Partner already has an AWS enterprise agreement

GCP Vertex AI	      - Partner's ML stack already lives on GCP

Microsoft Foundry     - Partner uses Entra ID / Azure enterprise agreement



Claude Code's Extra Layer: Shaping vs. Governing:

|Type|Layers|Purpose|
|-|-|-|
|Shaping (what it knows/does)|CLAUDE.md, Skills, Subagents, MCP servers|Gives the agent context and capabilities|
|Governing (what it's allowed to touch)|Hooks, Permission boundaries, Sandboxing|Controls/limits what actions can actually happen|



Regulated-Industry Constraints Come FIRST:

|Constraint|Who it applies to|What survives review|Why|
|-|-|-|-|
|Attorney-client privilege|Law firms|API/SDK behind the firm's own app, with full audit trail|Firm needs to prove, end-to-end, exactly where privileged data went|
|HIPAA|Healthcare (patient data)|API/SDK — but only on the exact setup covered by a signed BAA|A BAA only covers the specific configuration it names, not everything|
|GDPR / data residency|Anyone with data location rules (e.g., EU data)|CSP route (Bedrock/Vertex) with region locked to the approved location|Must prove data never leaves the approved geographic boundary|
|FedRAMP (government)|US federal/government work|Claude for Government, Bedrock GovCloud, or Vertex Assured Workloads|Requires specially certified gov-grade cloud environments, not regular commercial cloud|
|Internal data-residency policy|Any org with its own IT/vendor rules|Whatever cloud vendor is already on the company's approved list|Pure company policy — not about technical capability|









\------------------------------------------------------------------------------------------------------------------------------------------------------------







#### Evals as Acceptance Criteria





Normal (wrong) order: Build the system → see if it looks okay → test later.

Correct order: Write your evals first, before writing production code.



Why this matters — 3 reasons:



1.Forces you to define "success" in measurable terms upfront

2.Exposes hidden assumptions early, when they're cheap to fix

3.Gives you a gate to check if any future change (new model, new prompt) actually improved things



The 5-Stage Eval Workflow:

1\. Define the task - State exactly what behavior you're testing, in specific, measurable terms

2\. Build the golden dataset - Collect real inputs — including edge cases — with known correct answers

3\. Run automated checks	- Fast, cheap, code-based checks for clear-cut behaviors (format, schema)

4\. Score with a judge - For subjective behaviors (tone, reasoning quality), use another AI model to grade

5\. Interpret and act - Look at overall scores AND category breakdowns — a rising average can hide a hidden regression



3 Types of Evals:

1.Code-based

Clear-cut, objective checks (valid JSON? right schema? correct lookup?).

Very cheap, instant

2.Model-based (LLM judge)

Subjective stuff — tone, reasoning quality, instruction-following

costs an API call per check

3.Human review

High-stakes or brand-new behaviors nothing else can trust yet

Very expensive, slow



The Grading Ladder: Climb Only When You Must

1.Code-based first — if it can be checked in code, check it in code

2.LLM-as-judge next — only for things that need judgment; use clear rubrics, fixed answer choices (not free-form), and grade with a different model than the one being tested (to avoid bias)

3.Human grading last — reserved only for the highest-stakes or most novel cases





Judge Calibration: Don't Skip This

The trap: An LLM judge can be confidently wrong. If you never check it against real human judgment, you might trust a broken grading system without knowing it.

The fix: Test the judge against a set of human-labeled examples first, and confirm it agrees closely enough with humans before trusting it at scale.





Turning a Vague Requirement into a Real Eval:

|Step|What to do|
|-|-|
|1. Get specific|Turn a vague goal into an exact behavior|
|2. Set a threshold|Decide the pass/fail bar, from business needs — not from what your prototype happens to hit|
|3. Identify failure modes|List what "wrong" looks like|
|4. Include adversarial inputs|Add messy real-world cases, not just clean ones|





#### 

#### POC to Production



Your eval suite tells you if the system is correct. It tells you nothing about whether it can afford to run correctly at real production volume.



The 4 Things a Demo Hides:

|Dimension|Why it's invisible in a demo|How it fails in production|
|-|-|-|
|Cost|10–50 requests/day = tiny bill|Real volume produces a bill way over budget|
|Latency|Demo runs one request at a time|Under real concurrent load, cause SLA breaches|
|Reliability|No retries needed — developer just refreshes|One transient error takes down the entire user-facing system|
|Failure modes|Only tested on expected inputs|Real users send inputs nobody planned for|



Cost \& Latency Modeling: Do the Math BEFORE Building:

3 inputs you need: call volume + token budget per request + model tier → estimate monthly cost before writing code.



Common mistake: Using average token count to estimate cost. Real usage is often skewed — most requests are short, but a few long ones eat a disproportionate share of total cost.



Same idea for latency: Use p95 (the value 95% of requests fall under), not the median — SLA breaches come from the slow tail, not the typical case.



Best cost/latency lever: Prompt caching



3 Reliability Controls to Build In:

|Control|What it does|Where it belongs|
|-|-|-|
|Retry with backoff|Retries transient errors (rate limits, timeouts) with increasing delays|Close to the API call|
|Fallback chains|Auto-routes to a backup model/cached response instead of erroring out|Orchestration layer|
|Circuit breakers|Stops sending requests to a failing dependency once errors cross a threshold|Service boundary|



Failure Modes by Architecture Type:

|Architecture|Breaks first because of...|Fix|
|-|-|-|
|Agent|Unbounded tool use / growing context|Set turn limits, token budgets, minimal tool set; eval its stopping behavior|
|RAG|Retrieval quality drifts as docs change|Monitor precision/recall as a metric; separate live vs. static queries|
|Document pipeline|No exception path for low-confidence extractions|Add confidence scoring; route uncertain cases to human review|
|Orchestrator-subagents|Failures blur across boundaries, silent gaps at synthesis|Define recoverable vs. unrecoverable failures; shared trace ID; reconcile coverage|







#### Use-Case Sizing and Feasibility



Before writing any code, you need to answer 2 separate questions:



1.Can this be built within budget and constraints? (Feasibility)

2.Is it actually worth building? (Business value/ROI)



Part 1: Sizing a Use Case (Cost Modeling BEFORE Coding)

|Step|What to do|
|-|-|
|1. Call volume|Get this from the business owner, not developer intuition or a sample dataset|
|2. Token budget per request|Model the full distribution (not just average) — account for both typical and extreme-length inputs; use caching if the system prompt is long/stable|
|3. Project monthly cost|Volume × input rate + volume × output rate; use cache rates where applicable; compare against the budget ceiling — if it's over, change the architecture before writing code|
|4. Sensitivity analysis|Ask: what if volume doubles? What if inputs skew toward long documents? This shows how fragile your assumptions are|



Part 2: Scoping a Use Case (4 Steps)

|Step|What happens|
|-|-|
|1. Requirement → capability list|Break a vague goal ("process claims") into distinct, named capabilities|
|2. Capability list → architecture sketch|Assign each capability an owner: Claude / existing system / human (the Module 1 decomposition, applied)|
|3. Architecture sketch → boundary conditions|State exactly when the design works and when it doesn't|
|4. Boundary conditions → SOW|Put these limits in the contract, so both sides know what's in/out of scope|



Part 3: Technical Feasibility Assessment

|Verdict|Meaning|
|-|-|
|Feasible as scoped|Works cleanly, within budget/SLA, no compensating controls needed — but document your assumptions, since they can change|
|Feasible with constraints|Works only under specific conditions (doc length limits, refresh schedules, human review gates) — document every constraint explicitly|
|Not feasible|Some limitation can't be fixed within scope/budget — this is a correct, valuable answer that avoids a costlier failure later|



Part 4: ROI Mapping — Is It Worth Building?



4 Steps, in the business's own language:

1.Name the baseline — measure today's process in a unit the business already tracks (e.g., analyst-hours per claim), from real operational data, not guesses

2.Predict the post-deployment state — same unit, but if human review is required, don't project full automation — that overstates value

3.Subtract the run cost — use the actual sizing-model cost (not a rough average) as the recurring cost to subtract

4.State payback period + sensitivity — how long until gains cover build+run cost, and how that shifts if assumptions are wrong



5 ROI Pillars to Map Claims Against

Efficiency, Transformation, Productivity, Solution cost, Performance SLAs







#### Enterprise integration patterns: identity, auth, data, and observability



\- Compliance (HIPAA, GDPR, FedRAMP, privilege, residency) is checked FIRST — it eliminates entry points/routes before anything else is decided

\- 5 entry points exist: Direct API (full control, full responsibility), SDK (convenience wrapper, default choice), Claude Code (developer-only, never for customer-facing products), Agent SDK (multi-turn agent loop inside your own app), MCP (standard way to connect tools, adds a protocol layer)

\- Attorney-client privilege: needs a firm-owned gateway, SSO login, full audit log the firm controls

\- HIPAA: needs a BAA covering the exact configuration used (not just the provider generally); minimum necessary PHI only; reference IDs preferred over full data

\- GDPR/residency: needs execution region locked/pinned; direct API's inference\_geo doesn't support EU pinning, so EU residency needs a cloud route instead

\- FedRAMP: only specific authorized paths work (Claude for Government, Bedrock GovCloud, Vertex Assured Workloads) — regular Enterprise API doesn't qualify

\- Internal policy: just use whatever cloud vendor is already approved by procurement — not an engineering choice

\- After compliance, 4 more layers matter: identity/SSO, authorization, data handling, observability

\- User identity must be verified server-side and injected into the prompt by your system — never trust identity claimed by the user in their message (it can be faked)

\- Only include extra user context (role, department, permissions) when it's actually needed to shape the response — not by default

\- The Claude authorization layer must respect the same access rules as your existing systems — otherwise users can reach data through an unguarded path

\- The context window is NOT a privacy boundary — anything sent to Claude crosses the wire and likely gets logged by your own app, even though Anthropic doesn't retain it by default

\- Only put data in the context window if Claude actually needs it for the task — use reference IDs instead of full sensitive fields when possible

\- LLM systems fail silently — they don't crash, they just give a subtly wrong answer, so normal error-logging alone won't catch problems

\- Log 4 things always: the request (model, tokens, prompt ID), the response (output tokens, latency, stop reason), the context (user role, session ID, caching), the outcome (was it accepted downstream)

\- An action taken but not logged = an action a security reviewer will treat as unapproved — no audit trail means no agent approval

\- Audit every connected tool: keep only what's essential, remove what's merely convenient, document why each removal happened

\- In multi-agent setups, scope each subagent's tools narrowly to its own task only

\- Fixing missing PII redaction after the fact (redacting old logs) is far more expensive than preventing it upfront

\- Building observability after an incident means reconstructing what happened from a system that wasn't designed to answer that question

\- Shared API keys across multiple tenants make it impossible to trace which tenant caused a rate-limit spike — use separate keys per tenant for attribution





#### 

#### A/B Testing \& Observability



Once Claude is integrated into the enterprise stack, 2 new questions matter:



Is it performing well? → Observability answers this

Can we improve it, and prove the improvement is real? → A/B testing answers this



Structured A/B Testing: The 4 Required Components

|Component|What it needs|
|-|-|
|Hypothesis|Specific, testable — names the treatment, metric, and threshold|
|Treatment(new version)/control assignment(current version)|Random, consistent per user/session|
|Primary metric|Chosen BEFORE the experiment runs|
|Sample size|Calculated in advance, accounting for LLM output variance|



Bad hypothesis: "The new prompt is better" (names nothing specific)

Good hypothesis: "Replacing summarize with extract-action-items will increase task success by 5%+ without hurting latency"



Reading Results Without Overclaiming

2 questions to ask before declaring a "winner":

1.Is the improvement big enough to justify the cost of maintaining the new version?

2.Did any secondary metric get worse (e.g., cost jumped 30% for a small accuracy gain)?



Live A/B Test vs. Shadow Testing

||Live A/B Test|Shadow Testing|
|-|-|-|
|What happens|Some real users get the new version, rest get the old|New version runs silently in the background; only old version is ever shown to users|
|Who sees the new output|A portion of real users|Nobody — it's logged, not served|
|Best used when|Risk is low and traffic is high enough for a fast, meaningful result|A bad output is too risky, traffic is too low, or it's a regulated industry|
|Upside|Real user behavior signal (did they accept it? follow up?)|Zero user exposure — completely safe to test|
|Downside|Some users experience the regression before you catch it|No real user feedback — must grade with offline rubrics/golden answers|



Observability at Scale: 4 Layers

1.Request-level tracing — log everything per request: model version, tokens, latency, stop reason, tool calls

2.Metric aggregation — roll this up into dashboards: cost, latency (p50/p95), success rate, error rate

3.Anomaly detection — set alert thresholds (e.g., cost >150% of 7-day average); gradual "model drift" needs periodic comparison, not just thresholds

4.Change attribution — when a metric moves, figure out why: was it the model drifting, the input data shifting, or a model version update? Each needs a different fix



Failure Taxonomy: What Kind of Failure Is It?

|Failure type|What it means|The fix|
|-|-|-|
|Prompt failure|Instructions were vague, model guessed|Fix the prompt|
|Hallucination|Confident, fluent, but ungrounded content|Add retrieval/tool use/verification — NOT just better wording|
|Model mismatch|Wrong tier for the task, or swapped without re-eval|Re-select the model, gated by an eval|
|Orchestrator-worker failure|Something broke in a multi-agent system|Trace across both orchestrator and subagents to find where|



Discernment: Judging Output Quality (Not Just Watching Metrics)



Discernment = actively judging whether outputs are actually good — not just watching a metric go up or down.



Classify each output: acceptable / needs revision / needs override — and feed that back into evals and monitoring.



Why this matters: A team that only watches metrics move, without ever checking if the underlying outputs are genuinely good, can miss real quality problems hiding behind healthy-looking numbers.







\------------------------------------------------------------------------------------------------------------------------------------------------------------







#### What the model's training enforces versus what you own



The Core Question This Answers:

How much safety is already built into Claude, and how much do YOU still have to build yourself?



Getting this wrong in either direction is bad: assuming Claude covers something it doesn't (dangerous), or duplicating protections Claude already handles (wasteful).



Claude's Built-In Layer: Training-Time Alignment

Anthropic trains Claude against a written "constitution" — a document describing the values/behaviors Claude should follow (most recent version: January 2026).



What this does:

\-Shapes how Claude responds to ambiguous/sensitive requests

\-Sets a priority order for conflicting goals: be broadly safe → be ethical → follow guidelines → be genuinely helpful

\-This ordering is holistic, not rigid — Claude weighs these together rather than mechanically checking boxes in sequence



What this reliably covers:

Broad, general-purpose harm — the kind of harm that applies no matter who's using Claude or for what.



What it does NOT cover:

Your specific domain policy, your data-handling rules, your authorization model.



Two Layers, Two Different Jobs

|Layer|When it applies|What it handles|Who owns it|
|-|-|-|-|
|Training-time alignment|Built in before any deployment exists|Broad, general harm — refusing clearly dangerous requests|Anthropic|
|Inference-time control|Configured by YOU for your specific deployment|Your domain-specific rules — system instructions, input/output checks, tool permissions, human review gates|You (the architect)|



The 4-Layer Stack:

|||Who owns it|
|-|-|-|
|Trained behavior|Claude's built-in general safety|Anthropic|
|System-prompt instruction|your specific rules stated in the prompt|Architect|
|Runtime screening|checking inputs/outputs as they flow through|Architect|
|Authorization|verifying what this specific user/role is allowed to do|Architect|





#### 

#### Risks, limitations, and failure modes of LLM systems



The 5 Recurring Risk Categories:

1\. Direct prompt injection

A user writes input specifically designed to override the system prompt's instructions and hijack Claude's behavior.

2\. Indirect prompt injection

Malicious instructions sneak in through retrieved content or tool outputs

3\. Token-budget exhaustion

Oversized or deliberately "padded" inputs eat up the context/output budget

4\. Tool and action abuse

The model gets tricked into calling a real, side-effecting tool (like sending money, deleting a file) outside of policy

5\. Data exposure

Sensitive data ends up in the context window or in logs where it shouldn't be







#### Placing Screening \& Authorization



The Core Problem: What Happens When a Guardrail ITSELF Fails?



A failing guardrail can still look healthy. If a screening service errors but traffic keeps flowing anyway, requests continue — but the protection is silently gone.



The critical decision you must make explicitly: When a guardrail fails, does it fail open (let traffic through unprotected) or fail closed (block until healthy again)?



If you don't decide this explicitly, the default is almost always "fail open" — silently giving you zero protection while looking fine.



Where the 3 Guardrails Sit in the Request Path

|Guardrail|When it runs|What it decides|
|-|-|-|
|Input screening|Before the model call|Should this request even reach the model?|
|Output screening|Before the response reaches the user|Is what the model produced safe to return?|
|Tool-call authorization|Before any side-effecting action (email, refund, DB write)|Is this caller allowed to do this action, right now?|



Model-Based vs. Deterministic Checks: Which for Which?

|Decision Point|Use Model-Based When|Use Deterministic When|
|-|-|-|
|Input screening|Catching ambiguous jailbreak/injection patterns that rules can't fully capture|The rule is clear-cut (blocklist, regex, length check)|
|Output screening|Judging toxicity/policy compliance — needs language understanding|Checking for a known string, forbidden field, schema violation|
|Tool authorization|Rarely|Almost always — must be deterministic, so it's auditable and provable|



Why chain both types:

\-Model-based checks can be evaded (clever phrasing bypasses them)

\-Deterministic rules are brittle (miss anything not explicitly anticipated)

\-Neither catches everything alone — so you stack them in series, each covering the other's blind spot



The Second Injection Vector: Hidden in Retrieved Content



User-input screening only catches what the user typed directly. It does NOT catch malicious instructions hiding inside retrieved documents or tool outputs.



Why this matters: In RAG or agentic systems, a poisoned document or tool response can carry hidden instructions that Claude treats as legitimate — and it slips past input screening entirely, because it never came from the user's message.



The fix: Run the same model-based screening on retrieved content and tool outputs before they enter the context — a separate, dedicated control, because the source (and therefore the blind spot) is different.



Handling Refusals from Claude's API

\-When Claude's built-in safety triggers, the API returns stop\_reason: "refusal" with a stop\_details object naming the policy category (e.g., cyber, bio) — available on newer models

\-Don't treat every refusal the same — read the category and route accordingly

\-After a refusal: reset the conversation context (remove/rephrase the triggering turn) — resending the same context just returns more refusals



Fail Open vs. Fail Closed — The Architect's Choice

||Fail Open|Fail Closed|
|-|-|-|
|What happens on error|Traffic passes through unscreened|Traffic is blocked until the control is healthy|
|Risk|Silent, invisible loss of protection|Availability impact (requests blocked)|
|Best practice|Almost never the right default|Generally the safer choice for guardrails you built|



The Full Guarded Request Path (Put Together)

User request

→ Input screening (model-based + deterministic, fail closed)

→ Model call

→ Output screening (judge model or validator, fail closed)

→ Tool-call authorization (deterministic allowlist + identity/scope check)

→ Response to user



Skill Supply-Chain Security



The objection: "Skills are a black box — how can I trust one before it runs?"



Why this is risky: A skill bundles code + instructions together. A malicious skill can carry a code-execution exploit that runs the moment it's invoked — and neither input filters nor output monitoring will catch it in time, because the threat was baked in upstream, before the conversation even started.



The Defense: Move Earlier in the Chain

1.Audit before trusting - Anomalous calls, Out-of-scope behavior

2.Run It in a Sandbox Anyway

3.Only Use Skills From Trusted Sources







#### Fairness, Unequal Outcomes \& Transparency



The New Kind of Failure This Covers



Everything so far has been about stopping bad outputs from leaving the system. This lesson is about something sneakier:



An output can pass EVERY safety check — and still produce different outcomes for different people.



That's not something a normal filter catches, because on the surface, the output looks completely normal. It's harder to detect and harder to attribute than a typical failure.



The 4 Places Unfairness Can Sneak In

|Entry Point|How bias sneaks in|
|-|-|
|Retrieval corpus|The documents Claude retrieves might over- or under-represent certain groups — so the context Claude sees is already skewed|
|Prompt framing|The way a prompt is worded can bake in an assumption that quietly pushes outcomes in one direction|
|Few-shot examples|The example cases you show Claude can carry the same skew as the corpus|
|Downstream routing|What happens after Claude's output — some groups might get routed down different, less favorable paths|



3 Different Audiences Need 3 Different Explanations

|Who's Asking|What they need|What you must capture to give it to them|
|-|-|-|
|The affected person|A clear, actionable reason why the decision affected them|The specific inputs and reasoning behind their outcome, in plain terms|
|A regulator|Proof similar cases were treated consistently, and that decisions can be reconstructed on demand|A durable, queryable record of inputs/outputs/decision path|
|Your own build team|Enough detail to debug why a flagged decision went wrong|The full trace: prompt, retrieved context, output, every routing step|







#### Routing decisions to people



Which decisions need a human to weigh in BEFORE they take effect, and what should that person see?



What Sets the Stakes of a Decision

1.Reversibility	- How easily can a wrong decision be undone?

2.Cost of being wrong - What does the mistake actually cost if it goes uncorrected?

3.Confidence - The system's own self-reported certainty score



Reversibility + Cost = the actual stakes of a decision. Confidence doesn't change the stakes — it just tells you how likely this particular output is wrong, which helps decide how much volume you can safely skip reviewing.



The Combined Rule

Route to a human when: low confidence AND (irreversible OR high-cost).

Let it run automatically when: confident AND reversible AND low-cost.



Where the Human Sits: A Safety-vs-Speed Tradeoff

|Placement|What it gives you|What it costs|
|-|-|-|
|Pre-action approval|The action cannot take effect until a person approves it, so nothing irreversible happens unreviewed.|It adds latency to every routed decision and a person must be available, so it does not scale to high volume.|
|Post-action audit|The action runs immediately and a person reviews it afterward, so throughput stays high.|A wrong action has already taken effect by the time it is caught, so it only suits reversible, lower-cost decisions.|
|Sampled review|A fraction of decisions are reviewed to monitor quality without slowing the overall process.|A bad decision can slip through unsampled, it monitors the system rather than guarding individual outcomes.|



The "Consent Fatigue" Trap:

If a system asks for approval dozens of times in a row, humans start clicking "approve" without actually reading anything — this is called consent fatigue.



The fix used in Claude Code: Instead of approving every single step, a person reviews the plan as a whole, at higher-value checkpoints (like plan review or exception handling) — not every micro-action.







#### Turning Compliance Obligations Into Real Controls



Earlier, compliance rules (HIPAA, GDPR, FedRAMP) were used to filter out which entry points/routes were even allowed. That gets you to a compliant option — but that alone isn't proof you're actually following the rule.



A reviewer never accepts "we picked a compliant entry point" as proof of compliance. They ask: who owns this control, and what evidence proves it's actually running?



The Key Insight: Regulations Say WHAT, Not HOW



HIPAA, GDPR, FedRAMP state outcomes ("data must be protected"), not implementations ("use this specific technical setup").



You have to fill in the gap — turning a vague legal outcome into something concrete.



Every obligation must become 3 specific things:



A technical control — the actual mechanism that achieves the required outcome

An owner — a specific person accountable for it

Evidence artifact — proof, right now, that it's actually operating (this is the part most often forgotten)



Worked Examples: Obligation → Control → Evidence → Owner

|Obligation|Technical Control|Evidence a Reviewer Accepts|Owner|
|-|-|-|-|
|HIPAA (protected health data)|HIPAA-ready plan/config with a signed BAA, HIPAA compliance enabled, only eligible features used|The signed BAA + admin setting screenshot + eligible-feature list|Security lead|
|FedRAMP (government workload)|Deliver through an authorized route at the required impact level|The route's authorization record + proof the workload runs exclusively on it|Platform owner|
|Data residency|Configure regional processing/storage; check logs, caches, monitoring stay in-region too|Residency config + a data-flow record showing where every copy actually lives|Data owner|
|Decision transparency (cross-framework)|The decision logging system|An actual sample reconstruction of one real decision, pulled from the live log|Architect|







\------------------------------------------------------------------------------------------------------------------------------------------------------------







#### A discovery call is a structured elicitation, not a conversation



A discovery call (talking to a stakeholder about what they want) isn't just a casual chat — it's a structured process to figure out the real requirements, not just their surface-level wishes.



The 3-step filter:

1. Listen – hear what the stakeholder says in plain business language.

2\. Translate – turn their vague wish into real requirements and open questions.

3\. Write it down – so the design isn't based on guesswork later.



The big skill: Translation (turning wishes into real constraints)



Stakeholders speak in preferences ("make it feel seamless"), but you need constraints (actual measurable rules) to design anything.



Example: A stakeholder says "we want this to feel seamless." That word alone tells you nothing you can build with. So you ask: "What would make it feel NOT seamless?" Their answers might reveal:



"User shouldn't wait more than 2 seconds" → that's a latency target

"User shouldn't have to re-type info we already have" → that's an integration requirement

"Errors shouldn't show a scary technical message" → that's a safe failure design

"Everything should happen in one app" → that's a workflow rule



The 4 questions to ask in every discovery call:

1. What must the system DO? (its job)

2\. What must the system NOT do? (its limits — ask this directly, people rarely say it on their own)

3\. What must it COST? (budget, speed, volume limits)

4\. What must it PROVE? (evidence needed, especially for legal/compliance stuff)



The output: a translation table

What they said

What constraint it really means

What design decision that forces

Any assumption you're making (to double check later)

example,

|They said|Really means|You design|Assumption|
|-|-|-|-|
|"Clinicians will review the output anyway"|A human must approve it before it's official|Add a mandatory human-approval checkpoint|Confirm this review is actually required, not optional|







#### Tradeoffs





Part 1: Presenting tradeoffs



Every design choice has a tradeoff — you gain something, you give up something else. Your job isn't to pick the "right" answer for them — it's to lay out the choices clearly enough that they can decide.



Use this 4-question frame:

What do we gain?

What do we give up?

What does it cost to reverse this choice later? (the one people usually skip — but often changes the whole conversation)

What does this do to our compliance situation?



Example: Say you're choosing between "stuff the whole document into the AI's memory" vs. "only fetch small relevant pieces when needed."



Gain: Simpler to build, everything's visible at once.

Give up: Gets expensive and slow as usage grows.

Reversal cost: If costs spike later, you'll have to redesign the whole thing — plus explain to your boss why you didn't see it coming.



Key point: Don't just say "here's my technical recommendation." Translate it into: "Here's what happens to your business if we choose wrong." Executives don't care about "latency" — they care about "will this hurt us later?"



Present it as a package, not a verdict — options considered, criteria used, your recommendation, and the risks left over. That way the stakeholder can defend the decision to their boss too.



Part 2: Designing a good demo



A discovery call can go great, and then a bad demo can ruin all that trust.



Two kinds of demos:

❌Capability demo — "Look what this AI can do!" (generic, feels like a sales pitch)

✅ Scenario-specific demo — "Here's what it does with YOUR exact workflow and data" (feels real, builds confidence)



📌 Example: If you're demoing to a hospital, showing a generic "AI answers questions about a PDF" won't impress them. But showing it handling their actual nurse dictation workflow with realistic patient-note formatting? That lands.



4 things to decide before building the demo:

Pick a familiar scenario — something they'll instantly recognize from their own work.

Choose 1–2 limitations to mention upfront — don't hide them; naming a limit early makes you look disciplined, not weak.

Work with the sales team first — so the demo answers what the buyer actually asked, not random stuff.

Use realistic-looking data (anonymized if needed) — fake generic data makes the whole demo feel fake too.



📌 Example on limits: If a buyer asks "what does this NOT handle well?" — a vague answer kills confidence. A clear answer like "it doesn't do X because of Y, and if you ever need X, here's what that requires" makes you look trustworthy, especially in regulated industries like healthcare or finance.







#### Feedback Loops



Once an AI system is live (in production), it doesn't stay perfect forever — it can quietly get worse over time without any dramatic failure. You need a feedback loop to catch that, and an SLA to define what "good enough" means and what happens when it's not met.



Why systems drift without you noticing

📌 Example: A customer support AI launches and works great — fast, on-brand, accurate. But months later, usage patterns shift, questions get trickier, and slowly the answers start being a bit slower or a bit worse. Nothing crashes or breaks — it just quietly gets worse, and if nobody's watching, users notice before you do.



Observability vs. the Feedback Loop (they're different things!)

Observability = the raw sensors: latency numbers, error rates, quality scores. It just shows you data.

Feedback loop = the decision-making layer on top of that data — deciding what actually matters and what to do about it.



The 5-step feedback loop:

1. Signals – What is the system showing us?

2\. Triage – What needs attention now vs. later?

3\. Decide – Team fix? Stakeholder review? Or no action needed?

4\. Act – Make the correction, update a rule, or escalate.

5\. Review – Did the fix actually work? Should the rule change?



SLAs: naming the promise you're making



An SLA (Service Level Agreement) just answers 3 questions:

What are we measuring? (e.g., response speed)

What counts as a failure/breach?

What happens if it breaches?



Important rule: The thresholds shouldn't be made up randomly — they should trace back to something real:



Latency → based on actual user expectations

Availability → based on how critical the system is to the business

Quality → based on the eval scores/acceptance criteria you already set earlier



📌 Example of a bad SLA: Picking "99.9% uptime" just because it sounds impressive, with no connection to what the business actually needs — that's an arbitrary, indefensible number.



Watch out: Cost is the #1 thing that breaks after launch



This connects to the CTO story from before! A pilot/demo runs at small volume, so cost looks tiny. But production volume can be 10–100x bigger than the pilot — so a "cheap" cost per call can turn into a huge, unexpected monthly bill.



Fix: Before launch, give the stakeholder:

A cost forecast at real production volume (not pilot volume)

A plan for controlling spend (caching, cheaper model tiers, budget alerts)

This should be discussed before the first invoice shocks them — not after.



Regulated industries need scheduled check-ins too:

In places like healthcare, you sometimes need to review things on a schedule, even if nothing looks wrong — not just react when something breaks.









#### Documentation for handoff and audit



Documentation isn't just "notes about the system" — it's what keeps the system safe and working after you leave. If the reasoning behind your decisions only lives in your head, it disappears the moment you're gone.



One document, THREE different readers:



1\. The person taking over (handoff recipient)

They need to know not just what was decided, but what was rejected and why.

📌 Example: If you chose "small AI model" over "big AI model" because of cost, but you never wrote that down — the next person might switch to the big model thinking "this will work better!" without realizing you already tried that path and rejected it for a good reason. Without the rejected options, they can't tell why things are built the way they are.



2\. The compliance auditor

They don't want to hear "trust us, it's secure" — they want proof.

For every legal/regulatory requirement, the doc needs:

What's the requirement?

What control handles it?

Who owns that control?

What's the evidence that it's actually working?

📌 Example: Saying "we have data encryption" isn't enough. You need something like: "Requirement: patient data must be encrypted → Control: AES-256 encryption on database → Owner: Security team → Evidence: encryption audit log from \[date]."



3\. The returning Architect (you, months later!)

Even you will forget the details eventually. So the doc must be so clear that:

Every decision has a date

Every guess is clearly labeled as an assumption, not stated as fact

Every unfinished item has an owner and a clear way to know when it's resolved

The test is practical: after reading the document, can a competent Architect who was not present at the design sessions make a safe change to the system? If the answer is no, the document is not complete.







#### Entry Point



Once your AI system is actually live, two things matter a lot: which "route" it runs through (like which door it enters through), and writing down proof of the value it created (so people can point to it later).



Part 1: Picking the right "entry point" (route)



📌 Simple analogy: Think of these like different doors into the same building. Each door has its own rules — some are faster, some fit better with certain companies' existing systems (like if a company already uses AWS for everything), and some have specific regional/legal requirements.



Quick guide:



Route	          	    Best for

Direct API	            Default choice — fastest access to new features

AWS Bedrock	            Company already standardized on AWS, needs data to stay in-region

GCP Vertex AI	  	    Company already standardized on Google Cloud

Microsoft Foundry (Azure)   Company's procurement/legal rules specifically require it



Why multi-platform setups get tricky



If your system uses more than one entry point at once (e.g., API for one task, Azure for another), new problems pop up that don't exist with just one:



The same AI model might have a different name/ID depending on the route.

New features might arrive later on some routes than others.

Regional settings (like "keep data in Europe") must be manually configured — if you just use default settings, you might accidentally break a data residency rule without realizing it.



📌 Example: Imagine using the API for normal chat, Claude Code for coding tasks, and a special AWS setup for sensitive regulated data. If nobody wrote down clearly "this AWS route is ONLY for regulated data," someone might accidentally start routing other stuff through it too — without meaning to, and without anyone deciding that on purpose.



Fix: Create a simple map that says: "This entry point handles THIS task, for THIS reason." This stops entry points from quietly taking on jobs they weren't meant for.



Part 2: The "Outcome Document" — proving the value you created



6 things it should include:



Use case – what the system does (and doesn't do)

Metric before – how things were before the AI was added

Metric after – how things improved after

Control – proof this improvement is real and auditable, not just claimed

Owner – who's responsible for tracking this going forward

Reuse potential – can this same solution be used for other customers/projects?



📌 Example: Instead of just saying "we deployed an AI system," a good outcome doc says: "Customer support response time went from 24 hours → 2 hours (Before/After), verified via ticket logs (Control), owned by the Support Ops team (Owner), and this pattern could be reused for our Retail division (Reuse potential)."







\------------------------------------------------------------------------------------------------------------------------------------------------------------







#### Configuring Claude tooling and environments for teams







Setting up Claude for yourself is quick and easy. Setting it up for a whole team is a different job — you need shared rules, controlled costs, and a way to update/undo things centrally



1\. Environment — everyone starts from the same baseline



Instead of everyone configuring their own personal setup (and slowly drifting apart), the team agrees on one shared starting point: shared config file, agreed tools, and permission rules.



2\. Rollout — don't flip the switch for everyone at once



Rolling out to the whole team in one go usually fails. The better approach: champions → then batches.



How it works:



Pick one "champion" per department first.

Give them time to prove it works on a real task.

They teach a small group of peers.

Repeat until the whole org has adopted it — each group with a local expert already in place.



3\. Skills distribution — how do you share reusable tools with your team?



1\)Org-provisioned skill:

\-Everyone in the company needs it

\-❌ No real rollback — must manually re-upload to fix



2\)Plugin assigned to a group

\-Only specific teams need it

\-✅ Best option — version control + rollback built in



3\)Project skill (lives in the code repo)

\-One team's own project	

\-✅ Rolls back with the code repo itself



4)API skill

\-Used automatically by other software, not humans

\-✅ Supports version pinning



📌 Real example of what goes wrong: A team packaged a "release notes" skill and pushed it out flatly (not through a proper plugin) to 40 engineers. Someone edited the skill and accidentally broke its formatting — and since there was no version control or rollback, the broken skill kept producing bad output for everyone until someone manually fixed it by hand.



Lesson: If more than one person depends on a shared skill, it needs versioning and a way to roll it back — otherwise one bad edit breaks it for everyone with no quick fix.



4\. Spend — set cost limits BEFORE the first bill, not after



Default model – which model people start with

Allowed models – which ones people are even allowed to switch to

Effort level – how hard the model works on a task

Spend/rate caps – limits per user or per team







#### Improving developer workflows with AI tooling



Part 1: Fit Claude INTO the existing workflow (don't make it a separate thing)



AI tools work best when they live inside the tools developers already use — the code editor, the review process, the testing loop — not as a side chat window people visit occasionally.



📌 Example: Instead of developers copy-pasting code into a separate Claude chat window to ask questions, Claude should be built directly into their coding environment and review process — so it's part of the natural flow, not an extra errand.



Two common failure patterns to avoid:

1)Lumpy adoption — only a few enthusiastic developers use it, everyone else ignores it, so the team never really benefits as a whole.

(Fix: the champion → batch rollout from before.)



2\)Stalling at basic chat — the team only ever uses Claude like a Q\&A box, and never levels up to smarter workflows (like tool use or repo-aware help) because nobody set those up.

(Just giving people access isn't the same as actually enabling real use.)



Diligence = taking responsibility for verifying AI output before you use or ship it — the same way you'd check anyone's code.



Part 2: Diligence — don't blindly trust AI-written code



The danger: Engineers can start accepting AI-generated code just because "it looks right and passed the tests" — even if they don't fully understand why it works. That's a quiet, dangerous habit.



The fix: build a Verification Checklist



Before AI-generated code goes to production, it should pass checks across 4 areas:



Correctness – does it actually work right?

Security – is it safe?

Maintainability – can someone else fix/update it later?

Human understanding – does the person merging it actually understand what it does?







#### Supporting debugging and operational issue resolution



When something goes wrong with a live AI system, an Architect's real job isn't just "fix it and move on" — it's teaching the team how to figure it out themselves next time. Quick fixes help once. Teaching the pattern helps forever.



The key distinction: Firefighting vs. Support

Firefighting = you personally fix the one problem, then leave.

Real support = you teach the team the reasoning path — symptom → cause → action — so they can solve the next similar issue without you.



Connecting symptoms to real causes:

The team usually notices the symptom (something looks wrong), but not the cause (why it's happening). The Architect's job is to connect those two.



|What they see|What's likely really wrong|What to check first|
|-|-|-|
|Output quality slowly got worse (no code changes)|The model/prompt changed, or the data being retrieved has drifted|Compare against eval scores; check what changed|
|Latency spiked|Bigger context size, a slow tool, or caching stopped working|Check token counts, slowest tool calls, cache hits|
|Tool randomly fails sometimes|Auth issues, rate limits, or an unhandled error|Trace one failed request start to finish|
|Costs went up (without more usage)|Sneaky switch to a pricier model tier, or caching broke|Check model tier per request + cache hit rate|



Building self-sufficiency: Runbooks + Escalation paths



Runbook = a written guide of "if you see X symptom, it's probably Y cause, so do Z action" — basically the table above, written down for the team's own system.

Escalation path = a clear rule for "here's what the team can handle themselves, and here's when they need to call in extra help."



Goal: The team should only need the Architect for brand-new problems — not ones they've already been taught how to solve.







\------------------------------------------------------------------------------------------------------------------------------------------------------------

