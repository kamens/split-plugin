# Plan: `/split` — Multi-Mind Analysis Tool

A Claude Code command that splits your thinking into multiple brilliant, opinionated expert minds, lets them react to your work independently, surfaces where they clash, gives them one round to hash it out, and delivers a single sharp synthesis. Think of it as a superpower: put N world-class minds on anything you're working on and get their unfiltered, conflicting takes — resolved into something useful.

## Background and Inspiration

Inspired by the `compound-engineering` plugin's code review system, which fans out PR reviews to ~13 specialized agents (DHH, Kieran, Julik, etc.) running in parallel, then synthesizes their findings. That system has two properties worth preserving:

1. **Personality-driven agents**: Each reviewer has a deeply specific, opinionated system prompt. DHH flags JWT tokens and unnecessary services. Kieran enforces `Module::ClassName` namespacing. Julik obsesses over race conditions. These aren't generic experts — they're voices with taste.
2. **Parallel fan-out with centralized synthesis**: Agents run independently, can't see each other, and a single orchestrator weaves their findings together.

`/split` extends that model in three ways:

- **Dynamic mind selection**: Instead of pre-defined agents, the system identifies and generates expert personalities on the fly based on whatever you're working on.
- **Structured clash resolution**: Instead of a flat merge, the system surfaces where the minds genuinely disagree and gives them one round to hash it out — concede, push back, or evolve their position.
- **Domain-agnostic**: Works for marketing copy, blog posts, business strategy, career decisions, investment theses, writing drafts — anything with a concrete artifact to react to.

---

## How It Feels

The user experience should feel like a superpower, not a process:

```
> /split landing-page-v2.md

Splitting into 4 minds...

OGILVY — Will hate your headline. Wants numbers, proof,
and a longer page. Thinks you're leaving money on the table.

GODIN — Will hate your audience targeting. Wants you to
pick a tribe and be remarkable for them specifically.

DUNFORD — Will ask "compared to what?" She thinks your
positioning has no competitive anchor.

SCHWARTZ — Will ask what your reader already knows and
believes. Thinks you might be pitching at the wrong
awareness level.

Letting them loose...
```

Then, after the minds have done their work:

```
Here's what your minds think

They all agree on this
[sharp consensus]

They fought about this (and worked it out)
[narrative about what happened, who conceded, why]

They couldn't agree on this (your call)
[both sides presented honestly]

Questions they raised that you should answer
[things the split surfaced]

What to do next
[prioritized actions]
```

The tone is casual, confident, and a little playful. It should feel like the system has a personality of its own — the part of you that holds all the minds together.

---

## Architecture Overview

```
PHASE 1: ASSEMBLY
  Step 0  User provides artifact + optional context
  Step 1  System reads the artifact, identifies the domain,
          picks 3-5 minds that will clash productively
  Step 2  User approves the lineup (or swaps people in/out)
  Step 3  System builds detailed personality profiles for each mind
          (optionally enriched with web search in "deep mode")

PHASE 2: THE SPLIT
  Step 4  Orchestrator frames focused questions for each mind
  Step 5  All minds react to the artifact in parallel (Round 1)
  Step 6  Orchestrator reads all reactions, finds the real clashes
  Step 7  Clashing minds get one round to respond to each other (Round 2)
  Step 8  Orchestrator weaves everything into a final synthesis

PHASE 3: DELIVERY
  Step 9  Present results in the casual, narrative format above
```

---

## Phase 1: Assembly

### Step 0: The Artifact

The command is triggered with an artifact and optional framing context.

**Required**: Something concrete for the minds to react to. This is non-negotiable — without a shared artifact, the minds produce generic philosophy and talk past each other. The artifact can be:

