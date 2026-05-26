# The Big Prompt & Best of N: Two Techniques for Serious AI-Assisted Development

*From the Vanderbilt University Coursera course: Claude Code: Software Engineering with Generative AI Agents — with notes on why these techniques matter in practice.*

---

Most developers interact with AI assistants the way they interact with a search engine: a short query, a quick scan of the result, on to the next thing. That approach works for simple lookups. It consistently underperforms for complex engineering work.

The two techniques covered here — the **Big Prompt** and **Best of N** — come directly from the Vanderbilt University Coursera course *Claude Code: Software Engineering with Generative AI Agents*. Both require more upfront investment than a casual prompt. Both produce outputs meaningfully closer to production-quality work.

---

## The Big Prompt

The **Big Prompt** is a comprehensive, carefully constructed system prompt written before any task begins. Rather than building context incrementally through conversation, you front-load everything the model needs to operate with genuine understanding of your project: the stack, the architecture, the conventions, the constraints, and the specific problem at hand.

Claude has no inherent knowledge of your codebase, your team's decisions, or the tradeoffs that shaped your current architecture. The Big Prompt closes that gap — not by piecing together context mid-conversation, but by building a reusable context document that establishes the full working environment upfront. Think of it less as a prompt and more as a project brief. The difference between "build me a streaming response component" and a fully-specified system context is not marginal — it is the difference between generic scaffolding and code that fits your actual system.

### A Concrete Example

Here is a Big Prompt for **Liminal** — a therapeutic AI journaling application built on the Anthropic API. Liminal helps users process complex emotional content, track psychological patterns across entries, and receive research-informed reflections. It requires careful handling of sensitive user content, nuanced AI behavior, and strong privacy practices baked into the architecture from the start.

```
You are a senior full-stack engineer and applied AI specialist working on 
Liminal, a therapeutic journaling application built on the Anthropic API.
The product helps users process emotional and psychological content through
structured, AI-assisted reflection. It is a consumer app where trust, 
privacy, and psychological safety are product-critical — not afterthoughts.

Tech stack:
- Next.js 14 (App Router) with TypeScript in strict mode
- Anthropic Claude API (claude-sonnet-4-20250514) via the official Node.js SDK
- PostgreSQL via Prisma ORM
  Schema: Users, JournalEntries, ReflectionThreads, EmotionalPatterns, 
  UserPreferences, SafetyEvents
- Tailwind CSS + shadcn/ui for all interface components
- Zod for all request/response validation at API boundaries
- NextAuth.js with Google OAuth and magic link fallback
- Vercel AI SDK for streaming Claude responses to the client

Architecture decisions (already made — do not relitigate):
- All Claude API calls route through /lib/anthropic/client.ts
  Never call the Anthropic SDK directly from components or API routes
- System prompts live in /lib/anthropic/prompts/ as typed template functions
  Each prompt function accepts a typed context object and returns a string
- Journal entries are encrypted at rest with AES-256 before database writes
- Emotional pattern analysis runs as a background job (queue-based, 
  not blocking) — results are never surfaced in the same session that 
  triggered them
- The reflection thread model: each journal entry can spawn a Claude-guided 
  follow-up thread; threads are capped at 8 exchanges to prevent dependency loops
- Safety detection runs before every Claude response via a pre-flight classifier
  If triggered, control passes to SafetyResponseHandler — never to inline Claude output

Code conventions:
- Functional React components only; complex client state managed with Zustand
- API routes return { data: T | null, error: string | null } envelopes
  Route handlers never throw — all errors are caught and shaped into this envelope
- All database access goes through /lib/db/ service files
  Never use Prisma client directly from components, pages, or API routes
- Streaming responses use the Vercel AI SDK useChat hook
- Privacy requirements: no user content in console.log or error messages
  Error messages exposed to the client are sanitized — stack traces never leave the server
- All Claude response metadata (token usage, latency, model version) is logged 
  to a separate analytics table, never mixed with user content

Claude behavior guidelines for Liminal:
- Reflections are warm, curious, and non-directive
  The model asks questions more than it offers interpretations
  It does not give advice unless explicitly asked
- Pattern observations across entries are named gently and framed as 
  observations, not conclusions ("I notice..." not "You tend to...")
- Crisis language or self-harm indicators route immediately to SafetyResponseHandler
  The model never attempts to handle these inline
- All Claude outputs include a structured metadata block appended after the 
  visible response, used by the pattern analysis background job

Current task: Build the ReflectionThread component. It should stream a 
Claude-generated reflection in response to a new journal entry, render the 
output with Markdown support, and allow the user to continue the thread with 
follow-up entries. Implement loading, error, and empty states. Use the 
existing /lib/anthropic/client.ts, follow all conventions above, and ensure 
the safety pre-flight check runs before any Claude output is displayed.
```

