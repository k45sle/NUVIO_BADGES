## Best single fit: **Kev-4B**

If the priorities are:

1. **reduce expensive LLM token usage as much as practical**
2. **retain enough capability that Hermes does not constantly fall back to a frontier model**
3. **run locally**
4. **fit naturally into Hermes Agent**

then I would start with **Kev-4B**, not Laya and not Jev.

### Why Kev-4B

The key metric for your use case isn't model size. It's:

> **How many requests can this model confidently intercept before Hermes has to call the expensive generative model?**

Kev-4B currently looks like the best balance.

Its latest model reports **0.817 out-of-domain development accuracy / 0.838 locked-test accuracy**, and on its reported calibration test it could automatically handle about **62% of decisions while keeping the accepted subset at ≤5% error**. Jev reports higher coverage at about 70%, but Jev is a hosted API, while Kev runs locally. :chatgpt-content-reference{index="0"}

That distinction matters for your goal:

```text
Jev
request → cloud → input tokens billed → decision

Kev-4B
request → your hardware → no API-token bill → decision
```

Laya is dramatically smaller and faster, but its own documentation explicitly describes the base checkpoints as **a fast base to specialize rather than a strong zero-shot decision engine**. On Laya's typed-decision benchmark, the base English checkpoint scored 0.362 versus 0.766 after task-specific fine-tuning. :chatgpt-content-reference{index="1"}

So without building and maintaining a custom Laya fine-tune, I'd expect more:

```text
Laya → uncertain/wrong → fallback to main LLM
```

which can erase the theoretical savings from having a tiny 421M model.

---

## How I'd use Kev inside Hermes

**Do not make Kev Hermes's main model.**

Kev isn't a generative model. Instead, put it **in front of expensive decisions**.

Your Hermes architecture should look roughly like this:

```text
                     USER / EVENT
                          │
                          ▼
                Deterministic checks
                  (zero AI tokens)
                          │
                          ▼
                       Kev-4B
                  LOCAL DECISION GATE
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
    No LLM needed     Cheap model       Frontier model
     / direct tool     / worker          GPT / Claude /
        action                            strong model
```

Kev should answer questions such as:

```text
Does this require reasoning?
YES / NO

Which model tier is appropriate?
- deterministic
- cheap
- standard
- frontier

Does this need a subagent?
YES / NO

Which tool family is relevant?
- filesystem
- web
- GitHub
- email
- none

Does this require user approval?
YES / NO / REVIEW

Is this request routine enough for the cheap worker?
YES / NO

Does this result require another reasoning iteration?
YES / NO
```

That is exactly the type of work these decision models are designed for.

---

# Where this could save Hermes the most tokens

Hermes already has a lot of places where model calls occur.

Its current architecture distinguishes the **main model** from auxiliary models used for compression, vision, web extraction, approval, MCP routing, skill search, and other side jobs. :chatgpt-content-reference{index="2"}

It also supports subagents, and the Hermes documentation specifically notes that **child agents often consume the majority of total tokens in a delegated run**. :chatgpt-content-reference{index="3"}

So I'd attack token usage in this order.

### 1. Subagent creation

This is potentially the largest target.

Currently the main model has to reason about whether delegation is appropriate.

Kev can provide a cheap first opinion:

```text
Task:
"Rename these 70 files according to this pattern."

Kev:
requires_reasoning = 0.04
requires_subagent = 0.02

→ execute_code
```

versus:

```text
Task:
"Investigate why this distributed service
occasionally loses writes."

Kev:
requires_reasoning = 0.97
requires_subagent = 0.88

→ strong agent
```

Hermes itself recommends `execute_code` for mechanical workflows because it avoids the full LLM reasoning loop. :chatgpt-content-reference{index="4"}

That's a perfect place for a decision gate.

---

### 2. Model-tier selection

This might be the most valuable Kev integration.

Instead of:

```text
everything
    ↓
GPT-5.6 / Claude / expensive model
```

make Kev choose:

```text
                Kev
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
    cheap      normal     frontier
    model       model       model
```

You might define:

```text
A = deterministic/tool-only
B = cheap fast model
C = capable general model
D = frontier reasoning model
```