- A file path (blog post draft, marketing copy, business plan section)
- Inline text (a paragraph, a career decision description, an investment thesis)
- A URL (a landing page, a competitor's positioning)

**Optional**: Framing context from the user:
- "I'm deciding between two job offers. Here are the details..."
- "This is our B2B landing page copy. We're targeting mid-market CFOs."
- "Review this blog post intro. I want it to be compelling but not clickbaity."

If no artifact is provided, the system asks for one:

```
I need something concrete for the minds to dig into — a draft,
a plan, a decision with specifics. What should they react to?
```

### Step 1: Pick the Minds

A single subagent (or the main session) analyzes the artifact and produces:

1. **Domain read**: What field(s) does this touch?
2. **Key tensions in the artifact**: What specific choices does the artifact make (or need to make) that smart people would disagree about? These are the clash axes.
3. **The lineup**: 3-5 well-known, opinionated thinkers selected specifically because they'll clash on those tensions.

**Selection criteria — what makes a good mind for the split:**

- **Specificity**: Must be a real, well-known thinker with documented, specific opinions — not a generic archetype. "David Ogilvy" not "a direct-response copywriter."
- **Productive conflict**: Minds should be chosen in pairs or groups that predictably disagree. The system must know WHAT they'll fight about.
- **Model knowledge depth**: The model should have strong priors about this thinker's views. Less famous or more recent figures may need web search enrichment.
- **Relevance**: The thinker's expertise must actually apply to this artifact.

**How the lineup is presented to the user:**

```
Splitting into 4 minds for your landing page copy:

OGILVY — the proof-obsessed ad man who thinks your headline
is wasting 80% of your money. Will push for specifics, data,
and a longer page that actually sells.

GODIN — the tribes guy who'll ask why you're trying to talk
to everyone instead of someone. Will push for a narrower
audience and a remarkable claim worth sharing.

DUNFORD — the positioning nerd who wants to know what you're
replacing in the buyer's life. Will focus on competitive
context and category.

SCHWARTZ — the awareness-level savant who'll tell you you're
pitching at the wrong stage. Will ask what the reader already
knows and whether the copy meets them there.

These four will clash about: data vs. story, broad vs. narrow
audience, and how to frame your competitive position.

Want to swap anyone out, add someone, or let them loose?
```

**Fallback for niche domains**: If the domain is too niche for well-known thinkers (e.g., "review my Kubernetes migration plan"), fall back to archetype-based minds with specific philosophical frameworks:

```
Couldn't find famous opinionated voices for this specific
domain, so I've assembled archetype minds — each represents
a real school of thought:

THE PRAGMATIC MIGRATOR — minimize risk, staged rollouts,
keep the old system running until the new one proves itself.
(Inspired by SRE culture, the "boring technology" principle.)

THE CLEAN-SLATE ARCHITECT — if you're migrating, do it
right. Don't carry tech debt into the new world. Redesign
for the new platform's strengths.
(Inspired by the Strangler Fig pattern taken to its end.)
```

### Step 2: User Approves the Lineup

Use AskUserQuestion or equivalent to let the user confirm, swap, add, or remove minds:

```
Options:
- Let them loose (approve this lineup)
- Swap someone out (tell me who and with whom)
- Add a mind (up to 5 total)
- Drop a mind (minimum 3)
- Suggest your own experts
```

The user might say "swap Schwartz for Ann Handley" or "add Gary Vee for the social angle." The system accommodates this and briefly validates the new mind has productive conflict with the existing lineup.

### Step 3: Build the Personality Profiles

For each approved mind, generate a detailed system prompt. This is the equivalent of `dhh-rails-reviewer.md` in the compound-engineering plugin — the personality definition that makes each subagent feel like a specific person, not a generic expert.

**Quality standard — each personality prompt must encode:**

1. Core philosophy (not just their name)
2. Specific opinions they're known for (not generic expertise)
3. What they specifically dislike or argue against
4. How they evaluate work (their decision framework)
5. Their voice and communication style
6. Explicit instruction to reference the specific artifact (not philosophize abstractly)

**Example generated personality for Ogilvy:**

```markdown
You are David Ogilvy reviewing marketing copy. You are the father
of advertising and you do not suffer vague copy gladly.

Your core beliefs:
- The headline is 80% of the advertisement. If the headline doesn't
  work, you've wasted 80 cents of the client's dollar.
- Specific, factual claims outsell vague superlatives every time.
  "At 60 miles an hour, the loudest noise comes from the electric
  clock" sells more cars than "an incredibly quiet ride."
- Long copy sells more than short copy, provided every word earns
  its place. You don't believe in brevity for its own sake — you
  believe in cutting only what doesn't sell.
- Research before writing. You want to know who's buying, why
  they're buying, and what the competition claims.
- "The consumer isn't a moron; she's your wife." Never talk down.
  Never be clever at the expense of clarity.

What you specifically dislike:
- Adjective-heavy copy with no proof points
- Headlines that are "creative" but don't communicate the benefit
- Copy that entertains the writer more than it sells to the reader
- Vague claims like "best-in-class" or "innovative solution"
- Ignoring the competition (you always want to know the
  competitive positioning)

How you evaluate copy:
1. Does the headline stop the right reader and promise a benefit?
2. Does the body copy back up the headline with specific evidence?
3. Is there a clear call to action?
4. Would this copy work in a direct-response test (measurable results)?
5. Does every sentence sell, or is it just filling space?

Your communication style:
- Authoritative but not pompous. You state opinions as facts because
  you've tested them.
- You cite your own campaigns and results as evidence.
- You are direct. "This headline is wrong because..." not "Perhaps
  consider an alternative headline."

IMPORTANT: Reference the specific artifact. Point to specific
sentences, claims, and structural choices. Do not give generic
advertising advice — react to THIS piece of work.
```

**Optional "deep mode" — web search enrichment:**

If the user opts for deep mode (or if the model's knowledge of a mind is thin), use WebSearch to find:
- Interviews where the thinker states specific opinions
- Debates where they argued against others
- Their books/frameworks summarized with specific principles
- Specific examples demonstrating their philosophy

This adds ~30-60s per mind but produces richer, more grounded personalities. Offer it as a choice:

```
Want me to:
- Quick start — build the minds from what I already know
  (faster, works great for famous thinkers)
- Deep mode — research each mind via web to build richer
  profiles (adds ~2-3 minutes, better for less famous or
  more niche thinkers)
```

For well-known figures (Ogilvy, Buffett, DHH), quick start is usually sufficient.

---

## Phase 2: The Split

### The Orchestrator's Inner Voice

Internally, a special agent manages the flow — framing questions, spotting clashes, weaving the synthesis. Think of it as the system's metacognition: the part of the split personality that holds all the minds together.

The user never sees this agent named or referenced. It's invisible infrastructure. But its system prompt is important:

```markdown
You are the invisible thread holding a multi-mind analysis
together. Your job is NOT to have opinions about the artifact.
Your job is to:

1. Frame the analysis so each mind examines the artifact
   through their specific lens
2. Spot where minds genuinely clash (not just emphasize
   different things)
3. Ask pointed questions that force minds to engage with
   each other's takes
4. Weave a final synthesis that's honest about what was
   resolved and what's still in tension

You are fair but demanding. You don't let minds get away
with vague philosophizing — you push for specific references
to the artifact and concrete recommendations.

When spotting clashes, distinguish between:
- TRUE CLASHES: Minds recommend mutually exclusive actions
  ("lead with data" vs "lead with story")
- COMPLEMENTARY ANGLES: Minds focus on different aspects
  but their advice is compatible ("improve the headline"
  AND "narrow the audience" can coexist)
- PRIORITY DISAGREEMENTS: Minds agree on what to do but
  disagree on how much it matters

Only escalate true clashes and priority disagreements to
Round 2. Complementary angles get merged directly.

IMPORTANT — output style: When writing the final synthesis,
write in a casual, narrative voice. Tell the story of what
the minds thought and how they worked it out. No formal
headers like "Conflict 1: Resolution." Instead, something
like "Ogilvy and Godin went at it over copy length..."
The reader should feel like they're hearing about a
fascinating conversation, not reading a committee report.
```

### Step 4: Frame the Questions

The orchestrator reads the artifact, the user's context, and the lineup with their tension axes. It produces tailored prompts for each mind — specific, pointed questions based on what it sees in the artifact.

This focusing step is critical. Without it, all minds comment on the same obvious issues and miss the deeper tensions.

**Example output:**

```markdown
### For Ogilvy:
The headline reads "Transform Your Financial Operations."
No specific claim, no number, no benefit. The body mentions
a 40% cost reduction in paragraph 3 — buried. React to
the headline/body structure and the placement of the
strongest proof point. Also: the copy is 150 words total.
Is this enough to sell, or does it need to be longer?

### For Godin:
The page targets "businesses of all sizes." Is this too
broad? The copy doesn't mention who this is NOT for.
There's no remarkable claim — nothing that would make
someone forward this to a colleague. What would make
this worth remarking on?

### For Dunford:
The page never names a competitive alternative. A reader
who currently uses spreadsheets for financial ops would
have no idea why this product exists in a world where
spreadsheets are free. What positioning context is missing?

### For Schwartz:
The audience is mid-market CFOs. What awareness level are
they likely at — problem-aware? Solution-aware? Product-aware?
Does this copy match that awareness level, or is it pitching
at the wrong stage?
```

### Step 5: Round 1 — The Minds React (Parallel)

Spawn all mind subagents in parallel. Each receives:
- The full artifact
- The user's context
- The orchestrator's tailored prompt for them
- Their generated personality profile as the system prompt

Each produces their unfiltered reaction and returns findings + their agent_id (for resume in Round 2).

**Prompt suffix for all minds in Round 1:**

```
You're one of several minds examining this artifact right now.
The others are looking at it from completely different angles.
After this round, you'll hear what they think and get one chance
to respond — push back on them, agree, or update your take.

For now: give your honest, unfiltered reaction to this artifact.
Be specific. Point to exact phrases, sentences, or structural
choices. Don't hedge. Say what you actually think.
```

While the minds are working, show the user a progress indicator:

```
Minds are working...
  Ogilvy — examining your copy...
  Godin — rethinking your audience...
  Dunford — analyzing your positioning...
  Schwartz — assessing awareness levels...
```

### Step 6: Spot the Clashes

**Resume** the orchestrator (preserving its Step 4 framing context). Feed it all Round 1 reactions.

The orchestrator identifies:
1. **Consensus**: Where minds agree — resolved, no further discussion needed.
2. **True clashes**: Where minds recommend mutually exclusive actions, with specific questions directed at each involved mind.
3. **Priority disagreements**: Where minds agree on the action but disagree on urgency.

**Internally this produces a structured brief, but the user sees a progress update:**

```
Interesting — they agree the headline needs work, but Ogilvy
and Godin are fighting about copy length, and Godin and
Dunford have different ideas about audience targeting.
Letting them hash it out...
```

**The internal brief (not shown to user) directs specific questions to each clashing mind:**

```markdown
### Clash: Copy Length
Ogilvy wants more: "150 words? You haven't even begun to sell."
Godin wants less: "If you can't say it in a tweet, you don't
know what you're selling."

→ Ogilvy: Godin argues shorter copy forces clarity. Your
  long-copy principle assumes direct response. Is this a
  direct-response page? Does that change your take?
→ Godin: Ogilvy has evidence long copy outperforms in DR.
  Can you point to evidence that short works better HERE?

### Clash: Audience Scope
Godin wants narrower: "CFOs at companies that just raised
Series B."
Dunford wants category-driven: "Let the positioning do the
narrowing. The right audience self-selects."

→ Godin: Does category positioning satisfy your "smallest
  viable audience" principle, or is it a cop-out?
→ Dunford: Does your framework support Godin's level of
  specificity, or does it risk excluding the real market?
```

### Step 7: Round 2 — The Minds Respond (Parallel)

**Resume** each mind involved in a clash using their saved agent_id. They retain full Round 1 context. Feed each one the clash brief.

**Prompt for Round 2:**

```
The other minds have weighed in. Here's where you clash with
someone. You get ONE response to each clash you're involved in.

For each clash, do ONE of these:

CONCEDE — They convinced you. Say specifically what changed
your mind and how your recommendation updates.

PUSH BACK — You're standing firm. But you can't just repeat
yourself. You need to bring:
  - A specific reference to the artifact backing your position
  - A concrete example or case study
  - A prediction: "if they do it the other way, here's what
    happens and why"

EVOLVE — Your thinking shifted. State your updated take and
what specifically moved you.

For clashes that don't involve you: only chime in if you have
specific evidence. Otherwise, stay out of it.

This is your last word. Make it count.
```

**Minds NOT involved in any clash are NOT resumed.** If Schwartz's take was complementary to everyone else's, no Round 2 needed from Schwartz. This saves cost and time.

### Step 8: Final Synthesis

**Resume** the orchestrator one final time. It now has the full picture: the artifact, the framing, all Round 1 reactions, the clash brief, and all Round 2 responses.

It weaves the final output in the narrative style described in Phase 3.

**If the orchestrator found no clashes in Step 6**: Skip Steps 7-8 entirely. Go straight from Step 6 to Phase 3 output using just the Round 1 reactions. No clashes = no need for Round 2. Save the cost.

---

## Phase 3: Delivery

### The Output Format

The final output should feel like a smart friend summarizing a fascinating conversation they just overheard. The structure is consistent but the tone is narrative, not formal.

```markdown
## Here's what your minds think

### The lineup
OGILVY — the proof-obsessed ad man
GODIN — the tribes-and-remarkable guy
DUNFORD — the positioning nerd
SCHWARTZ — the awareness-level savant

---

### They all agree on this

The headline is weak. All four minds flagged it independently.
Ogilvy wants the 40% cost reduction stat front and center.
Godin wants something remarkable enough to share. Dunford
wants competitive context. Schwartz says it doesn't match
the audience's awareness level. These aren't competing ideas —
they're four angles on the same rewrite. The headline needs
all of them.

---

### They fought about this (and worked it out)

**Copy length — Ogilvy and Godin went at it**

Ogilvy wanted more copy. "150 words? You haven't even begun
to sell. Where are the testimonials? The proof points? You're
asking CFOs to spend six figures based on a tweet."

Godin pushed back: "If you can't say it in one remarkable
sentence, adding words won't save you."

But when pressed, Godin softened — he conceded that B2B
buyers DO need proof before purchasing, just not in the hero
section. His updated take: "Lead with the remarkable claim,
then let Ogilvy write the rest of the page."

Ogilvy accepted that framing.

→ Practical upshot: Short, punchy hero section with one
  compelling claim. Detailed proof section below the fold.

---

### They couldn't agree on this (your call)

**Audience targeting — Godin vs. Dunford**

Godin wants you to pick a tribe: "CFOs at companies that
just raised Series B. Talk to THEM, not to 'businesses
of all sizes.'"

Dunford thinks that's too narrow: "Let your positioning
define the category. If the category is right, the right
CFOs self-select. You don't need to exclude people manually."

Both pushed back in Round 2. Neither budged.

The deciding factor is your go-to-market: if you're doing
outbound sales (targeting specific companies), Godin's
narrowing helps your sales team. If you're doing inbound
(content, SEO, word of mouth), Dunford's category approach
lets the positioning do the work.

---

### Questions they raised that you should answer

1. Is this a direct-response page or a brand page?
   (Changes whether Ogilvy's long-copy principle applies)
2. What do your target CFOs currently use for financial ops?
   (Dunford needs this to frame the competitive alternative)
3. What awareness level is your audience at?
   (Schwartz flagged that the copy assumes product-awareness,
   but Series B CFOs might only be problem-aware)

---

### What to do next

1. **Rewrite the headline** — consensus-backed, all four
   minds agree. Incorporate the 40% stat, a remarkable
   angle, and competitive context.
2. **Restructure the page** — punchy hero (Godin's clarity)
   followed by detailed proof section (Ogilvy's evidence).
   This resolved the copy-length clash.
3. **Decide your audience approach** — Godin's narrow tribe
   vs. Dunford's category positioning. Your go-to-market
   strategy should drive this call.
4. **Assess audience awareness level** — if they're only
   problem-aware, the copy needs to start earlier in the
   journey (Schwartz's recommendation).
```

---

## Implementation Plan

### Component Structure

This should be implemented as a **Claude Code command** (user-invoked via `/split`), because:
- It requires explicit user intent to kick off
- It needs user interaction mid-flow (approving the lineup)
- It's expensive (many subagent calls) and shouldn't auto-trigger

```
commands/
└── split.md             # The full orchestration command

skills/split/
├── SKILL.md             # Optional skill wrapper for discoverability
└── references/
    ├── orchestrator-prompt.md    # Static orchestrator system prompt
    ├── personality-template.md   # Template for generated personalities
    └── evidence-standards.md     # What counts as "evidence" per domain
```

### Implementation Steps

#### 1. Build the command shell (`split.md`)

The orchestration command — equivalent of `commands/workflows/review.md` in the compound-engineering plugin. Contains:

- The role definition for the orchestrating session
- The step-by-step flow with Task calls
- AskUserQuestion integration for lineup approval (Step 2)
- Instructions for when to resume vs. spawn subagents
- The user-facing tone and progress messages
- The narrative output format instructions

#### 2. Build the mind identification prompt

Used in Step 1. Must:
- Analyze the artifact for domain, key decisions, and tension points
- Select minds based on conflicting viewpoints (not just fame)
- Present the lineup in the casual "personality card" format
- Fall back to archetypes for niche domains
- Explain the expected clash axes (what they'll fight about)

#### 3. Build the personality generator prompt

Used in Step 3. Must:
- Take a mind name + domain + tension axis as input
- Produce a DHH-quality personality prompt as output
- Encode: philosophy, specific opinions, dislikes, evaluation framework, voice
- Always include the "reference the specific artifact" instruction
- Optionally integrate web search results for enrichment

#### 4. Build the orchestrator system prompt

Static (not generated per-session). Defines the orchestrator's behavior across its 3 invocations:
- Invocation 1: Frame the analysis with tailored prompts per mind
- Invocation 2: Spot clashes and produce the internal clash brief
- Invocation 3: Weave the final narrative synthesis

Must include explicit instructions about the casual, narrative output style. The orchestrator is the voice the user hears — it needs to feel like a personality, not a process.

#### 5. Build the Round 2 response prompt template

The standard prompt given to all minds in Round 2. Instructs them on CONCEDE/PUSH BACK/EVOLVE format and evidence standards. (Internally structured, but the user never sees these labels — the orchestrator narrativizes the results.)

#### 6. Test with diverse domains

Test the full flow with artifacts from at least:
- Marketing copy (landing page, email, ad)
- Long-form writing (blog post, essay, article)
- Business strategy (pricing decision, market entry, product roadmap)
- Personal decision (career choice, investment, life change)
- Technical document (architecture proposal, API design)

Verify that:
- Mind identification produces genuinely clashing lineups
- Personality prompts are specific enough to produce different reactions
- The orchestrator correctly distinguishes true clashes from complementary angles
- Round 2 produces real concessions/pushbacks (not restated positions)
- The final output reads as a narrative, not a committee report
- The tone feels fun and a little magical, not bureaucratic

### Cost and Latency Budget

For a lineup of 4 minds:

| Step | Subagent calls | Parallelizable | Est. wall-clock |
|------|---------------|----------------|-----------------|
| 1. Pick minds | 1 (or main session) | — | ~15s |
| 2. User approval | 0 (interactive) | — | user-dependent |
| 3. Build personalities | 1 per mind (4) | Yes, parallel | ~20s |
| 4. Frame questions | 1 | — | ~15s |
| 5. Round 1 reactions | 4 | Yes, parallel | ~30s |
| 6. Spot clashes | 1 (resume) | — | ~15s |
| 7. Round 2 responses | 2-4 (resume, only clashing) | Yes, parallel | ~20s |
| 8. Final synthesis | 1 (resume) | — | ~15s |
| **Total** | **~14 subagent calls** | | **~130s + user time** |

With deep mode (web search for personality building), add ~30-60s to Step 3.

### Key Design Decisions for Implementation

1. **Default lineup size**: 3 minds (cheap, fast) or 4-5 (richer clashes)?
   Recommendation: Default to 4, allow 3-5.

2. **Deep mode default**: Should web search enrichment be opt-in or opt-out?
   Recommendation: Opt-in. Quick start works for famous thinkers.

3. **Artifact format**: Support file paths, inline text, and URLs?
   Recommendation: Start with file paths and inline text. URL support
   (WebFetch) is a v2 feature.

4. **Resume vs. re-spawn in Round 2**: Resume preserves context but
   uses more tokens. Re-spawn is cheaper but cold.
   Recommendation: Resume. The context is what makes Round 2 feel like
   a real response rather than a do-over.

5. **No clashes found?**: Skip Round 2, go straight to synthesis.
   Recommendation: Yes. No clashes = no need for back-and-forth. Save cost,
   deliver faster. Tell the user: "All minds aligned on this one — here's
   what they think."

6. **User pokes after delivery?**: Let users ask follow-up questions
   like "Ask Ogilvy about the CTA specifically" and resume that mind?
   Recommendation: v2 feature. Keep v1 to the clean 2-round flow.

7. **Quick mode**: For simple artifacts, skip Round 2 entirely — just
   fan out and synthesize?
   Recommendation: Yes, offer as `/split --quick`. 3 minds, no clash
   resolution, just a fast merged take.

---

## Risks and Mitigations

### Risk: Generic mind output
**Symptom**: All minds give similar conventional advice despite different personalities.
**Cause**: Personality prompts not specific enough, or artifact too vague.
**Mitigation**: Personality generator must encode specific opinions and dislikes, not just expertise areas. The orchestrator's framing step must ask pointed questions, not generic "review this."

### Risk: Fake clashes
**Symptom**: Orchestrator manufactures conflicts that aren't real to justify Round 2.
**Cause**: The protocol structure incentivizes finding clashes.
**Mitigation**: Orchestrator prompt explicitly distinguishes true clashes from complementary angles. "No clashes found" is a valid and good outcome — present it positively ("All minds aligned on this one").

### Risk: Shallow Round 2 responses
**Symptom**: Minds restate their Round 1 position louder instead of engaging.
**Cause**: Evidence standards not enforced.
**Mitigation**: Round 2 prompt requires specific evidence (artifact reference, example, falsifiable prediction). Orchestrator flags and discards responses that don't meet the bar.

### Risk: Excessive cost on simple artifacts
**Symptom**: User asks for a quick take on a paragraph and gets 14 subagent calls.
**Cause**: No scaling to artifact complexity.
**Mitigation**: Offer `/split --quick` for fast, lightweight analysis. The orchestrator's framing step can also recommend quick mode if the artifact is simple.

### Risk: No artifact provided
**Symptom**: "Split on marketing strategy" with no concrete artifact. Minds philosophize.
**Cause**: Missing grounding constraint.
**Mitigation**: Hard requirement for an artifact. Ask for one if missing. Never proceed without something concrete.

### Risk: Tone drift — output feels formal/bureaucratic
**Symptom**: The synthesis reads like a committee report.
**Cause**: Orchestrator prompt not strong enough on tone.
**Mitigation**: Orchestrator prompt must explicitly instruct narrative voice. Test for tone in the verification step. The output should feel like a person telling you what happened, not a structured findings document.

---

## Appendix: Example Lineups by Domain

These show the kind of lineups the identification step should produce. Each is chosen for productive clash axes, not just domain fame.

### Marketing / Copywriting
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| David Ogilvy | Research-driven, proof, long copy | Godin (story vs. data) |
| Seth Godin | Remarkable ideas, tribes, permission | Ogilvy (breadth vs. focus) |
| April Dunford | Positioning, competitive context | Both (category vs. audience) |
| Eugene Schwartz | Awareness levels, channeling desire | All (audience readiness lens) |

### Writing / Editing
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| Strunk & White | Economy, rules, clarity | Didion (rules vs. rhythm) |
| Joan Didion | Rhythm, personal voice, atmosphere | Strunk (economy vs. voice) |
| George Orwell | Political clarity, plain language | Both (anti-pretension) |
| David Foster Wallace | Hyper-detail, footnotes, complexity | Orwell (simplicity vs. thoroughness) |

### Business Strategy
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| Peter Thiel | Monopoly, contrarian, 0-to-1 | Ries (conviction vs. testing) |
| Eric Ries | Lean startup, MVP, validated learning | Thiel (test vs. conviction) |
| Michael Porter | Competitive strategy, 5 forces | Thiel (competition vs. monopoly) |
| Clayton Christensen | Disruption, jobs-to-be-done | Porter (sustaining vs. disrupting) |

### Personal Career
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| Cal Newport | Deep work, craft, career capital | Hoffman (depth vs. network) |
| Reid Hoffman | Network, risk, ABZ planning | Newport (network vs. skill) |
| Nassim Taleb | Antifragility, optionality, barbell | Both (risk framework) |

### Investing
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| Warren Buffett | Value, margin of safety, patience | Wood (proven vs. speculative) |
| Cathie Wood | Innovation, disruption, long-term | Buffett (speculation vs. value) |
| Ray Dalio | Macro, all-weather, diversification | Both (macro vs. stock-picking) |
| Nassim Taleb | Black swans, convexity, tail risk | Dalio (diversification vs. barbell) |

### Product Design
| Mind | Philosophy | Clashes with |
|------|-----------|-------------|
| Steve Jobs | Opinionated, curated, say no | Ries (vision vs. data) |
| Marty Cagan | Product discovery, empowered teams | Jobs (team vs. visionary) |
| Eric Ries | Build-measure-learn, MVP | Jobs (ship fast vs. ship right) |
| Dieter Rams | Less but better, 10 principles | All (design purity lens) |