This is a prompt Claude can do real work with. It knows the architecture, the data model, the behavioral constraints around sensitive content, and the exact task. The output is code that fits the actual system — not a plausible approximation of what such code might look like.

---

## Best of N

**Best of N** is the practice of generating multiple distinct implementations of the same feature or design problem, evaluating them against each other, and either selecting the best or synthesizing across them. Rather than treating the first output as final, you treat it as one sample from a space of possible good answers and sample that space deliberately.

The same prompt, run multiple times, produces meaningfully different outputs. One run might surface a cleaner architectural separation; another might handle an edge case the first missed; a third might surface a tradeoff the first two papered over. The variance isn't noise — it's the model drawing on a genuinely broad solution space each time.

What makes this practical rather than theoretical comes down to four properties of working with AI:

**Cheap.** Running three to five variations of a component or architecture costs a fraction of what parallel human exploration would cost. The economic barrier to systematic design exploration has effectively disappeared.

**Fast.** Multiple generations complete in minutes, not days. Best of N exploration that would previously require sprint-level planning can happen in an afternoon.

**Creative.** Claude doesn't anchor on its first answer. It draws on a wide range of approaches and tradeoffs across generations, often producing a third option that reframes the problem in a way neither of the first two did. This isn't stochasticity as a bug — it's a genuine source of design insight.

**Egoless.** Claude has no investment in any particular solution. It will not defend a previous answer or resist when you select a different approach, and will evaluate its own outputs critically when asked. You can generate five implementations, ask Claude to score them, and discard four — and it will do exactly that, without friction. Human designers bring expertise and attachment. Claude brings expertise without the attachment.

### Best of N in Practice: Git Branches as Variant Containers

The most disciplined way to run Best of N on a real codebase is to give each variant its own git branch — keeping implementations isolated, allowing each to reach a meaningful level of completeness before evaluation, and preserving all variants until you've made a deliberate choice.

Here is the high-level process:

1. **Start from a clean base.** Ensure your working branch is up to date and committed.
2. **Create N branches** — one per variant — before generating any code. Naming convention: `experiment/<feature>-v1`, `experiment/<feature>-v2`, etc.
3. **Generate each implementation** on its designated branch, switching between variants so implementations stay isolated.
4. **Build each variant to a meaningful level** — not just a skeleton, but enough to evaluate real tradeoffs: component structure, data flow, error handling, and at least one test.
5. **Run a structured cross-branch analysis** with Claude before making any comparative judgment.
6. **Evaluate across branches** and merge the winner — or the best synthesis — into your main working branch.

The example below uses **Axiom**, an agentic SaaS platform for hard science fiction writers that grounds creative work in real scientific research. The system retrieves papers from ArXiv, synthesizes peer-reviewed literature, and helps writers build technically credible worlds — treating Kim Stanley Robinson's research rigor as a product requirement, not a stylistic preference.

The feature being explored is Axiom's `ResearchSynthesisAgent` — the core agent that takes a writer's creative premise and returns a structured scientific context document.

---

**Step 1: Create the branches before writing any code.**

```bash
# From your base branch (e.g., main or develop)
git checkout -b experiment/synthesis-agent-v1
git checkout main
git checkout -b experiment/synthesis-agent-v2
git checkout main
git checkout -b experiment/synthesis-agent-v3
git checkout main

# Begin work on variant 1
git checkout experiment/synthesis-agent-v1
```

---

**Step 2: Generate and implement variant 1.**

