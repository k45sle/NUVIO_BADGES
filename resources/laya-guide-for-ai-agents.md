# Laya: an operational guide for AI agents

**Verified against public sources on September 30, 2026.**

**Use Laya as a bounded decision component inside an application that owns evidence, policy, and execution.** It can select a predefined label, estimate a yes/no probability, or score an ordered scale. A typed answer is useful decision evidence; it does not establish truth, grant permission, or make an action safe.

This guide concerns Convai Innovations' Laya decision-model family. It is separate from Jev, from OpenAI products, and from unrelated projects named Laya. The public project publishes code and model weights under Apache 2.0. “System 1” describes its fast decision role; it is not a certification of reasoning ability or reliability. [Project overview][overview] [English model card][english]

**Evidence labels used here:**

- **Verified interface:** inspected public package metadata, configuration, or implementation.
- **Reported result:** a publisher or researcher's measurement, not a benchmark rerun for this guide.
- **Recommendation:** the operational design proposed by this guide, not a built-in Laya guarantee.

The reviewed package is **`laya==0.3.22`**, published September 29, 2026; it requires Python 3.10 or newer. Its principal inference, routing, confidence, serving, and structured-output modules matched the repository snapshot inspected for this guide. Model weights were not loaded, and inference benchmarks were not rerun. [PyPI release][pypi] [Package configuration][package]

## Contents

