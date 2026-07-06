# How this idea got here

This prototype didn't start as "build a hook that captures rationale." It
went through a few framings first, and some of the wrong turns are worth
keeping around — partly because they explain the current design better than
the final version does on its own, and partly because that's the whole
point of the project: showing the path, not just the destination.

## Framing 1: "Explain your work" UI

The original idea was bigger and vaguer: after an agent finishes a task, it
should generate a purpose-built UI — not a code dump, not a chronological
log — structured around the questions a human reviewer actually asks. Why
this dependency. Why this structure. What alternatives did you consider.
What are you assuming.

This framing was right about the *shape* of the output (judgment-oriented,
not execution-oriented) but assumed the hard part was rendering — that if
you just built the right UI, the content would follow. It didn't yet
address where the content was supposed to come from.

## Framing 2: Is this even real reasoning, or fabrication?

Early pushback (including from Claude, in an earlier conversation) was that
asking a model to explain itself after the fact risks pure confabulation —
a plausible-sounding story invented to fit a decision that was actually made
some other way.

The response that mattered: it doesn't need to be literally true to be
useful. A fabricated rationale for a reasonable decision tends to sound
reasonable. A fabricated rationale for a bad or incoherent decision tends to
have seams. So "sus" is still a usable signal even without certainty. This
removed fabrication as a blocking concern rather than solving it — it's
accepted as a tradeoff, not eliminated.

## Framing 3: The permission-prompt problem

The idea narrowed to something concrete and immediate: Claude Code's
permission prompts show a raw command with no context, which is where the
lack of explanation is most keenly felt day-to-day. This looked at first
like a UI gap — surface the reasoning that's already there, next to the
prompt.

## Framing 4: Wait — is the reasoning even there?

Pushback on framing 3: does the reasoning actually exist somewhere, or is
"post-hoc" doing more work in that description than it should? This turned
out to be the right question. Two real possibilities exist, not one:

- The reasoning was generated (visible thinking text) but just isn't shown
  at the prompt — a genuine UI gap.
- The reasoning was never generated at all, because adaptive thinking
  treated the step as "simple" and skipped deliberation entirely — not a UI
  gap, a computation gap.

Confirmed via docs: Claude Code uses adaptive thinking, which decides for
itself how much to reason based on perceived complexity and an effort
setting. That explained why explanations felt inconsistent, especially
across multi-step command sequences where each individual step can look
simple in isolation.

## Framing 5: Fix it with CLAUDE.md / effort settings

Reasonable next step: raise the effort setting, add trigger phrases, write a
CLAUDE.md instruction asking the agent to narrate plans before multi-step
sequences. This is a real, usable fix — and still worth doing — but it
surfaced a bigger frustration: a config file is a request the model can
ignore, not a guarantee. Every fix up to this point still routed through
"the user configures the AI to behave better," which is itself a form of
the friction this project is trying to remove.

## Framing 6: The actual target — capture, not display

The reframe that stuck: stop trying to fix the live moment (the permission
prompt, the terminal scrollback) at all. Instead, force the rationale to be
generated and captured silently, as structured data, regardless of whether
anyone reads it in real time. Nothing about this requires the user to
engage with it live — it removes the "does the reviewer actually stay
engaged with this" problem entirely, because there's no live review step to
disengage from.

This is also the point where the project stopped being about explaining a
single decision and became about accumulating a rationale log that can
power multiple downstream artifacts later — PR descriptions, orientation
docs, review-priority flags — none of which need to be designed yet, because
they're all just different consumers of the same captured log.

## What carried through every framing

- The reasoning should be structured around reviewer judgment, not
  execution order.
- Fabrication/reconstruction risk is tolerable if the output is used for
  orientation and suspicion, not as a certified audit trail.
- The fix should be as close to inherent/default behavior as possible, not
  something the user has to remember to configure per project.