```
You are working on Axiom, an agentic research platform for hard science 
fiction writers. The system retrieves scientific papers from ArXiv and 
synthesizes them into structured world-building material — technically 
accurate, narratively useful, and honest about the limits of current science.

You are implementing VARIANT 1 of the ResearchSynthesisAgent on branch 
experiment/synthesis-agent-v1.

VARIANT 1 ARCHITECTURE: Single-pass synthesis with explicit uncertainty mapping.

Design and implement a ResearchSynthesisAgent that:
- Accepts a NarrativePremise object: { premise: string, scientificDomains: 
  string[], accuracyRequirement: 'hard' | 'soft' | 'speculative' }
- Performs a single structured pass: retrieve relevant ArXiv abstracts via 
  tool call, score each for relevance to the narrative premise (not just 
  topical similarity), synthesize into a ScienceContextDocument
- The ScienceContextDocument has three explicit sections:
  EstablishedScience (high-confidence findings, >3 corroborating papers),
  ActiveFrontier (contested or recent findings, flagged with uncertainty),
  SpeculativeSpace (extrapolations clearly marked as authorial, not scientific)
- Handles conflicting findings by surfacing the disagreement explicitly 
  rather than resolving it — writers need to know where science is unsettled
- Includes the ArXiv paper IDs used in synthesis so outputs are auditable

Implement the agent class, its tool definitions, the output schema (use Zod),
and a unit test that verifies the three-section structure is always present 
even when source papers are sparse.

This is Variant 1. Focus on this architecture specifically — do not blend 
approaches from other possible designs.
```

---

**Step 3: Commit variant 1, switch to variant 2.**

```bash
git add -A
git commit -m "experiment: synthesis-agent-v1 — single-pass with uncertainty mapping"
git checkout experiment/synthesis-agent-v2
```

---

**Step 4: Generate variant 2 on its branch.**

```
You are working on Axiom. Same product context as before.

You are implementing VARIANT 2 of the ResearchSynthesisAgent on branch 
experiment/synthesis-agent-v2.

VARIANT 2 ARCHITECTURE: Multi-step chain-of-thought with adversarial review.

Design and implement a ResearchSynthesisAgent that:
- Accepts the same NarrativePremise input schema as Variant 1
- Operates in three explicit reasoning steps:
  Step 1 — Domain Scoping: Given the premise, identify which scientific 
    subfields are most relevant and generate targeted ArXiv queries
  Step 2 — Synthesis: Build a draft ScienceContextDocument from retrieved papers
  Step 3 — Adversarial Review: The agent critiques its own draft, asking:
    "What would a domain expert object to here? What claims are overstated? 
     What uncertainty did I flatten?" Then revises accordingly
- The adversarial review step must produce a visible revision log — 
  writers can see what the agent changed and why
- Outputs the same three-section ScienceContextDocument structure as Variant 1 
  for comparability

Implement the agent class with explicit step boundaries, the revision log 
schema, and a test that verifies the adversarial step produces at least 
one revision when given a premise that contains a scientifically contested claim.
```

---

**Step 5: Commit variant 2, switch to variant 3.**

```bash
git add -A
git commit -m "experiment: synthesis-agent-v2 — multi-step with adversarial review"
git checkout experiment/synthesis-agent-v3
```

---

**Step 6: Generate variant 3 on its branch.**

```
You are working on Axiom. Same product context as before.

You are implementing VARIANT 3 of the ResearchSynthesisAgent on branch 
experiment/synthesis-agent-v3.

VARIANT 3 ARCHITECTURE: Specialist sub-agent ensemble.

Design and implement a ResearchSynthesisAgent that:
- Accepts the same NarrativePremise input schema
- Decomposes the scientific domains in the premise and spawns one 
  specialist sub-agent per domain (e.g., one for astrophysics, one for 
  materials science, one for biology if the premise spans multiple fields)
- Each specialist sub-agent independently retrieves papers and produces 
  a domain-scoped summary
- An orchestrating agent synthesizes the specialist outputs into the 
  final ScienceContextDocument, explicitly noting where findings from 
  different domains interact, conflict, or create opportunities for 
  speculative narrative
- Designed for writers working at the intersection of multiple fields — 
  the primary use case where existing tools fall flat

Implement the orchestrator, the specialist sub-agent template (generic, 
domain-injected at runtime), the synthesis logic, and a test that verifies 
correct handling of a premise that spans two distinct scientific domains.
```

---

**Step 7: Commit variant 3.**

```bash
git add -A
git commit -m "experiment: synthesis-agent-v3 — specialist sub-agent ensemble"
```

**Step 8: Systematic cross-branch code review.**

Before asking Claude to make a recommendation, have it conduct a structured analysis of each branch in turn. The goal is not yet a verdict — it is documentation. You want Claude to examine each implementation on its own terms: not just what it does, but how it accomplishes its goals and what tradeoffs were embedded in those decisions.