Kev gives a probability distribution:

```json
{
  "A": 0.03,
  "B": 0.71,
  "C": 0.23,
  "D": 0.03
}
```

Hermes sends it to the cheap worker.

No frontier call.

---

### 3. Tool routing

Hermes currently has **70+ registered tools across roughly 28 toolsets**, so tool-selection context itself can become substantial. :chatgpt-content-reference{index="5"}

You don't necessarily want your expensive model reasoning over every available tool description.

Kev can first decide:

```text
Relevant toolsets:

filesystem
github
web
email
calendar
memory
none
```

Then Hermes exposes only the relevant group to the real agent.

That potentially produces a **double token saving**:

```text
fewer LLM calls
+
smaller prompts when an LLM is called
```

That's particularly attractive.

---

### 4. Skill selection

Hermes's skills system already uses progressive disclosure specifically to reduce token consumption. :chatgpt-content-reference{index="6"}

Kev could strengthen that model:

```text
2,000 available skills
        ↓
retrieval shortlist
        ↓
10 possible skills
        ↓
Kev
        ↓
1–3 relevant skills
        ↓
main LLM sees only those
```

I would **not** give Kev hundreds of options directly.

Kev can handle more complex decisions than Laya, but hierarchical routing is still preferable.

---

### 5. Smart approval

Hermes already has a dedicated auxiliary slot for smart approval and recommends using a cheap model because using an expensive reasoning model there is wasteful. :chatgpt-content-reference{index="7"}

This is almost tailor-made for Kev:

```text
command + context
      ↓
Kev

safe_low_risk       0.97
needs_user_approval 0.02
deny                0.01
```

But deterministic security policy should remain outside Kev.

For example:

```text
rm -rf /
```

shouldn't become acceptable because Kev assigns it 99% confidence.

---

# What Kev should **not** replace

This distinction is important.

Kev cannot replace Hermes auxiliary jobs that **generate content**.

So don't use Kev for:

| Hermes job | Kev? |
|---|---:|
| Model routing | **Yes** |
| Tool routing | **Yes** |
| Skill selection | **Yes** |
| Approval classification | **Yes** |
| Delegation decision | **Yes** |
| Notification importance | **Yes** |
| "Continue vs stop" | **Yes** |
| Context compression | ❌ |
| Web-page summarization | ❌ |
| Writing subagent results | ❌ |
| Coding | ❌ |
| Research synthesis | ❌ |
| Main conversation | ❌ |

Hermes's compression system, for example, explicitly requires a small **generative** model to create summaries. :chatgpt-content-reference{index="8"}

So you'd still want a cheap generative model for those jobs.

---

# Why not Laya?

Laya has one major advantage:

## It's tiny.

```text
Laya
322M–421M

Kev-4B
~4.7B base
```

And Laya reports roughly **33–40 ms** single-question latency on a T4, with much higher batched throughput. :chatgpt-content-reference{index="9"}

If we trained Laya **specifically on your Hermes decisions**, it could eventually become the better Tier-0 router.

For example, collect 20,000 examples of:

```text
request → toolset
request → cheap/frontier
request → delegate/no-delegate
request → approval/no-approval
```

Fine-tune Laya on those exact decisions.

Then:

```text
               Laya
              ~400M
                │
        90% obvious cases
                │
                ▼
              done
```

could be extremely efficient.

But **today**, without that custom dataset, I would trust Kev-4B with a broader variety of zero-shot semantic decisions.

Laya itself publishes some significant limitations, including high-cardinality choices and examples where negation produced confidently wrong answers. :chatgpt-content-reference{index="10"}

For an autonomous agent, those edge cases matter.

---

# Why not Jev?

Jev is attractive if you care only about capability and operational simplicity.

Based on the public comparisons, it generally remains stronger and better calibrated across difficult heterogeneous workloads. Kev's own evaluation shows Jev ahead on several outside test sets. :chatgpt-content-reference{index="11"}

But your priority is specifically **reducing token usage**.

A hosted decision API still means:

```text
Hermes
  ↓
internet
  ↓
TypeSafe
  ↓
metered token processing
```

