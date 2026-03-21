---
name: deep-research
description: Conduct systematic, multi-source technical research to evaluate technology choices, compare architectural options, or investigate domain-specific engineering decisions. Trigger this skill when the user asks to "research", "evaluate", "compare", or "investigate" a technology, framework, library, architecture, protocol, or technical approach — especially before making a consequential decision. Also trigger when the user asks "what's the current state of X", "should we use X or Y", or "deep dive into Z".
version: 1.0.0
---

# Deep Research

Systematic technical research that produces defensible, evidence-backed conclusions — not summaries of blog posts.

## When to Activate

Trigger this skill when the request involves:

- Evaluating whether to adopt a technology, library, or framework
- Comparing two or more technical approaches with real tradeoffs
- Understanding the current state of a fast-moving technical domain
- Investigating a problem that requires synthesizing multiple authoritative sources
- Due diligence before a significant architectural decision
- Understanding production behavior, known failure modes, or performance characteristics

**Good research questions (specific, bounded, answerable):**
- "Should we use Kafka or Pulsar for our event streaming pipeline given <constraints>?"
- "What are the real-world performance characteristics of Bun vs Node.js for I/O-heavy workloads?"
- "What is the current state of WebAssembly support in browsers as of 2026?"
- "What are the known failure modes of distributed sagas vs 2PC for financial transactions?"
- "How does React Server Components change the mental model for state management?"

**Bad research questions (vague, too broad, or answerable by recall alone):**
- "Tell me about databases" — too broad; define the decision context first
- "Is Python good?" — not a research question; requires a specific comparison axis
- "Explain microservices" — educational, not research; skip this skill
- "What is GraphQL?" — definitional; answer from knowledge, no research needed

## Source Hierarchy

Evaluate sources in this priority order. Higher-priority sources override lower-priority ones when they conflict.

### Tier 1 — Authoritative Primary Sources
- Official specifications (RFCs, W3C specs, ECMA standards, ISO standards)
- Official documentation of the technology itself (not tutorials)
- Peer-reviewed papers (ACM, IEEE, arXiv with institution affiliation)
- Official changelogs, release notes, and migration guides
- Primary maintainer blog posts on the official project site

### Tier 2 — High-Quality Secondary Sources
- Engineering blogs from organizations at scale (Google, Meta, Netflix, Cloudflare, Stripe, etc.)
- Postmortems and incident reports (real failure data is gold)
- Benchmark studies with disclosed methodology and reproducible setups
- Conference talks from domain experts (QCon, StrangeLoop, USENIX, CppCon, etc.)
- GitHub issues and PR discussions from the official repo (captures real pain points)

### Tier 3 — Useful but Verify
- High-quality developer blogs (check author credentials and date)
- Stack Overflow answers (accepted + highly upvoted + recent)
- Community RFCs and proposals (shows direction, not current reality)
- Vendor comparison pages (useful for feature lists; distrust claims)

### Tier 4 — Low Trust; Use Sparingly
- Medium/Dev.to posts (high variance; check author background)
- Tutorial sites (Baeldung, DigitalOcean, etc.) — good for API surface, not tradeoffs
- Reddit/HN discussions — useful for gathering anecdotes, not conclusions
- Marketing content, sponsored posts, press releases

## Research Workflow

### Step 1: Sharpen the Research Question

Before searching anything, restate the research question in this form:

```
Context: <what system/team/constraints are we working in>
Question: <the specific technical question>
Decision: <what will this research inform>
Success criteria: <what does a good answer look like>
Non-goals: <what this research does NOT need to cover>
```

If the user's request is vague, ask one clarifying question — the most important missing constraint. Do not ask multiple questions upfront; it stalls the work.

### Step 2: Identify the Research Axes

Break the question into 3–6 independent evaluation dimensions. Example for "Kafka vs Pulsar":