Switch to each branch and run an analysis prompt:

```bash
git checkout experiment/synthesis-agent-v1
```

```
You are reviewing the ResearchSynthesisAgent implementation on branch 
experiment/synthesis-agent-v1. Analyze this implementation and produce 
a structured technical summary covering:

1. Architecture — What is the agent's core reasoning structure? 
   How does it move from input to output?
2. Code organization — How are responsibilities divided across files 
   and classes? Is the separation of concerns clean?
3. Libraries and dependencies — What does this implementation rely on 
   beyond the project's standard stack, and why?
4. Implementation patterns — What specific patterns or techniques are 
   notable? What does this approach do that the others likely do not?
5. Tradeoffs made — What did this architecture optimize for, and what 
   did it deprioritize as a result?
6. Risk surface — Where is this implementation most likely to fail 
   or behave unexpectedly under real usage?

Be specific. Reference actual class names, function signatures, and 
file paths. This analysis will be compared directly against the same 
analysis for two other implementations, so precision matters more 
than summary.
```

Commit the analysis output to an `ANALYSIS.md` on each branch, then repeat for the remaining variants:

```bash
# After reviewing v1
echo "[Claude's analysis output]" > ANALYSIS.md
git add ANALYSIS.md
git commit -m "experiment: synthesis-agent-v1 — add technical analysis"

git checkout experiment/synthesis-agent-v2
# Run analysis prompt, then:
echo "[Claude's analysis output]" > ANALYSIS.md
git add ANALYSIS.md
git commit -m "experiment: synthesis-agent-v2 — add technical analysis"

git checkout experiment/synthesis-agent-v3
# Run analysis prompt, then:
echo "[Claude's analysis output]" > ANALYSIS.md
git add ANALYSIS.md
git commit -m "experiment: synthesis-agent-v3 — add technical analysis"
```

This produces a durable record of what each approach actually does, written when the code is in front of you and the reasoning is fresh.

---

**Step 9: Ask Claude to evaluate across variants.**

With all three implementations analyzed and documented, you can ask Claude to compare them:

```
Review the three ResearchSynthesisAgent implementations across these branches:
- experiment/synthesis-agent-v1 (single-pass, uncertainty mapping)
- experiment/synthesis-agent-v2 (multi-step, adversarial review)
- experiment/synthesis-agent-v3 (specialist sub-agent ensemble)

Evaluate each on:
1. Output accuracy risk — which is most likely to produce confident but 
   wrong scientific claims?
2. Latency and cost per synthesis run — which is most viable at scale?
3. Transparency to users — which gives writers the most insight into 
   where the output came from and what its limits are?
4. Implementation complexity and maintenance burden

Axiom's users are serious writers with genuine scientific literacy who 
will fact-check outputs. Recommend which variant to ship for MVP and 
which, if any, is worth building toward in a future version.
```

Claude's evaluation across a structured set of alternatives is usually substantive — it will surface tradeoffs you may not have weighted correctly and make a recommendation with explicit reasoning. Importantly, the recommendation does not have to be "pick one." Claude will often identify that two variants solve different parts of the problem well and suggest a synthesis: take the adversarial review loop from V2, the specialist decomposition from V3, and keep V1 as a lighter-weight fallback. That kind of synthesis is harder to reach without the structured per-branch analysis that precedes it.

---

## Using Both Together

The Big Prompt and Best of N operate at different layers. The Big Prompt establishes the environment — the stack, the conventions, the constraints — that gives Claude what it needs to produce relevant work. Best of N explores multiple paths within that environment: different architectures, different tradeoffs, different solutions given shared context. Together they produce an AI collaborator that understands your system deeply and thinks about your problems from multiple angles before converging.

---

## What This Requires of You

Both techniques ask more of you upfront. The Big Prompt requires you to articulate your project's context clearly enough to transfer it. Best of N requires you to frame a problem as an explicit design space rather than a single question.

That investment is also a forcing function. Writing a Big Prompt surfaces ambiguities in your own thinking. Structuring a Best of N prompt forces you to identify what tradeoffs actually matter before you see any code.

Claude is a genuinely capable technical collaborator when given the context and structure to operate well. These techniques are how you provide that.

---

*Based on the Vanderbilt University Coursera course: [Claude Code: Software Engineering with Generative AI Agents](https://www.coursera.org/learn/claude-code/home/welcome)*