Kev gives you:

```text
Hermes
  ↓
localhost
  ↓
Kev
```

I'd use Jev as a **benchmark**, not the everyday first-line router.

For example:

```text
             Kev-4B
                │
        confidence high
          │          │
         yes         no
          │          │
          ▼          ▼
        use it    main LLM
                     │
          optionally Jev during
           evaluation/shadowing
```

---

# What about Kev-9B?

Kev-9B gives you somewhat more capability, but **not enough improvement to justify more than doubling the model size for this role**.

Current project results:

| Model | New-source dev | Locked test |
|---|---:|---:|
| Kev-4B | 0.817 | **0.838** |
| Kev-9B | 0.822 | **0.852** | :chatgpt-content-reference{index="12"}


That's only a modest improvement.

Meanwhile:

```text
Kev-4B → ~12 GB GPU class
Kev-9B → ~24 GB GPU class
``` :chatgpt-content-reference{index="13"}


For a routing/gating layer, **4B is a much more sensible point on the curve**.

Save your compute for the actual reasoning model.

---

# The setup I'd choose for you

Not:

```text
Hermes → Kev for everything
```

Instead:

```text
                    HERMES
                       │
                       ▼
              ┌────────────────┐
              │ deterministic  │
              │ rules / code   │
              └───────┬────────┘
                      │ unresolved
                      ▼
                ┌───────────┐
                │  Kev-4B   │
                │ LOCAL GATE│
                └─────┬─────┘
                      │
       ┌──────────────┼───────────────┐
       │              │               │
       ▼              ▼               ▼
 execute_code    cheap generative   main capable
 / direct tool      worker             agent
       │              │               │
       │              │          genuinely hard
       │              │               │
       └──────────────┴───────────────▼
                               frontier model
```

And for Hermes itself:

```yaml
Main agent:
  capable model

Subagents:
  cheaper generative model

Compression:
  tiny fast generative model

Web extraction:
  tiny fast generative model

Decision routing:
  Kev-4B LOCAL

Skill routing:
  Kev-4B LOCAL

Toolset routing:
  Kev-4B LOCAL

Approval classification:
  Kev-4B LOCAL
```

Hermes already supports independent auxiliary-model slots and separate worker models, so this architecture matches the direction the project already takes. :chatgpt-content-reference{index="14"}

## My recommendation

**Start with Kev-4B.**

Then, once Hermes has accumulated enough actual routing history, train **Laya on your own Hermes decisions** and test whether it can replace Kev for the easiest Tier-0 decisions.

That gives you a longer-term architecture like:

```text
Deterministic
     ↓
Laya ~400M
     ↓ uncertain
Kev-4B
     ↓ uncertain / requires generation
cheap generative model
     ↓ genuinely complex
frontier model
```

That is the configuration I'd expect to maximize **actual avoided frontier-model tokens** while preserving Hermes's capabilities, rather than simply choosing the smallest model on paper.

Yes — **Kev-4B is probably too heavy for a typical home server unless you have a fairly capable GPU or a 32 GB Apple Silicon Mac**.

Current Kev-4B requirements are roughly:

| Resource | Kev-4B |
|---|---:|
| Model weights | ~9 GB VRAM in bf16 |
| Serving with batching | ~14 GB VRAM |
| Recommended Mac | **32 GB unified memory** |
| Recommended NVIDIA | roughly **16 GB+ VRAM** to be comfortable |
| CPU-only | Technically possible, but not a good fit for a low-latency routing layer |
| Python | 3.12 or 3.13 |
| Apple Silicon | Supported via MLX |

The project's current model card says Kev-4B needs about **9 GB for the weights and ~14 GB with its serving buffers**. On Apple Silicon, it reports about **721 ms for five questions on a new ~270-token input** on an M5, versus tens of milliseconds on a suitable CUDA GPU. 

So if your server has something like:

```text
8 GB RAM
16 GB RAM
integrated graphics
older Intel CPU
no discrete NVIDIA GPU
```

I would **not** choose Kev-4B as your always-on Hermes router.

## Better options for you

The practical choices become:

### **1. Laya — most likely the best local fit**

