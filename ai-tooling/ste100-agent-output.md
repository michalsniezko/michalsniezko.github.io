---
layout: default
title: STE100 for Agent-Facing Text
parent: AI Tooling
nav_order: 1
---

## STE100 for Agent-Facing Text

ASD-STE100 is a controlled-language standard built by the aerospace and defense industry to prevent maintenance technicians from misreading aircraft instructions. A misread torque spec or a misread sequence of steps can have severe consequences, and the intended readers are often not native English speakers with no author to call for clarification.

The standard's fix: one meaning per word, one instruction per sentence, active voice, simple tenses, short sentences. No hedging stacks, no synonym rotation, no multi-clause prose.

An AI agent parsing another agent's output is in a similar position. No back-channel, no way to ask "did you mean X or Y?". Tool descriptions, error messages, system prompts, and inter-agent instructions all carry the same risk a maintenance manual does: a sentence that looks clear to a human can branch two ways for a parser.

---

## The Problem with Typical Agent-Facing Text

Standard developer English is written for humans who fill in gaps from context. Agents cannot reliably do that.

Three patterns cause most misparsing:

**Synonym rotation** - the same thing gets several names in one document. The agent cannot tell whether "the user", "the customer", and "the client" are one entity or three. Pick one name and use it every time.

**Hedge stacking** - helper verbs pile up until the sentence says nothing actionable: "it is important to note that this may potentially help to improve performance". Either state the claim or delete it.

**Long multi-clause sentences** - a downstream model tokenizes these and the parse tree can go wrong. "Open the file and read line 3, then check if it matches" is one sentence with three steps. Two of those steps may be skipped, reordered, or treated as conditional.

---

## ASD-STE100 Structural Rules

The standard has structural rules (sentence shape) and lexical rules (word choice against a ~900-word approved dictionary). The structural rules can be applied without the dictionary.

### Active voice

Write who does what.

```
Before: "The state is synchronized across the configured backends."
After:  "The tool synchronizes state across the configured backends."
```

When the actor is genuinely unknown or irrelevant, passive is acceptable. Everywhere else, name the actor.

### One instruction per sentence

```
Before: "Open the file and read line 3, then check if it matches the expected value."

After:  "Open the file.
         Read line 3.
         Check that the value matches the expected value."
```

A downstream agent reading a numbered list is far less likely to skip a step than one parsing a compound sentence.

### Sentence length limits

- Procedures and instructions: 20 words maximum
- Descriptions and explanations: 25 words maximum

Long sentences are not always wrong because of length - they are wrong because of structure. The length limit is a proxy for structural complexity.

### No semicolons

ASD-STE100 bans the semicolon outright. A semicolon almost always joins two ideas that should be separate sentences for an agent reader.

```
Before: "The request succeeded; the response contains the updated record."
After:  "The request succeeded. The response contains the updated record."
```

### No phrasal verbs

A two-word verb ("spin up", "reach out", "kick off") has meanings the parts do not predict. Use the plain single verb.

```
Before: "Spin up the worker."     → "Start the worker."
Before: "Reach out to the API."   → "Call the API."
Before: "Kick off the pipeline."  → "Start the pipeline."
Before: "Dive into the logs."     → "Read the logs."
```

### No noun clusters longer than three words

```
Before: "high pressure fuel pump inlet valve assembly status check"
After:  "the check that reads the status of the high-pressure fuel pump valve"
```

Four or more nouns stacked as a modifier force the reader to find the head noun and then re-parse the chain. Models do not always find the right head.

### No nominalization

An action frozen into a noun makes the sentence longer and hides who acts.

```
Before: "Perform an analysis of the log file."
After:  "Analyze the log file."

Before: "The tool provides assistance to downstream agents."
After:  "The tool helps downstream agents."
```

### No marketing adjectives

Words that claim quality instead of showing it add noise without information: seamless, robust, powerful, cutting-edge, effortless, blazing-fast. Delete them or replace with the measurement that earns the claim.

```
Before: "The tool offers a seamless integration with your existing pipeline."
After:  "The tool integrates with your existing pipeline."
```

### Keep modality intact

Hedges carry the author's confidence, and confidence is content. A rewrite that upgrades a hedge to a fact changes the claim.