1. Throughput and latency at target message volume
2. Operational complexity (deployment, tuning, monitoring)
3. Feature completeness (multi-tenancy, geo-replication, schema registry)
4. Community health and long-term support trajectory
5. Migration cost from current stack
6. Known production failure modes and recovery patterns

Each axis will drive a targeted search, not a general one.

### Step 3: Execute Targeted Searches

For each axis, construct precise search queries. Use `WebSearch` if available; otherwise `WebFetch` on specific high-value URLs.

**Search query construction rules:**
- Include the technology name + version year where relevant: `"Kafka 3.x" OR "Kafka 2026"`
- Target specific source types: `site:engineering.atscale.com`, `site:github.com`, `site:arxiv.org`
- Use negative terms to filter noise: `-tutorial -"getting started" -"hello world"`
- Search for failure evidence, not just success stories: `"Kafka production issues"`, `"Pulsar limitations"`
- Search for official migration guides when evaluating adoption cost

**Minimum source targets per axis:**
- 1 official documentation page
- 1 engineering case study or postmortem
- 1 benchmark or quantitative comparison (when applicable)

### Step 4: Fetch and Read Key Sources in Full

Do not rely on search snippets. For every source that appears substantive:

1. Fetch the full page with `WebFetch`
2. Note: publication date, author affiliation, disclosed conflicts of interest
3. Extract claims that are specific and falsifiable (numbers, version-specific behaviors, architectural constraints)
4. Flag claims that are vague, unquantified, or vendor-motivated

Discard sources that:
- Have no publication date or are more than 3 years old for fast-moving domains
- Make strong claims without evidence or attribution
- Are clearly vendor-produced comparisons where the vendor wins

### Step 5: Triangulate and Identify Conflicts

For each evaluation axis, look for agreement and disagreement across sources:

- **Strong signal**: Multiple independent Tier 1–2 sources agree on a specific, quantified claim
- **Weak signal**: Only one source makes a claim, or all sources are from the same Tier 3–4
- **Conflict**: Sources disagree — dig into why (version differences, different workload profiles, different scales)
- **Gap**: No authoritative source covers an axis — this is a finding, not a reason to guess

When sources conflict, do not average them. Investigate the conflict. The explanation is usually one of:
- Different versions of the technology
- Different workload characteristics (write-heavy vs read-heavy, batch vs streaming)
- Different scale (100K req/s vs 10M req/s)
- One source is stale or wrong

### Step 6: Apply a Decision Framework

Once evidence is gathered, evaluate each option against the axes using this matrix:

```
| Axis                     | Weight | Option A | Option B | Notes          |
|--------------------------|--------|----------|----------|----------------|
| Throughput at target load| High   | ++       | +        | A: 2x in bench |
| Operational complexity   | High   | -        | ++       | A needs ZK     |
| Feature completeness     | Medium | ++       | +        | A has more     |
| Community/support        | Medium | ++       | +        | A larger OSS   |
| Migration cost           | High   | ++       | -        | B needs rewrite|

Rating: ++ strong advantage, + slight advantage, ~ neutral, - disadvantage
```

Weight each axis by what actually matters for the specific decision context.

### Step 7: Write the Research Output

Structure the output as follows. Be specific; omit padding.

---

```markdown
# Research: [Topic]

Date: [YYYY-MM-DD] | Confidence: [High / Medium / Low] | Sources: [N]

## Bottom Line Up Front

[1–3 sentences. The conclusion and the primary reason for it. State it even if
nuanced. If there is no clear winner, say that explicitly and say why.]

## Context and Constraints

[The specific decision context that shapes the evaluation. What changes if
constraints change.]

## Findings by Evaluation Axis

### [Axis 1: e.g., Throughput]
[Specific findings with source attribution. Quantify where possible.
Flag where evidence is thin.]
Source: [Title](url) — [one-line relevance note]

### [Axis 2]
...

## Comparison Matrix

[The weighted table from Step 6]

## Known Unknowns

[What this research could not determine. What would need a proof-of-concept
or internal benchmark to validate. Do not omit this section.]

## Recommendation

[Clear recommendation. If context-dependent, state exactly which context
leads to which choice. Not "it depends" — specify what it depends on.]

## Sources

1. [Title](url) — Tier [1–4], [date], [why it was used]
2. ...
```

