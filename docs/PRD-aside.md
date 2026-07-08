# PRD: aside — invisible rationale capture for coding agents

**Status:** Prototype idea
**Author:** Nicole Dicochea

## Problem

AI coding agents — regardless of framework — make plenty of small judgment
calls while working — adding a dependency, introducing an abstraction,
structuring code a certain way. 

How much reasoning you get is left to the model's own judgment about what counts 
as worth explaining. Simple-looking steps — a new dependency, a structural choice, 
a rename — often get none, even though those are frequently exactly the moments a 
reviewer would want a reason for. The developer still has to review and ship the 
result, but has no reliable guarantee of *why* a specific choice was made, only a
tendency that varies by task complexity and effort setting.

Existing agent UIs solve for **status** ("edited 14 files, tests passed"),
not **guaranteed judgment** ("chose X over Y because Z, every time this
category of decision comes up"). Post-hoc summaries, generated cold at the
end of a task, are weaker than they could be because the model has no
memory of the intermediate reasoning by the time it's asked to reconstruct
it.

## Observed example

2026-07-05, starting a fresh Claude Code session, effort set to **High**:
told the agent to look at an existing project directory. It immediately ran
a compound Bash command (`ls -la && echo "---" && find . -maxdepth 3 ... |
head -100`) with no preceding explanatory text — just "Running a command"
straight into the permission prompt. The prompt itself showed only the raw
command string.

Update: this was less so evidence that Claude Code under-explains in general.
On longer or more substantive tasks, especially at higher effort, visible
reasoning shows up far more often. What this example actually demonstrates
is narrower and more useful: even at the highest effort setting, a step
the model classifies as "simple" can still produce zero visible reasoning
— which means *whether* an explanation appears is still discretionary, not
guaranteed, regardless of how good the model's reasoning generally is.

That's the specific gap this project targets, not a claim that agents
don't explain themselves.

## Goal

Capture the rationale behind flagged categories of agent decisions *at the
moment they happen*, without interrupting the developer in real time, so
that rationale can power better downstream artifacts: PR descriptions,
orientation docs, review-priority flags. The mechanism should generalize
to any agentic framework with an equivalent pre-action interception point,
not just the one it's first built against.

## Non-goals

- Not a permission/approval system. Nothing blocks the developer or requires
  live review.
- Not claiming captured rationale is guaranteed-truthful reasoning. Some
  reconstruction/confabulation risk is accepted — the rationale needs to be
  *useful*, not certified accurate. A rationale that reads as strange or
  evasive is itself a signal worth having.
- Not trying to capture reasoning for every tool call. Scope is limited to a
  narrow set of higher-stakes decision categories.
- Not claiming agents generally under-explain themselves. The claim is
  narrower: explanation is currently discretionary (the model decides when
  it's warranted), and this project makes it guaranteed for a specific,
  configurable set of decision categories, regardless of how "simple" the
  model judges the step to be.
- Not framework-exclusive. This PRD describes Claude Code as the reference
  implementation because that's what's actually being built and tested
  first, but the underlying mechanism — intercept before a flagged action,
  force rationale, capture invisibly — generalizes to any agentic
  framework with an equivalent pre-action hook or interception point.

## How it works (reference implementation: Claude Code — see aside-sketch.md for detail)

1. A `PreToolUse` hook watches for specific trigger categories (new
   dependency, new file, new abstraction, edits to sensitive paths).
2. On a match, the hook forces the agent to produce a short rationale before
   the tool call is allowed to proceed.
3. The rationale + tool call context is appended to a structured log
   (`aside-log.jsonl` or `.md`), invisible to the user during the session.
4. At any checkpoint (end of session, before a PR, on request), the log can
   be fed back to the agent to generate a summary, PR description, or
   review-priority list grounded in captured rather than reconstructed
   reasoning.

Claude Code is the first build target because it already has a documented
hook system that supports this mechanism directly. Any other framework with
an equivalent pre-action interception point (middleware, a plugin hook,
anything that can inspect a proposed action before it executes) could
implement the same pattern.

## Success looks like

- A developer can generate a PR description or session summary that
  correctly explains *why* non-obvious changes were made, without having to
  scroll back through a session or ask the agent to explain itself cold.
- The log surfaces at least some decisions a developer would otherwise have
  missed entirely (e.g., a dependency added three steps before the part of
  the diff they were focused on).
- Flagged rationale that reads as inconsistent or thin correlates with
  changes that turn out to need a closer look.
- The mechanism (trigger → forced rationale → invisible capture) proves out
  well enough on Claude Code that porting it to a second framework would be
  a matter of re-implementing the interception point, not rethinking the
  concept.

## Relationship to loop engineering

In mid-2026, "loop engineering" emerged as the term for a layer above
manual prompting: instead of a human prompting an agent turn by turn, the
human designs a self-running loop — discovery, handoff, verification,
persistence, scheduling — that lets an agent work autonomously across many
cycles, often checked by a separate evaluator model rather than a human at
each step. Boris Cherny (head of Claude Code) described this as "my job is
to write loops," not individual prompts.

This is not a competing approach to what's described in this PRD — it
operates at a different layer, and the two problems don't cancel out:

- **Loop engineering optimizes for reducing how often a human has to look
  at anything.** Its entire value proposition is autonomy: more work gets
  done per human touchpoint.
- **This project optimizes for what happens at the moment a human still
  has to look.** No matter how autonomous the loop, some point still
  exists where a person is accountable for the output — merging the PR,
  shipping the feature, putting their name on it. Loop engineering doesn't
  remove that point, it just makes it rarer and pushes more accumulated
  work behind it.

An evaluator model in a loop answers a narrower question than the one this
project is about: "did this pass a verifiable condition," not "would a
human with real context and judgment have made this same choice," and
certainly not "am I willing to be accountable for this." Those are
different questions. A generator/evaluator pair grading its own output can
tell you the tests pass. It has no mechanism for surfacing that a human
reviewer would have flagged a specific tradeoff, chosen a different
approach, or found the agent's suggestion not worth the effort it cost —
because it doesn't have that reviewer's taste or stakes in the outcome.

This mirrors a pattern that's already shown up earlier in this project's
own history (see THINKING.md): a CLAUDE.md instruction can't reliably
force good behavior because it's a request competing for the model's
attention, not a guarantee. The same logic applies one level up — an
automated evaluator can't close the accountability gap either, for the
same underlying reason. Trust that lets a human sign off has to terminate
in something a human actually looked at and understood. It can't be
delegated to another layer of automation, no matter how good that layer's
own checks are.

**Practical implication:** as loops get longer and more autonomous, the
moments a human actually reviews something get rarer and higher-stakes,
with less native context carried into each one (a human reviewing the
output of many autonomous cycles has far less continuity than someone who
watched a single session unfold). That makes a captured rationale log more
valuable over time, not less — it's the thing that can be handed to a
human at whatever point they re-enter an increasingly autonomous process,
giving them a shortcut back into judgment they didn't personally build.

Also worth noting: agent loops are already accepted as costing roughly
4–15x a standard chat interaction, largely to improve output quality via
extra verification cycles. Against that backdrop, this project's token
overhead (small, and confined to flagged decisions) is a comparatively
cheap price for solving a problem loops actively make worse — reviewability.

## Prior art

Checked before writing this up. The pieces exist; the specific combination
doesn't seem to, at least not surfaced publicly:

- **Hook-based audit logging is common.** Several open-source projects
  capture and visualize every Claude Code hook event (tool name, input,
  result) in real time via `PreToolUse`/`PostToolUse` scripts. This covers
  the "record what happened" half, but records raw tool calls, not a forced
  rationale — there's no step that makes the model explain *why*.
- **Deny-with-reason is a standard, documented hook pattern.** A
  `PreToolUse` hook returning a deny decision with a reason string, fed
  back to the model, is a first-class supported mechanism — not something
  novel about this idea. What's novel is using that mechanism specifically
  to manufacture a rationale rather than to block something dangerous.
- **Feedback-loop hooks exist for correctness, not judgment.** The common
  pattern (`PostToolUse` running a linter and injecting the result back as
  `additionalContext`) is a similar shape — hook output feeding back into
  the model's context — but applied to fixing errors, not to capturing
  reasoning behind a choice.
- **exe.dev / Shelley validate the underlying thesis, from a different
  angle.** Their engineering blog makes essentially the same core argument
  independently: when their coding agent produced a change they didn't
  like, the useful move was asking the agent what it *saw* (what existing
  pattern it modeled its change on) rather than what it did — which
  revealed the agent was being consistent with a flawed convention already
  in the codebase, not being careless. Their Shelley agent also uses a
  `think` tool whose output is preserved and surfaced to the user across
  sessions ("live thoughts"). This is closer to a general reasoning trace
  than a targeted rationale log — it captures whatever the model chooses to
  write down, rather than forcing rationale on specific reviewer-relevant
  categories of decision — but it's real evidence that "surface the model's
  reasoning, not just its output" is being independently discovered as
  valuable elsewhere.
- **Nothing found that accumulates forced rationale into a structured log
  for downstream summarization** (PR description, orientation doc,
  review-priority flags). That combination — force the explanation, capture
  it invisibly, reuse it later — appears to be the actual gap.

## Alternatives considered

- **CLAUDE.md instruction instead of a hook.** Tried first. Rejected because
  it's a soft nudge competing with everything else in context — nothing
  forces the model to actually comply, and in practice it's inconsistently
  respected. A hook can structurally deny a tool call; a markdown file can
  only ask.

- **Live permission-prompt with an explanation attached.** Considered
  showing the rationale to the developer at the moment of the tool call,
  the same place Claude Code already prompts for permission. Rejected as
  the primary mechanism because it re-introduces the interruption problem
  this is trying to avoid — more informative prompts are still prompts.
  Capturing invisibly and surfacing on demand gets the same information
  without the added friction.

- **Force deeper reasoning ("thinking harder") vs. force reasoning to be
  reported.** These are different fixes for different failure modes: one
  makes the model actually deliberate before choosing (changes what's
  computed), the other makes it state a rationale regardless of how the
  choice was actually reached (changes what's said). This project takes the
  second approach — reporting is forced via the hook — on the bet that even
  a reconstructed-in-the-moment rationale is more useful than none, and that
  an oddly thin or inconsistent rationale is itself a signal worth having.

- **Treat this as a fabrication/trust problem to solve first.** Considered
  whether captured rationale needed some grounding or verification step
  before it's usable. Rejected as a blocker — the target use case is a
  developer skimming for orientation and red flags, not treating the log as
  an audit trail. A rationale that doesn't hold up under a second look is a
  prompt to look closer, not proof of malfunction.

## Cost tradeoffs

The deny/retry mechanism has a real token cost, and it's worth being
upfront about where it comes from rather than treating this as free:

- The deny message itself (small, fixed cost per trigger).
- The model generating the rationale (small — 1–3 sentences).
- **The retry of the original tool call** — this is the real tax. The
  model re-processes context to re-issue a call it had already formed
  once. You're not just paying for the explanation, you're paying to
  re-run part of a decision that already happened.

This scales with how often the trigger list fires, not with session
length. A narrow, well-tuned trigger list (a handful of fires per long
session) is a rounding error. A loose trigger list compounds fast — the
same failure mode as permission-prompt fatigue, except paid in tokens
instead of attention. This is one more reason to keep the trigger list
narrow and tune it empirically rather than casting wide from the start.

**Cheaper fallback if cost becomes a real problem:** switch from
`PreToolUse` deny/retry to `PostToolUse`. Let the tool call happen once,
normally, then immediately prompt for the rationale afterward — appended
to context rather than causing a second pass at the same decision. This
roughly halves the cost per triggered decision, since there's no repeated
tool call. The tradeoff: you lose the "no result exists yet to rationalize
around" property of the pre-hook version, since the model has already seen
the outcome by the time it explains itself. Given the project's existing
tolerance for reconstructed/imperfect rationale, this may be a reasonable
trade if the pre-hook version proves too expensive in practice — worth
prototyping both and comparing, rather than committing to one up front.

## Open questions

- What's the right default trigger list, and should it be
  project-configurable from the start?
- JSONL vs. markdown log format — optimizing for machine re-consumption vs.
  human skimmability.
- How to handle a rationale that fails to satisfy the hook's expectations
  without creating a denial loop.
- Does this hold up on a real multi-hour session, or does the trigger list
  need per-project tuning to avoid noise?

## Related / prior thinking

- Originated from a broader idea: agents should expose engineering
  judgment ("why this dependency," "what alternatives," "what's risky") in
  a reviewer-oriented format, not a chronological execution log.
- Directly motivated by the gap between what Claude Code already generates
  (reasoning text, sometimes, depending on the model's own read of task
  complexity) and what's actually guaranteed to reach the developer at
  decision points — which today is nothing, unless the model happens to
  judge that particular step worth explaining.