1. [Agent decision checklist](#1-agent-decision-checklist)
2. [What Laya is and how it works](#2-what-laya-is-and-how-it-works)
3. [Checkpoint selection and scope](#3-checkpoint-selection-and-scope)
4. [Suitable and unsuitable tasks](#4-suitable-and-unsuitable-tasks)
5. [Architecture and routing patterns](#5-architecture-and-routing-patterns)
6. [Input and question conventions](#6-input-and-question-conventions)
7. [Output semantics](#7-output-semantics)
8. [Confidence, abstention, and escalation](#8-confidence-abstention-and-escalation)
9. [Worked examples](#9-worked-examples)
10. [Failure modes and responses](#10-failure-modes-and-responses)
11. [Safety and security boundaries](#11-safety-and-security-boundaries)
12. [Evaluation and benchmarking](#12-evaluation-and-benchmarking)
13. [Fine-tuning and calibration](#13-fine-tuning-and-calibration)
14. [Deployment considerations](#14-deployment-considerations)
15. [Rollout, monitoring, and maintenance](#15-rollout-monitoring-and-maintenance)
16. [Agent questions and answers](#16-agent-questions-and-answers)
17. [Evidence record and sources](#17-evidence-record-and-sources)

## 1. Agent decision checklist

Use this checklist before adding Laya to a workflow. Treat “unknown” as a reason to gather evidence or retain the existing fallback.

- [ ] **Decision:** The question can be expressed as a fixed choice, yes/no probability, or ordered scale.
- [ ] **Need:** Simple rules, exact lookups, or an existing classifier do not already solve it adequately.
- [ ] **Evidence:** The input contains the facts needed for this decision, with source, freshness, and uncertainty visible.
- [ ] **Scope:** Language, domain, checkpoint, schema, option count, and document length match an evaluated workload.
- [ ] **Budget:** Required evidence and distinguishable option descriptions survive tokenization.
- [ ] **Quality:** Independent held-out evaluation beats an appropriate baseline at the required error cost.
- [ ] **Confidence:** The exact decision policy has been validated; a convenient threshold is not being substituted for evidence.
- [ ] **Consistency:** Related answers satisfy application constraints or are escalated.
- [ ] **Authority:** The application separately checks identity, authorization, and action prerequisites.
- [ ] **Fallback:** Missing context, invalid output, low confidence, timeout, or unavailable inference leads to a defined safe path.
- [ ] **Execution:** Approved actions have bounded scope, deduplication, outcome checks, and a recovery path.
- [ ] **Operations:** Package, checkpoint, schema, calibration, and policy versions are recorded and monitored.

**Decision rule:** Use Laya when this checklist supports a measurable, bounded advantage. Otherwise use deterministic handling, a better-suited model, or human review. Start new integrations in observation mode.

## 2. What Laya is and how it works

Laya is a **non-autoregressive decision model**: it scores answers without generating a response token by token. Its input consists of a **state**, meaning the information being assessed, and one or more typed questions. Its native primitives are `choice`, `noul`, and `score`. [Project overview][overview]

The inspected implementation constructs a sequence containing question type, instructions, option markers and descriptions, and serialized state. A bidirectional encoder reads this sequence; an added transformer decision head scores the option-marker representations. A softmax turns those scores into a probability distribution, with temperature scaling applied during decoding. A separate action head produces `act_probability`. Multiple questions are represented as separate sequence rows that can share a batched forward call; they are not free additions to one fixed-cost document encoding. [Sequence construction and model architecture][common-code] [Inference and decoding][agent-code]

In plain terms: **Laya reads evidence and scores supplied answers. The application interprets those scores and decides what happens next.**

The project's training approach is called **RLCD**, reinforcement learning for calibrated decisions. Published specialist training uses rewards based on proper scoring rules, together with soft teacher targets. A proper scoring rule rewards reporting the underlying probability distribution honestly in expectation. That objective does not prove that a deployed checkpoint is calibrated on new data. [Specialist training description][typed] [Fine-tuning workflow][finetune]

It has no native browsing, tool execution, live system inspection, or general-purpose prose generation in this interface. External tools must gather evidence. A general agent can prepare the state, call Laya, validate its answer, and continue the workflow. If an explanation is needed, generate it from inspected evidence and policy; do not invent a Laya reasoning trace.

## 3. Checkpoint selection and scope

A **checkpoint** is a saved set of trained model weights. Select and evaluate the particular checkpoint, rather than treating the family name as a capability guarantee.

| Public checkpoint | Reported architecture and size | Default total token budget | Intended starting point |
|---|---|---:|---|
| `convaiinnovations/laya` | ModernBERT-large, 421M parameters | 512 | English decisions |
| `convaiinnovations/laya-multilingual` | mmBERT-base, 322M parameters | 1,024 | Multilingual decisions; explicit longer-context use |
| `convaiinnovations/laya-typed-decisions` | ModernBERT-large, 421M parameters | 1,024 | Agent traces, customer service, invoices, and security incidents resembling its specialist training |

These are model-card descriptions and inspected configuration defaults. The state receives only the space remaining after question text and options. The multilingual card documents an explicit `max_len=8192` override; this is not the default and does not establish uniform long-document accuracy. [English card][english] [Multilingual card][multilingual] [Specialist card][typed] [English configuration][english-config] [Multilingual configuration][multilingual-config] [Specialist configuration][typed-config]

**Routing details:** `Router()` defaults to language-based selection between English and multilingual checkpoints. Automatic specialist task detection is off by default. Select the specialist explicitly with `model="typed-decisions"`, or deliberately enable `auto_task_detection=True` after evaluating its behavior. A router route is checkpoint selection, not a decision about the business action. [Router implementation][router-code]

Recommendations:

- Pin a checkpoint for a stable, evaluated workflow. Use automatic routing only when the router itself is tested as part of the system.
- For mixed languages, evaluate language detection, transliteration, short messages, and brand names. Supply a trusted language hint when available.
- Treat the specialist as a domain candidate, not an upgrade for every task.
- Treat context-length changes, question changes, quantization, and calibration changes as changes to the evaluated system.
- Keep scope in a manifest: allowed domains, languages, schemas, maximum input size, output interpretation, and action risk.

An invoice-related ticket can fit the specialist's topic while still requiring novel reasoning. Topic overlap alone does not establish task coverage.

## 4. Suitable and unsuitable tasks

The following are **candidate uses to evaluate**, not promises of performance.

| Task | Suitable bounded role | Required boundary |
|---|---|---|
| Support triage | Assign a queue or identify a request type | Preserve an unknown/review path; evaluate overlapping intents |
| Email organization | Suggest labels such as billing, promotion, or support | Keep deletion, sending, and financial actions behind separate policy |
| Agent routing | Select among evaluated worker or model categories | Judge actual downstream success and total cost |
| Retrieval support | Score whether a passage addresses a question | Compare with retrieval/reranking baselines; verify the final answer separately |
| Workflow monitoring | Label an agent trace as complete, blocked, or requiring review | Compare claims with tool results and explicit completion rules |
| Operations triage | Recommend a diagnostic category from fresh observations | Gather live evidence and control remediation separately |
| Severity scoring | Estimate a defined ordered level | Specify anchors and error costs; inspect the distribution |
| Spam or abuse triage | Add a probabilistic signal to a broader pipeline | Evaluate current adversarial traffic and critical false negatives |

Prefer another mechanism for these cases:

| Task | Better fit or reason to escalate |
|---|---|
| Arithmetic, dates, account balances, exact identifiers | Deterministic computation or verified lookup |
| Extract arbitrary text, amounts, URLs, or names | Parser, extraction model, or generative model with validation |
| Write code, documents, explanations, or summaries | Generative model plus appropriate verification |
| Discover missing facts or diagnose an unseen system | Read-only tools, investigation, and a reasoning agent |
| Complex planning or multi-step causal reasoning | A planner with tools and explicit intermediate checks |
| Hundreds of closely related labels | Evaluate a specialist classifier, shortlist, or hierarchy first |
| Decisions requiring evidence beyond the input | Retrieve or inspect the evidence before classification |
| Grant access, approve payments, delete data, or change security | Deterministic authorization and an independently approved execution process |
| Final medical, legal, financial, or similarly consequential judgment | Qualified review and domain-specific safeguards |

For a rule such as “the measured free disk space is below the configured minimum,” code can check the condition directly. Laya adds value only if the residual decision needs semantic interpretation.

## 5. Architecture and routing patterns

### 5.1 Evidence, decision, policy, execution

Recommended baseline:

```text
Request or event
    -> validate and normalize
    -> collect fresh evidence
    -> deterministic eligibility checks
    -> choose evaluated checkpoint and question schema
    -> Laya inference
    -> validate output, scope, and consistency
    -> application decision policy
         -> bounded authorized action -> verify outcome
         -> stronger model or read-only investigation
         -> human review
         -> safe deferral
```

Keep the executor separate from the predictor. Map a permitted label to a fixed handler; never interpret a label as arbitrary executable text. The executor checks current prerequisites again immediately before acting.

Use an application-owned decision record around the native result:

```json
{
  "request_id": "evt-0182",
  "schema_version": "diagnostics-v1",
  "policy_version": "shadow-v1",
  "disposition": "advisory",
  "reason": "automation not yet evaluated",
  "proposed_handler": "inspect_remote_network",
  "execution_authorized": false
}
```

This envelope is not a Laya output format. Populate it from validated results and trusted policy. For automatic handling, require all of the following: eligible scope, complete evidence, valid and consistent outputs, a passing evaluated decision rule, current authorization, and satisfied action prerequisites. Otherwise record the reason and follow the defined escalation path.

### 5.2 Rules first

Apply inexpensive exact checks before inference. They can handle known events, reject missing prerequisites, enforce mandatory review, and remove impossible actions. Laya handles the remaining semantic decision. This avoids paying for inference where the answer is already established.

### 5.3 A cascade to a stronger model

Laya handles an evaluated easy subset. Cases outside that subset go to a stronger model or investigator. Mandatory-review cases go directly to review.

Choose the routing objective from **downstream task success**, not how plausible a category sounds. A request can be short but require difficult reasoning. Complexity labels need ground truth or measured downstream outcomes.

Estimate total cost:

```text
average cost = initial checks + Laya serving cost
             + escalation rate * fallback cost
             + review cost + expected failure/rework cost
```

A cascade is worthwhile only when the complete workflow beats the fallback alone on the chosen quality, cost, and response-time objectives.

### 5.4 Shortlists and hierarchies

Reduce a large label space with retrieval or a coarse classifier, then ask a smaller choice question. Always measure **candidate recall**: how often the correct label remains in the shortlist. The second stage cannot recover an excluded answer.

For a hierarchy, route to a broad family and then to a specific label. Include a recovery branch when the family is uncertain. Evaluate the whole tree; two accurate stages can still compound mistakes. Probabilities from different stages are not automatically a coherent global distribution.

A recent primary research preprint reports probabilistic inconsistency when decisions are decomposed, including tests of Jev and English Laya. Treat decomposition as an architecture to validate, not a probability-preserving rewrite. [Probabilistic coherence study][coherence]

### 5.5 Long-document handling

Use a larger validated context window, select relevant evidence, or evaluate a windowing strategy. Each changes the information the decision sees.

The inspected `predict_long` implementation scans overlapping windows. Its automatic aggregation takes the maximum `noul` probability and selects the most confident window for `choice` and `score`. It exposes deciding-window metadata. These are local evidence aggregation rules, not document-wide reasoning; the aggregate probability is not automatically calibrated for the whole document. [Long-document implementation][agent-code]

Recommendations: validate against document length and window count; preserve chronology and exceptions; inspect conflicting windows. A rule asking whether *any* passage contains a request is different from asking what the user's *latest* instruction authorizes.

## 6. Input and question conventions

### 6.1 State construction

Use strings or JSON-compatible state objects supported by the interface. A JSON object helps organize evidence, but Laya reads serialized text; its keys are not a guaranteed symbolic reasoning system.

Recommended state fields:

```json
{
  "event_id": "evt-0182",
  "observed_at": "2026-09-30T14:00:00Z",
  "request_text": "The player cannot connect.",
  "observations": {
    "service_running": true,
    "local_health_check": "passed",
    "remote_health_check": "failed"
  },
  "unknowns": ["whether the remote network is working"],
  "source": "read-only monitoring",
  "allowed_scope": "recommend a diagnostic category"
}
```

This is an application design example, not a required Laya schema. Authority stays outside the model even if an `allowed_scope` field is supplied.

Preserve facts that change the decision: negation, time, speaker, amounts, exceptions, and whether something is a claim or an observation. Represent unknowns explicitly. Avoid unrelated logs and conversation history that consume the evidence budget. If another model summarizes the evidence, evaluate the summary-to-decision chain for omitted or altered facts.

### 6.2 Question definitions

The public shape is a mapping from question ID to definition:

```json
{
  "queue": {
    "type": "choice",
    "instructions": "Which queue best matches the customer's request?",
    "criteria": {
      "billing": "Charges, invoices, payment problems, and refund requests",
      "technical": "Application failures, access problems, and service outages",
      "other": "Requests outside those categories"
    }
  },
  "refund_requested": {
    "type": "noul",
    "instructions": "Does the customer explicitly request a refund?"
  },
  "impact": {
    "type": "score",
    "instructions": "How much does the reported problem prevent use?",
    "criteria": [
      "0: Cosmetic; normal use remains possible",
      "1: Some functions fail; a practical workaround exists",
      "2: Core use is blocked; no workaround is reported"
    ]
  }
}
```

`choice` criteria are labels with descriptions; `score` criteria are ordered levels; `noul` asks a yes/no question and can omit criteria. Option order affects the rendered input. [Question conventions][questions]

Recommendations:

- Ask one operational question per field. Keep dependent questions in separate stages if the later one needs the earlier answer.
- Use stable semantic label IDs and concise distinguishing descriptions.
- Keep the question wording and label order versioned.
- Start with a small set of options. Budget based on rendered tokens, not label count alone.
- Define how ties, overlaps, missing evidence, and multiple intents are handled.
- An `other` label covers an outside category; an `insufficient_evidence` label expresses missing information. They have different meanings.
- Treat both as learned choices, not guaranteed abstention mechanisms. Keep external completeness checks.
- Avoid assuming conversational prompt tricks or “think step by step” will produce reasoning; there is no generated reasoning output.

For ordinal scales, define anchored levels. Laya's raw score uses **zero-based level positions** even if the descriptions contain different numeric values. It is not arbitrary numeric extraction.

### 6.3 Token budgets

`max_len` limits the total sequence. `head_max_len` controls space allocated to question/option text. Increasing option space may reduce state space. Check token-level truncation and option-collapse metadata rather than estimating from character count. The current configs specify head budgets of 192 for English and 256 for multilingual and specialist checkpoints. [Checkpoint configurations][english-config] [Multilingual configuration][multilingual-config] [Specialist configuration][typed-config]

Recommended preflight: validate schema and required fields, inspect rendered option distinctions, then confirm the inference result reports complete evidence. A successful HTTP request does not prove every important token was read.

## 7. Output semantics

The native result includes `answers` and `usage`; router results also include `routing`. Read these meanings carefully:

| Primitive or field | Meaning | Operational interpretation |
|---|---|---|
| `choice` | Highest-probability supplied label | A classification suggestion |
| `probabilities` for `choice` | Distribution over supplied labels | Conditional on this option set and schema |
| `noul` | Probability of true | Apply a task-specific positive/negative decision rule |
| `score` | Expected zero-based level index | Weighted average, potentially between levels |
| `legend` | Mapping of score positions to descriptions | Interpret the scale explicitly |
| `answer_confidence` | Largest probability in the answer distribution | Candidate confidence signal requiring validation |
| `confidence` for `choice`/`score` | One minus normalized entropy | Distribution concentration, not correctness probability |
| `confidence` for `noul` | `max(P(true), P(false))` | Same quantity as its `answer_confidence` |
| `action.act_probability` | Separate learned action-head probability | Advisory signal; never execution permission |

These definitions are verified from the released decoding implementation. **For `score`, `answer_confidence` belongs to the most likely discrete level, not a guarantee about the accuracy of the returned average.** [Output decoding][agent-code] [Confidence definitions][common-code]

For example, a distribution over three levels of `[0.45, 0.10, 0.45]` yields a raw score of `1.0`, although level 1 is the least likely level. Keep the full distribution where disagreement matters.

The schema helper `decide` supports a restricted JSON-schema subset: enums, booleans, and bounded numeric levels. It does not provide arbitrary strings, nested object extraction, or arrays. Its bounded numeric projection uses the most likely level; its boolean projection uses `noul >= 0.5`. Thus `decide` is a convenience transformation, not identical semantics to reading raw `score`. Use `return_details=True` to retain decision evidence. [Structured-output interface][structured]

Recommended output validation:

- Every required question has exactly the expected answer type.
- Returned labels belong to the approved label set.
- Probabilities and scores are finite and within their expected ranges.
- Probability distributions sum approximately to one; allow documented numeric rounding.
- Actual checkpoint, schema, calibration, and policy match the approved manifest.
- No critical state truncation or option collapse occurred.
- Related answers satisfy domain constraints.

Reject or escalate malformed or incomplete results. Missing confidence is unknown, not zero and not approval.

## 8. Confidence, abstention, and escalation

### 8.1 Calibration has to be measured

**Calibration** means that reported probabilities match observed frequencies on a defined population. If answers reported at about 0.8 are correct about 80% of the time, that group is approximately calibrated. This does not certify any individual answer.

There is a material public-source disagreement. The specialist model card describes overconfidence. A September 27 research preprint reproduces approximately the headline accuracy but reports **underconfidence** on the released specialist: a signed confidence-minus-accuracy gap around −0.214. Its frozen escalation policy missed its 10% accepted-set error target. This is one reported study on specified data, not a general performance guarantee. [Specialist limitations][typed] [Calibration and escalation study][assessor]

**Recommendation:** Measure both the amount and direction of miscalibration on your actual checkpoint and workflow. Do not assume every Laya variant needs softer probabilities. Underconfidence wastes escalation capacity; overconfidence admits errors.

### 8.2 Choose the correct signal

Use validated `answer_confidence` rather than treating entropy-based `confidence` as a probability of correctness. For a yes/no decision, use **`noul` itself** to decide which outcome is supported.

Example: if `noul=0.02`, `answer_confidence=0.98` means the model strongly favors **false**. It does not support taking the positive action.

For a score-driven action, evaluate the action boundary directly, such as the probability of a severe level or the error of the expected score. Top-level probability alone does not quantify expected-score error.

### 8.3 Abstention is opt-in and application-owned

The current `min_confidence` option adds `low_confidence: true` to low-confidence raw answers; it retains their answers. Structured `decide` can return `None` for abstained values. The released gate prefers `answer_confidence` but falls back to the older `confidence` field if needed. Do not let that compatibility behavior substitute for a validated application policy. [Gate implementation][confidence-code] [Structured interface][structured]

An application can abstain for reasons beyond confidence: missing evidence, unsupported language, stale state, contradictory answers, an out-of-domain request, or required review. None requires the model to admit uncertainty.

### 8.4 Threshold selection

Recommended procedure:

1. Define the error costs and maximum acceptable error rate for each action class.
2. Fit calibration on data separate from training and final testing.
3. Select candidate thresholds and eligibility rules on validation data.
4. Freeze the policy and assess its accepted subset on untouched test data.
5. Report **coverage**, the fraction automatically handled, and **selective risk**, the error rate within that fraction.
6. Inspect error rates by language, schema, option count, domain, and critical label.
7. Promote only slices meeting the predefined requirements with adequate evidence.

No universal threshold such as 0.85 or 0.95 follows from the model name. A public issue reports that a confidence gate at 20 options selected a worse subset than the overall model in its test. Its examples are evidence of a failure mode, not your deployment's measured rate. [Option-count gating report][option-gating]

For a calibrated binary event probability `p`, a simplified two-action policy can compare expected losses:

```text
loss if positive = (1 - p) * false-positive cost
loss if negative = p * false-negative cost
```

This calculation assumes the listed losses represent the real decision. Add escalation costs, downstream consequences, authorization, and other constraints when relevant.

### 8.5 Escalation destinations

| Trigger | Recommended destination |
|---|---|
| Missing or stale observations | Read-only evidence collection |
| Complex interpretation within authorized scope | Stronger reasoning model |
| Conflicting instructions, uncertain consent, or high-impact action | Human or designated policy review |
| Unsupported domain/language | Evaluated alternative or human review |
| Inference outage or timeout | Known fallback or safe deferral |
| Output inconsistency or malformed result | Validation failure path, with diagnostic logging |

Escalation means changing the method or obtaining evidence. Repeating the same request until a desired label appears does not establish correctness.

## 9. Worked examples

All numerical outputs below are **illustrative**, not measured predictions. These examples demonstrate interpretation and boundaries.

### 9.1 Support triage with the native Python interface

Example for the reviewed release. First inference downloads weights unless already cached.

```bash
python3 -m venv .venv
.venv/bin/python -m pip install "laya==0.3.22"
```

```python
import laya

# Immutable public model revision inspected for this guide.
# Review and evaluate the pinned artifacts before production use.
agent = laya.load(
    "convaiinnovations/laya",
    revision="55cf4c4ebb4ebe31b2550e8bdf3bd21b99753851",
    device="cpu",
)

questions = {
    "queue": {
        "type": "choice",
        "instructions": "Which queue best matches the customer request?",
        "criteria": {
            "billing": "Charges, payments, invoices, and refund requests",
            "technical": "Application faults and service availability",
            "other": "Requests outside those categories",
        },
    },
    "refund_requested": {
        "type": "noul",
        "instructions": "Does the customer explicitly ask for a refund?",
    },
}

result = agent.predict(
    {"customer_message": "I was charged twice. Please refund the duplicate."},
    questions,
)

if result.get("usage", {}).get("truncated", 0) > 0:
    raise RuntimeError("Review required: some evidence was not read")
if result.get("usage", {}).get("options"):
    raise RuntimeError("Review required: options lost token distinctions")

print(result["answers"])
# This example only displays decision evidence. It performs no refund.
```

The load and prediction interfaces are checked against source; this example was syntax-checked but not run with weights. [Loading and prediction][agent-code]

If the returned billing probability were 0.91 and refund-request probability 0.96, a validated policy might route the ticket to billing. Neither number establishes that a duplicate payment actually occurred or authorizes a refund. Verify transactions through the authoritative payment system.

### 9.2 Service incident: diagnose before remediation

Input: a user reports a connection failure; a fresh local health check passes; a fresh remote check fails.

Possible diagnostic labels:

```json
{
  "next_check": {
    "type": "choice",
    "instructions": "Which read-only diagnostic should be prioritized?",
    "criteria": {
      "remote_network": "Inspect remote connectivity and the public route",
      "service_health": "Inspect local service responsiveness",
      "host_resources": "Inspect resource pressure on the host",
      "review": "Available evidence cannot prioritize a diagnostic"
    }
  }
}
```

A suggestion of `remote_network` can select an approved inspection handler. After inspection, fresh evidence may support a separate repair proposal. Restarting a service merely because a classifier selected a category skips causal diagnosis and can interrupt healthy work.

### 9.3 Model routing

Use labels such as `routine_classifier`, `reasoning_model`, and `human_review`, with evaluated task definitions. Mandatory review and tool availability are deterministic gates before routing.

Test the chosen model's actual completion success. Count fallback calls and rework. If the inexpensive route regularly escalates after failure, direct routing to the stronger model may cost less overall.

### 9.4 Correctly interpreting a score

For levels `minor`, `moderate`, `critical`, suppose probabilities are:

```json
{"0": 0.10, "1": 0.25, "2": 0.65}
```

The raw expected index is `1.55`; the most likely level is `critical`; the discrete top-level confidence is `0.65`. A policy concerned with critical incidents should inspect `P(level=critical)`, not round `1.55` and call it a calibrated critical probability.

### 9.5 Missing evidence and negation

Input: “My manager suggested closing the account, but I disagree. Please keep it open.”

The execution layer must require valid cancellation authorization irrespective of model confidence. A public reproduction reports highly confident wrong cancellation choices on negated inputs, including both English and multilingual checkpoints in a small test. Preserve this kind of minimal pair in your evaluation set. [Negation reproduction][negation]

## 10. Failure modes and responses

The responses below are recommendations. A failure reported on a particular test does not establish a universal failure rate.

| Failure mode | Why it matters | Response |
|---|---|---|
| Wrong checkpoint or language route | Confident output may conceal poor language understanding | Test routing; pin or supply trusted hints for known workloads |
| Negation, retraction, or quoted instruction misunderstood | Can invert user intent | Preserve speaker/time/polarity; test minimal pairs; require independent authorization |
| Missing correct option | Softmax still assigns its mass somewhere | External scope check and explicit review path |
| Large option set or shared prefixes | Option token budgets may erase distinctions | Shortlist, redesign labels, or use another classifier |
| Changed label order or wording | Alters the model's rendered question | Version and evaluate schema changes |
| State truncation | Decision may miss an exception or later instruction | Check reported usage; retrieve, widen, or window with validation |
| Poor calibration | Confident errors or excessive escalation | Fit and validate on the target population |
| Multiple outputs conflict | Valid fields may imply an impossible overall decision | Apply deterministic cross-field constraints |
| Window aggregation amplifies a local false positive | More windows create more opportunities for a high score | Evaluate at document level and by window count |
| Expected score mistaken for a label | Average hides a split distribution | Inspect probabilities and legend |
| `noul` confidence mistaken for P(true) | Strong evidence for false becomes a positive action | Read `noul` for polarity |
| Domain or adversarial shift | Historical metrics no longer describe current traffic | Sample outcomes, monitor changes, and restrict automation |
| Quantization/backend change | Borderline labels or probabilities can change | Compare the deployed runtime with its baseline |
| Timeout or resource fallback | Latency and capacity assumptions fail | Bound calls; monitor actual device and fallback behavior |
| Repeated action after retries | One event causes duplicate side effects | Use event IDs and idempotent handlers |
| Hidden executor error | Correct classification fails to produce the desired result | Verify the action's actual outcome |

The project documents option-order sensitivity and an option-budget ceiling. Treat label order as part of the schema and measure robustness, rather than randomizing production options without a tested aggregation policy. [Benchmark limitations][benchmarks]

## 11. Safety and security boundaries

### 11.1 Prediction does not grant authority

Separate these questions:

1. What does the input appear to mean?
2. Is the requested action permitted for this caller and resource?
3. Have prerequisites been satisfied?
4. What is the approved way to execute and verify it?

Laya can assist with the first. Policy and authoritative systems answer the others. A model-selected label and `act_probability` cannot replace those checks.

### 11.2 Untrusted text remains untrusted

Email, documents, logs, and web text may contain instructions intended to influence the classifier. Structured output reduces free-form output parsing, but it does not prevent malicious text from causing a wrong label.

Keep question schemas and executable handlers in trusted application configuration. Do not let arbitrary input redefine labels, thresholds, tools, or permissions. Test attacks that target the desired decision, including disguised instructions, quoted authority, and conflicting messages.

### 11.3 Independent security controls

Use Laya as an additional signal if evaluated; keep authentication, access rules, transaction limits, and audit requirements independent. A classifier with imperfect recall is insufficient as the sole barrier protecting secrets or dangerous tools.

For high-impact operations, prefer a read-only recommendation or review queue. Preserve existing security controls during integrations. Human review should show source evidence and proposed effects, not just a confidence number.

### 11.4 Privacy and model supply chain

Local inference can keep request data on your infrastructure when the complete pipeline is local. Check model downloads, remote fallbacks, integrations, and logging before describing the whole workflow as offline or private.

Pin reviewed package and checkpoint versions. For stronger artifact control, Laya accepts expected SHA-256 file digests; only listed files are verified, and verification is at load time. Its security documentation notes that tokenizer configuration can be rewritten and is omitted from the recommended digest list. A matching digest establishes matching bytes, not trustworthy behavior. [Artifact integrity documentation][security]

Recommendations: protect model/configuration directories, limit privileges, redact sensitive logs, retain only necessary evidence, and fail deployment checks when integrity or revision requirements are unmet. Never regenerate trusted digests merely to silence an unexplained mismatch.

## 12. Evaluation and benchmarking

### 12.1 What public results establish

Publisher-reported typed-decisions results give specialist accuracy **0.766**, base English **0.362**, and a majority-label baseline **0.461**. These numbers concern the stated synthetic workflow benchmark; the specialist was trained on its training split. They do not establish general production accuracy. The same card quotes Jev **0.727** from other published work and explicitly says Jev was not measured there under matched conditions. [Specialist benchmark][typed]

The project reports one-question T4 timings of **39.5 ms for English** and **32.8 ms for multilingual**. These are configuration-specific inference measurements, not a deployment service-level promise. Its benchmark document distinguishes test runs and updates previously unreproduced multilingual figures. Read the run details before quoting a number. [Benchmark report][benchmarks]

The calibration reproduction preprint reports specialist accuracy around **0.767**, while identifying confidence and escalation limitations. Its metrics still assess agreement with synthetic teacher labels. Teacher agreement is not the same as verified correctness in the world. [Independent assessor study][assessor]

Do not combine the best accuracy, best calibration, fastest runtime, and widest language coverage from different checkpoints and tests into an imaginary single deployment.

### 12.2 Build a representative evaluation set

Recommended process:

1. Define the exact decision, downstream action, and cost of each error.
2. Collect representative real cases with lawful, appropriately protected data.
3. Establish labels from authoritative outcomes or qualified review. Mark unresolved cases explicitly.
4. Separate training, calibration/policy selection, and final test data. Avoid leakage across related cases, accounts, templates, or adjacent time windows.
5. Include rare critical events, ambiguous cases, out-of-scope requests, malformed evidence, and attacks.
6. Freeze checkpoint, runtime, schema, and policy before final testing.

Include stress pairs: positive versus negated request; current versus quoted request; presence versus absence of decisive evidence; early versus late evidence; alternate option order; multilingual and transliterated inputs; shorter versus longer documents.

### 12.3 Compare meaningful baselines

Evaluate against a majority-label baseline, simple rules, an existing specialist classifier, and a stronger model where relevant. Use identical states, label meanings, data splits, and outcome criteria.

For a cascade or shortlist, evaluate the full system. An excellent second-stage score on cases where the correct answer was supplied does not measure candidate-retrieval failures. A cheap routing call that sends difficult cases to an inadequate model does not count as success.

### 12.4 Metrics to report

| Objective | Metrics and interpretation |
|---|---|
| Choice quality | Accuracy, confusion matrix, per-label precision/recall, macro F1 |
| Binary decision | Precision/recall at deployed threshold; false-positive and false-negative rates; PR-AUC where useful |
| Ordinal score | MAE, within-tolerance rate, severe-level misses, and ordinal error costs |
| Probability quality | Brier score or log loss, reliability plots, ECE with its binning definition, and signed confidence gap |
| Selective automation | Coverage and accepted-set error; risk-versus-coverage curve |
| Routing quality | End-to-end task completion, fallback rate, rework, and total cost |
| Operational performance | Cold/warm p50/p95/p99 latency, throughput, errors, memory, and queueing |
| Robustness | Per-slice quality; sensitivity to wording, order, language, length, and attacks |

Accuracy measures hard-label agreement. **Brier score** measures squared probability error. **Log loss** strongly penalizes confidence assigned to wrong outcomes. **ECE**, expected calibration error, compares confidence and accuracy within bins; it depends on bins and can hide small dangerous slices. Report counts and uncertainty intervals, not only averages.

Do not call a system safe because it made no critical errors in a tiny sample. As a rough binomial guide, zero observed failures in `n` independent trials gives an upper 95% failure-rate bound around `3/n`; correlated or unrepresentative cases weaken that inference.

### 12.5 Use the evaluation harness appropriately

The official `laya-evals` harness validates JSONL datasets, runs evaluations, emits reports, supports slices, and compares against baselines. A row contains `state`, `questions`, and `expected`; optional tags, language, and model help identify slices. Its binary accuracy uses a 0.5 cutoff, so add custom metrics when your deployed cutoff differs. [Evaluation harness][evals]

```bash
laya-evals validate evaluation.jsonl
laya-evals run evaluation.jsonl --model english --device cpu \
  --slice language --json report.json --markdown report.md
```

These commands illustrate the reviewed interface. Add project-defined acceptance gates; this guide supplies no universal passing accuracy or ECE.

For hardware comparisons, report checkpoint and revision, dependency versions, device, precision, real input lengths, question/option counts, token budgets, batch size, warm-up, concurrency, and whether timing includes downloads, network, or preprocessing. Separate per-request latency from amortized time per decision.

## 13. Fine-tuning and calibration

**Fine-tuning** changes weights to improve a task. **Calibration** adjusts the interpretation of scores. Neither replaces the other.

Recommendations:

- Fine-tune when the task is valuable, stable, and supported by adequate labeled examples, and simpler baselines are insufficient.
- Include hard negatives, rare critical labels, ambiguity, and the production question phrasing.
- If a model supplies training labels or soft probabilities, retain their provenance and review their errors. Distillation can transfer systematic mistakes.
- Preserve a final untouched test set and evaluate beyond the fine-tuning domain before making broader claims.

The public training workflow includes soft-target RLCD training, held-out calibration, evaluation, and publication. Its documentation warns that inherited option-count temperature settings can override newly fitted per-type settings; configuration must be shipped consistently with the trained weights. [Fine-tuning documentation][finetune]

The released API includes `fit_temperatures`, `save_calibration`, and `load_calibration`. It gives option-count temperatures precedence over per-type defaults, and language overrides can affect the applied temperature. Calibration loading may warn about a checkpoint mismatch while still applying the file; enforce compatibility in the application rather than assuming a warning blocks deployment. [Calibration methods][agent-code]

Recommended calibration sequence:

1. Gather labeled outputs/logits using the exact intended inference configuration.
2. Fit on a separate calibration split, with adequate examples for each intended slice.
3. Confirm the fitted values are actually used after reload; account for runtime clamping and precedence.
4. Evaluate probabilities, action boundaries, and selected subsets on independent data.
5. Version the calibration artifact with its checkpoint, schema, runtime, and evaluation identity.

Temperature scaling normally preserves discrete argmax for positive temperatures, but changes probabilities and can change an **expected score** or threshold-based binary decision. Recheck the behavior you actually deploy.

## 14. Deployment considerations

### 14.1 Start with the smallest useful integration

Choose in-process Python for a single application; a private HTTP service for multiple clients; or an agent integration when the client already uses that protocol. The package exposes optional serving, MCP, structured, ONNX, and agent-framework integrations. Confirm extras and interfaces in the pinned release. [Package extras][package]

Keep a loaded agent resident on frequent paths. Measure CPU performance before adding a GPU. Choose the device based on measured workload and contention, not a headline benchmark.

A rough storage lower bound for weights is parameter count times bytes per parameter: 421M parameters at two bytes is about 842 MB decimal; 322M is about 644 MB. Actual storage and runtime memory also include heads where applicable, tokenizer/configuration files, activation memory, framework overhead, and batching. These estimates are not measured deployment requirements.

### 14.2 HTTP service

The reviewed `laya-serve` exposes **`POST /v1/systemone`** and health reporting. Authentication is optional through `LAYA_API_KEY`. The default bind address is `0.0.0.0`, which listens on all interfaces; deliberately bind locally for a local-only service. Protocol compatibility with Jev does not establish identical predictions or model internals. [HTTP API][http]

Version 0.3.22 also implements **`POST /v1/systemone/batch`**. The inspected HTTP guide still says there is no batch endpoint; this is a documentation mismatch. For that capability, the release's server implementation is the evidence used here. [Released server implementation][serve-code]

Local example after installing the `serve` extra:

```bash
.venv/bin/python -m pip install "laya[serve]==0.3.22"
LAYA_HOST=127.0.0.1 LAYA_MODELS=english LAYA_DEVICE=cpu \
  .venv/bin/laya-serve
```

Then send the same `state` and `questions` shape to `/v1/systemone`. Keep credentials in a protected secret source. For remote clients, place the service behind approved authentication and encrypted transport, with explicit network access rules.

The separate playground example uses `/predict` routes; do not confuse it with the packaged server. Confirm actual endpoint capabilities in the deployed version.

### 14.3 Operational controls

Recommendations:

- Set input, token, concurrency, timeout, and memory limits.
- Warm only needed checkpoints and confirm readiness after their first load.
- Enforce a model allowlist at the application boundary when needed. `LAYA_MODELS` chooses what to preload; it does not by itself prevent requests from selecting another checkpoint.
- Account for checkpoint eviction, worker duplication, and CPU fallback in capacity planning.
- Test cold start, sustained load, overload, and recovery.
- On overload, use bounded retries with backoff and a fixed deadline; do not loop indefinitely.
- Keep inference retries separate from action retries.
- Treat model/configuration failures as operational faults even when an HTTP status looks like a client error.

The server implements request limits and overload responses; accepted request limits are resource protections, not accuracy limits. A request may fit the HTTP option cap while exceeding a useful model option budget. The source reports device/fallback information through health handling. [Serving implementation][serve-code]

### 14.4 Offline use, accelerators, and exports

Provision weights and dependencies before enforcing offline operation. Test a fresh process with network access disabled; a successful warm process does not prove restart readiness.

For GPU, Apple Silicon, ONNX, compilation, or quantized execution, validate the exact runtime. Compare outputs and probability quality against the reviewed baseline. Confirm that the deployment uses the intended accelerator and does not silently meet availability goals by falling back to a much slower device.

### 14.5 Reproducibility and licensing

Record package/dependency versions, immutable checkpoint revisions, schema hash, calibration hash, policy version, runtime settings, and evaluation report. Distinguish the code commit from each model-repository commit.

Apache 2.0 publication allows use under that license's terms; retain required notices and review licenses of dependencies and datasets independently. Public weights and a training notebook do not establish that every historical training datum and training step is reproducible. [License][license]

## 15. Rollout, monitoring, and maintenance

The official adoption guide recommends observation, comparison, evidence-based policy selection, and bounded promotion while the application retains action ownership. [Staged adoption][adoption]

Recommended stages and completion criteria:

| Stage | Behavior | Criterion for progression |
|---|---|---|
| Offline evaluation | Replay representative labeled cases | Quality, cost, and robustness meet predefined gates |
| Shadow | Score real requests without affecting their handling | Adequate live evidence confirms eligible slices and failure paths |
| Advisory | Present recommendations with source evidence | Review shows usefulness and exposes remaining ambiguity |
| Limited automation | Enable a small eligible, reversible slice | Sampled outcomes and guardrails remain acceptable |
| Expanded use | Add separately evaluated slices | Each new slice passes its own acceptance criteria |

For each decision, log a request/event ID; evidence reference; schema, checkpoint, calibration and policy versions; actual route; answer and relevant probabilities; completeness flags; latency; fallback/review reason; and action outcome. Apply redaction and retention limits.

Monitor both input changes and reviewed outcome quality. A rising confidence average does not establish improvement. Track accepted-set errors, critical misses, escalation, truncation, option collapse, route changes, resource failures, and downstream failures.

Predefine rollback conditions. Examples include a confirmed unauthorized action, critical error above its budget, changed artifacts, failed output validation, or an unevaluated language entering automatic handling. Preserve a known-good fallback.

Reevaluate after changes to weights, dependencies, runtime precision, schema, routing, calibration, thresholds, or the traffic population. A schema-only change can invalidate an earlier policy even when weights remain fixed.

## 16. Agent questions and answers

**Can Laya replace a general reasoning agent?**

Use it for evaluated typed decisions. Retain a reasoning agent and tools for planning, evidence gathering, explanations, and cases requiring more than that decision interface.

**Can it generate arbitrary JSON?**

It returns typed decision results and offers a restricted schema projection. Use another method for arbitrary strings, open-ended structures, or exact extraction.

**Does a valid schema guarantee a valid conclusion?**

No. Shape validation checks format. Ground-truth evaluation checks meaning; authorization checks whether an action is permitted.

**Does no text generation mean no hallucinations?**

It avoids invented free-form passages in the native output. It can still select an unsupported or wrong answer. Assess decision error rather than relying on a marketing use of “hallucination.”

**Is `act_probability=1.0` a reason to execute?**

No. It is a model output. Execution requires independent application policy and current prerequisites.

**Should every unknown case be a model label?**

Use a review label when useful, but also enforce unknown/scope handling outside the model. Forced-choice probabilities cannot establish that the supplied categories cover the situation.

**Should I always choose the highest-confidence checkpoint?**

Choose the evaluated checkpoint for the workload. Confidence values from differently calibrated models are not a fair competition for routing.

**Can several answers be treated as independent?**

Assume dependencies may exist. Validate joint constraints. The marginals do not supply a trustworthy joint probability or coherent action plan.

**Can I use it to suppress notifications?**

Evaluate the cost of missed alerts. Keep mandatory critical alerts rule-based, and provide sampling or review of suppressed cases so failures remain observable.

**Can I immediately reuse a published threshold or calibration temperature?**

Treat it as a hypothesis. Fit and validate using the intended data, checkpoint, question shape, and runtime. Published values may have different precedence or be clamped by the current runtime.

**Is self-hosting free?**

There may be no per-call model-service fee, but compute, memory, electricity, maintenance, evaluation, and rework still cost resources. Compare complete workflows.

**Is Laya better than Jev or another model?**

Decide using matched evaluation on your tasks. Published cross-source comparisons are insufficient to rank products universally. This guide verifies Laya; it does not establish current competing-service specifications.

**What should an agent do if it lacks evaluation data?**

Keep results advisory or run in shadow mode, collect reviewed cases, and evaluate before enabling consequential automation.

## 17. Evidence record and sources

### Verification record

Verified September 30, 2026:

- Latest public PyPI release observed: `0.3.22`, uploaded September 29, 2026.
- Repository snapshot inspected: `6d942c92081fbc139e736bbd9ac0023223c29b7f`.
- Published wheel modules checked against that snapshot: `agent.py`, `router.py`, `common.py`, `confidence.py`, `serve.py`, and `structured.py`; all six matched byte for byte.
- Public standalone model revisions and configuration files inspected:

| Repository | Immutable revision |
|---|---|
| `convaiinnovations/laya` | `55cf4c4ebb4ebe31b2550e8bdf3bd21b99753851` |
| `convaiinnovations/laya-multilingual` | `e4e9ddf21a7b1903b7acffd8814ad4307bf63a67` |
| `convaiinnovations/laya-typed-decisions` | `1a793eb568e6718f15941d08f85432581df534e3` |

**Verification limits:** Public interface and configuration verification does not establish runtime compatibility on every platform, checkpoint integrity beyond inspected files, production accuracy, or benchmark reproducibility. Weight inference, training, accelerator performance, and end-to-end integrations were not executed for this document. Source examples were reviewed; the guide's Python example was syntax-checked. Research results are attributed to their authors and should be read with their stated test conditions.

### Source map

- Official project: [repository overview][overview], [PyPI 0.3.22][pypi], [package configuration][package], [license][license].
- Official model cards: [English][english], [multilingual][multilingual], [typed-decisions specialist][typed].
- Immutable configuration evidence: [English][english-config], [multilingual][multilingual-config], [specialist][typed-config].
- Implementation: [sequence/model/confidence definitions][common-code], [inference and calibration][agent-code], [router][router-code], [abstention][confidence-code], [HTTP service][serve-code].
- Official guides: [questions and answers][questions], [structured output][structured], [HTTP API][http], [artifact integrity][security], [evaluation harness][evals], [fine-tuning][finetune], [staged adoption][adoption].
- Reported performance and limitations: [official benchmark report][benchmarks], [negation reproduction][negation], [option-count confidence reproduction][option-gating]. Issue reproductions are primary reports with limited scopes.
- Research primary sources: [calibration and selective-escalation preprint, v1][assessor], [probabilistic-coherence preprint, v1][coherence]. Preprints are reported research evidence, not certification or universal product guarantees.

[overview]: https://github.com/NandhaKishorM/laya
[pypi]: https://pypi.org/project/laya/0.3.22/
[package]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/pyproject.toml
[license]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/LICENSE
[english]: https://huggingface.co/convaiinnovations/laya
[multilingual]: https://huggingface.co/convaiinnovations/laya-multilingual
[typed]: https://huggingface.co/convaiinnovations/laya-typed-decisions
[english-config]: https://huggingface.co/convaiinnovations/laya/blob/55cf4c4ebb4ebe31b2550e8bdf3bd21b99753851/rl_agent_config.json
[multilingual-config]: https://huggingface.co/convaiinnovations/laya-multilingual/blob/e4e9ddf21a7b1903b7acffd8814ad4307bf63a67/rl_agent_config.json
[typed-config]: https://huggingface.co/convaiinnovations/laya-typed-decisions/blob/1a793eb568e6718f15941d08f85432581df534e3/rl_agent_config.json
[common-code]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/laya/common.py
[agent-code]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/laya/agent.py
[router-code]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/laya/router.py
[confidence-code]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/laya/confidence.py
[serve-code]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/laya/serve.py
[questions]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/questions-and-answers.md
[structured]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/structured.md
[http]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/http-api.md
[security]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/security.md
[evals]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/evals.md
[finetune]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/finetune.md
[adoption]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/docs/staged-adoption.md
[benchmarks]: https://github.com/NandhaKishorM/laya/blob/6d942c92081fbc139e736bbd9ac0023223c29b7f/BENCHMARKS.md
[negation]: https://github.com/NandhaKishorM/laya/issues/377
[option-gating]: https://github.com/NandhaKishorM/laya/issues/394
[assessor]: https://arxiv.org/abs/2609.33843v1
[coherence]: https://arxiv.org/abs/2609.33971v1