This changes my recommendation if hardware is constrained.

Laya's standard models are only:

```text
322M–421M parameters
```

versus Kev-4B's roughly:

```text
4 billion parameters
```

That is about an **order of magnitude smaller**.

Laya is therefore much more realistic as an always-running service on modest hardware.

For a Hermes gatekeeper, you'd use it only for very bounded questions:

```text
Does this require a full LLM?
yes / no

Which class of task is this?
- code
- web
- filesystem
- email
- general

Should Hermes delegate?
yes / no

Which model tier?
- cheap
- normal
- frontier
```

Then anything Laya isn't confident about simply goes to Hermes normally.

That can still save substantial tokens without requiring a large GPU.

---

### **2. Kev-0.8B — middle ground**

Kev has a much smaller model:

**Kev-0.8B**

The project explicitly lists this as the model intended when **size matters more than accuracy**, and says it can run on **any Apple Silicon Mac** or an NVIDIA L4-class GPU. 

It's considerably lighter than Kev-4B:

```text
Laya        ~0.4B
Kev-0.8B    ~0.8B
Kev-4B      ~4B
```

Capability currently follows approximately the same pattern:

```text
Laya
  ↓
Kev-0.8B
  ↓
Kev-4B
  ↓
Kev-9B
  ↓
Jev
```

That is not a strict universal quality ordering, but it's a useful mental model for general zero-shot decision capability.

Kev-0.8B's reported out-of-domain result is materially lower than Kev-4B:

```text
Kev-0.8B:  ~0.697 locked test
Kev-4B:    ~0.838 locked test
```

So you'd expect more fallback to the real LLM.

But that may still be worthwhile if it runs comfortably on your machine.

---

## The important thing: **you don't need Kev at all**

For Hermes, the most efficient architecture may actually be:

```text
             Hermes request
                   │
                   ▼
          deterministic rules
                   │
                   ▼
                Laya
            ~400M local
                   │
           ┌───────┴────────┐
           │                │
      high confidence    uncertain
           │                │
           ▼                ▼
    cheap/direct path    Hermes LLM
```

You don't need:

```text
Laya → Kev → GPT
```

unless you have enough hardware to justify running multiple decision models.

Every additional model also adds:

- RAM use
- startup complexity
- model updates
- monitoring
- latency
- failure modes

For your stated goal, I would keep it simpler.

## What I would choose under different hardware

| Your hardware | Best starting point |
|---|---|
| 8 GB RAM, no GPU | **Laya** |
| 16 GB RAM, no GPU | **Laya** |
| Small Intel/AMD mini-PC | **Laya** |
| Raspberry Pi / very weak system | deterministic rules first; Laya may still be too much depending on configuration |
| 16 GB Apple Silicon Mac | **Laya or Kev-0.8B** |
| 32 GB Apple Silicon Mac | **Kev-4B becomes realistic** |
| NVIDIA 8 GB GPU | **Laya / possibly Kev-0.8B** |
| NVIDIA 12 GB GPU | Laya / Kev-0.8B; Kev-4B may be tight |
| NVIDIA 16 GB+ GPU | **Kev-4B becomes reasonable** |
| NVIDIA 24 GB GPU | Kev-4B or Kev-9B |

And there's one other option that's probably worth considering for you:

> **run Kev remotely only when needed.**

Kev includes a deployment path using Modal, where the GPU service can scale to zero when idle. Then Hermes could use:

```text
Laya locally
   ↓ uncertain
Kev-4B remotely
   ↓ still uncertain
main LLM
```

But at that point you're adding cloud compute cost and another network dependency, so I would first test whether **Laya alone intercepts enough Hermes traffic** to make the extra layer worthwhile.

### Revised recommendation

Given your hardware concern, I would change the starting recommendation to:

> **Use Laya as Hermes's local decision gate. Keep your current generative model as the fallback.**

Only introduce Kev later if real usage data shows that Laya rejects/escalates too many requests.

If you tell me **what machine Hermes is actually running on — CPU, RAM, and GPU if any — I can tell you exactly which Laya/Kev models are realistic on it and roughly how much memory they'll consume.**