---

## When to Stop Researching

Research has diminishing returns. Stop when any of these conditions are met:

1. **Saturation**: New sources are confirming existing findings without adding nuance. You have covered all 6 evaluation axes with at least Tier 2 sources.

2. **Diminishing precision**: Additional research would only sharpen a conclusion from "A is faster" to "A is 12% faster" — and that precision does not change the recommendation.

3. **Gap identified**: An axis cannot be answered from public sources. Name the gap and recommend how to fill it (e.g., "run a benchmark against your actual schema" or "contact the vendor for an enterprise eval").

4. **Time budget**: For time-sensitive decisions, a solid answer with known unknowns is better than a perfect answer that arrives too late. Flag confidence explicitly.

5. **The question has changed**: Mid-research you discover the original question was the wrong question. Stop, surface the reframing, and confirm before continuing.

**Do not keep searching to increase confidence from 85% to 90% if it does not change the recommendation.**

## Confidence Calibration

Label your output with an honest confidence level:

- **High**: Multiple Tier 1–2 sources agree; the finding is quantified; no significant conflicting evidence; the domain is stable (not shifting weekly).
- **Medium**: Evidence is directionally clear but thin in places; some axes rely on Tier 3 sources; the domain is moving fast enough that findings may age in 6 months.
- **Low**: Sparse evidence; significant conflict between sources; the question requires production-scale data that does not exist publicly; heavy extrapolation required.

Never suppress uncertainty. A Low confidence conclusion explicitly stated is more useful than a falsely confident one.

## Parallel Research with Subagents

For broad topics with 5+ independent axes, spawn parallel research agents using the Task tool — one agent per axis cluster. Each agent:

1. Receives a scoped sub-question and the source hierarchy rules
2. Returns findings with citations and a confidence assessment
3. Does not synthesize across other axes — that is the orchestrating session's job

Merge results by looking for cross-axis conflicts first (e.g., "Axis 2 says X has simpler ops, but Axis 4's postmortems show X has complex failure recovery — reconcile this").

## Quality Checklist Before Delivering Output

- [ ] Every specific claim (number, behavior, limitation) has a cited source
- [ ] At least one source per axis is Tier 1 or Tier 2
- [ ] No claim relies solely on vendor-produced content
- [ ] Conflicting sources are surfaced and explained, not silently averaged
- [ ] The "Known Unknowns" section is populated honestly
- [ ] The recommendation is specific — it names a choice, not just factors to consider
- [ ] Publication dates of all sources are noted; stale sources are flagged
- [ ] The output is scoped to the stated research question — no scope creep

## Anti-Patterns to Avoid

**False balance**: Presenting two options as equally valid when evidence clearly favors one. Hedge when genuinely uncertain; do not hedge to seem neutral.

**Recency bias**: Treating a 2024 blog post as more authoritative than a 2020 RFC. Date matters, but so does source tier.

**Confirmation searching**: Searching for evidence that confirms an assumed answer. Run searches designed to falsify the hypothesis too.

**Vendor trust**: Taking benchmark numbers from a vendor comparison at face value. Always check if the benchmark setup is disclosed and reproducible.

**Definitional drift**: Researching "serverless" when the user means "Lambda functions" vs "edge computing" vs "FaaS". Clarify the precise scope first.

**Synthesis from snippets**: Drawing conclusions from search result previews without reading the full source. Fetch and read the source.

**Omitting the failure case**: Most technology evaluations find the happy path. Actively search for failure modes, production incidents, and known limitations.