```
Before: "The request may have failed due to a timeout."
Wrong:  "The request failed due to a timeout."   ← removes "may have"
Right:  "The request may have failed. A timeout is the most common cause."
```

This is the most common mistake when rewriting for clarity: a shorter sentence that silently removes a hedge is not a simplification - it is a different statement.

---

## Before / After Examples

### Tool description

```
Before:
"This tool will attempt to synchronize state across the various backends that
have been configured, and if a conflict is detected it may resolve it
automatically depending on the strategy that has been set, or otherwise it
will surface the conflict for manual review."

After:
"The tool synchronizes state across the configured backends. If it finds a
conflict, it checks the current strategy. If the strategy allows automatic
resolution, the tool resolves the conflict. If not, the tool reports the
conflict for manual review."
```

### Error message

```
Before:
"An error may have occurred while processing your request due to a possible
mismatch in the expected data format, which could be caused by an outdated
client version."

After:
"The request failed. The data format did not match what the server expected.
Check your client version — an outdated client is the most common cause."
```

### System prompt instruction

```
Before:
"When considering whether to use a tool, you should take into account the
various constraints that may be applicable in the current context, including
but not limited to rate limits, permissions, and whether the operation could
have side effects that are difficult to reverse."

After:
"Before you use a tool, check three things:
1. Whether the rate limit allows the call.
2. Whether you have permission to run the operation.
3. Whether the operation is reversible.
Do not call the tool if any check fails."
```

---

## Applying This to Your Work

Three places where these rules have the highest return:

**Tool descriptions in AI systems** - the description is what the model reads when deciding whether and how to call a tool. Ambiguous descriptions cause wrong calls and wrong argument values. Apply the full structural ruleset here.

**Error messages returned to agents** - an agent parsing an error message decides whether to retry, fall back, or escalate based on what the message says. "An error occurred" is not actionable. "The upstream service returned 503. The request timed out after 30 seconds." is.

**Inter-agent instructions** - when one agent hands instructions to another (in a multi-step pipeline, a workflow, or a handoff message), the receiving agent has no context beyond what the message says. One instruction per sentence. Named actors. No synonym rotation.

---

## The Claude Code Skill

The [asd-ste100-skill](https://github.com/danyuchn/asd-ste100-skill) is a Claude Code skill that applies these rules on demand. Install it:

```bash
git clone https://github.com/danyuchn/asd-ste100-skill ~/.claude/skills/asd-ste100
```

Then trigger it by asking Claude to rewrite text:

```
Disambiguate this tool description
Rewrite this error message so an agent cannot misparse it
Apply STE100 to this instruction
```

The skill operates in two modes:

**Strict** - for tool descriptions, error messages, inter-agent instructions, and system prompts. All structural rules enforced, length caps applied.

**STE-flavored** - for READMEs, PR descriptions, and explanatory prose. Same structural rules, but the fixed-vocabulary lockdown is relaxed because prose needs some range.

Default output is the rewritten text only, nothing else. Add "show the diff" or "which rules did it break" to get a before/after table naming each rule that was violated.

One limit worth stating clearly: this fixes the form of a text, not its content. A vague instruction rewritten under these rules becomes a short, clean, unambiguous vague instruction. If the text has nothing precise to say, no rewrite fixes that.

---

### For AI agents

```
ASD-STE100 for agent-facing text: aerospace controlled-language standard adapted for AI agent consumption. Core structural rules (apply these without the official dictionary): active voice, one instruction per sentence, max 20 words (procedures) / 25 words (descriptions), no semicolons, no phrasal verbs (use plain single verb), noun clusters max 3 words, no nominalization, no marketing adjectives. Critical: preserve modality - "may have failed" must not become "failed"; hedges are information, not noise. Apply to: tool descriptions, error messages, system prompts, inter-agent instructions. Claude Code skill at github.com/danyuchn/asd-ste100-skill - install to ~/.claude/skills/asd-ste100, trigger with "disambiguate this" / "apply STE100". Two modes: Strict (all rules) and STE-flavored (structural only, for prose).
```

Reference: `https://michalsniezko.github.io/ai-tooling/ste100-agent-output.html`
