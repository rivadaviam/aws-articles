---
title: "Fowler Named It SPDD. AWS Already Built the Runtime for It."
published: false
description: "Martin Fowler gave prompts-as-artifacts a name. Bedrock Prompt Management is the native way to run it. Almost nobody connects the two."
tags: aws, amazonbedrock, generativeai, softwareengineering
cover_image: ""
canonical_url: ""
---

When Martin Fowler publishes something on martinfowler.com, engineers read it. He named refactoring. He named continuous integration. He helped the industry take microservices seriously. So when he published "Structured-Prompt-Driven Development" in April 2026, engineers paid attention. He'd put a name on a practice the best teams already run by instinct.

The idea is simple. A prompt is not a chat message. It's a delivery artifact. It deserves authorship, review, versioning, regression tests, and a record of why it was written the way it was. Treat it like anything else you ship.

Here's the part almost nobody is saying out loud. AWS already built the runtime for that idea, and it's been sitting in the Bedrock console the whole time. It's called Bedrock Prompt Management. Fowler gave the methodology a name. AWS gave it versions, aliases, metadata, and an audit trail. The theory and the service describe the same thing from two directions, and I've never seen anyone hold them up side by side.

So let me do that.

---

## The prompt in the hardcoded string

This one has no invoice attached. The cost is a moment: someone asks a simple question and nobody in the room can answer it.

You've seen this project. An engineer writes a prompt. Maybe it lives in a config file. More likely it's a triple-quoted string buried three functions deep in the code that calls the model. It works. It ships. Everyone moves on.

Weeks later, someone "improves" it. A word here, a reordered instruction there, a new line about output format. The behavior of the agent shifts. Not dramatically. Just enough that support tickets tick up and a downstream parser starts choking on responses it used to handle fine.

Now the question: what changed, and why was it written the old way in the first place?

Good luck. The prompt lived in a string, so its history lives in the code's git blame, tangled with unrelated refactors. The intent was never documented, because a string in a function doesn't invite a paragraph explaining the reasoning. Debugging means reconstructing the original author's thinking from scratch. Sometimes that author is you, six months ago, and you don't remember either.

I version prompts in my own harnesses every day. Not because a framework told me to. Because I got tired of that exact question having no answer. When a change makes an agent worse, I want to diff the prompt before I touch a single line of code. That instinct is the whole thesis of SPDD, and it's the reason this landed for me the moment I read Fowler's piece.

---

## What SPDD actually claims

Strip it down and SPDD makes a small number of claims, each one borrowed from a practice engineers already trust.

Prompts are first-class. They get versioned, reviewed, and documented the same way code does. The prompt is the source, not an afterthought pasted next to the source.

When reality diverges, fix the prompt first. If the model's output isn't what you expected, your first hypothesis is the prompt, not the surrounding code. <!-- VERIFY: exact wording of this principle from Fowler's article — https://martinfowler.com/articles/structured-prompt-driven/ --> That ordering matters more than it sounds. It flips the default debugging reflex.

Prompts get regression tests. A prompt change proves itself against a test suite before it reaches production, the same gate any other behavioral change passes through.

Intent gets documented. The prompt records why it was designed this way, not just what it does. It's an ADR for a prompt.

Fowler also proposes a structured canvas for writing the prompt itself, an acronym he calls REASONS. It breaks a prompt into named sections instead of one wall of free text: requirements, entities, and reasoning approach are the parts I'm confident about, with style, output spec, and safety guardrails among the rest.

<!-- VERIFY: the seed captured only the first components of REASONS and flagged the acronym as partial. Confirm all seven sections and their exact names against https://martinfowler.com/articles/structured-prompt-driven/ before publishing. Do not assert the full expansion until verified. -->

I'm deliberately not spelling out all seven letters here, because my source for the full acronym is second-hand and I'd rather flag the gap than fake precision. The shape of the idea is what matters: a prompt written as a document with sections beats a prompt written as a paragraph. If you've ever structured a good system prompt, you've done a rough version of this already.

---

## Mapping SPDD onto AWS

This is where it gets concrete. Take each SPDD principle and ask: does a native AWS service already implement it? For most of them, the answer is yes.

| SPDD principle | What Fowler is asking for | Where it lives in AWS |
|---|---|---|
| Prompts as first-class | Versioned, reviewed, documented like code | Bedrock Prompt Management — every prompt gets versions, aliases, and metadata |
| Fix the prompt first | Compare behavior across prompt revisions | Prompt versions stay immutable, so you can diff and roll back per version |
| Structured authoring (REASONS) | A prompt is a document with sections, not free text | The prompt object holds system text plus variables and configuration as structured fields |
| Regression tests | Changes pass a test suite before production | AgentCore Evaluations with Ground Truth to score behavior before a deploy |
| Documented intent | Record why the prompt exists, not just what it does | The `description` and `tags` fields on the prompt resource |
| Auditable change | Know who changed what, and when | CloudTrail logs on the Bedrock API calls |

The mapping isn't forced. Bedrock Prompt Management was designed around versioning and aliases from the start, which happens to be exactly the primitive SPDD needs. A version is an immutable snapshot of a prompt. An alias is a movable pointer, like `production` or `staging`, that names which version is live right now. Change the pointer, and traffic moves. That's the whole trick, and it's the same trick container image tags and DNS records have used for years.

---

## The deployment flow, end to end

Here's what a prompt change looks like when the prompt is a real artifact instead of a string.

A developer writes a new version of the prompt in Bedrock Prompt Management. It becomes version N, immutable, sitting next to versions 1 through N-1. Nothing in production has moved yet.

That change opens a pull request. Not of code, of intent. The diff shows what changed in the prompt text, and the PR description explains why. This is the code review Fowler wants for prompts, and it reads exactly like a normal review, because a reviewer can see the before and after.

AgentCore Evaluations runs the regression suite against version N. <!-- VERIFY: confirm AgentCore Evaluations can target a specific Bedrock prompt version as its subject-under-test, and whether it can diff two prompt versions' outputs automatically. Seed lists this as an open question. --> If the scores hold, the `staging` alias moves to point at version N. QA exercises staging against the new version with zero code deployed.

If staging looks good, the `production` alias moves to version N. That's the deploy. No rebuild, no container push, no redeploy of the calling service. The application code never changed. It always asks Bedrock for "the production prompt," and Bedrock hands back whatever version the alias points at right now.

CloudTrail records the alias change: which identity moved which alias to which version, and when. The question that had no answer at the top of this article now has one, and it's a log line.

Rollback is the same move in reverse. Point `production` back at version N-1. The old prompt was never deleted, so there's nothing to rebuild. You're back, fast, without touching a deploy pipeline.

```text
Prompt change lifecycle in Bedrock

  write ──▶ version N (immutable)
             │
             ├── PR: prompt diff + intent
             ├── AgentCore Evaluations vs. suite
             │
  pass ──────▶ alias "staging"  ─▶ version N   (QA, no code deploy)
             │
  approve ──▶ alias "production" ─▶ version N   (live, zero-downtime)
             │
  regress ──▶ alias "production" ─▶ version N-1 (rollback, no rebuild)

  CloudTrail logs every alias move: who, what, when.
```

[GRAPHIC: architecture diagram | the write → version → PR → evaluations → staging alias → production alias flow, with CloudTrail logging alias moves and a rollback arrow back to N-1 | "A prompt deploy is an alias move, not a code deploy."]

I'll be honest about a seam here. Whether Bedrock Prompt Management wires directly into CodePipeline for this flow, or whether you script the alias moves yourself with the SDK in a pipeline stage, is something I'd confirm before promising a turnkey setup. <!-- VERIFY: native integration between Bedrock Prompt Management and CodePipeline for automated prompt deploys — seed open question, not yet confirmed. --> The primitives are all there either way. The alias update is a single API call, and a single API call is trivial to drop into whatever CI you already run.

---

## Why this earns its place only at scale

I want to push back on my own article for a second, because the brand rule here is real: don't gold-plate.

A team with one agent and two prompts does not need any of this. You can hold two prompts in your head. Versioning them through a managed service is ceremony you'll resent. If that's your stack, close the tab and go ship.

The math changes fast. Ten agents, fifty prompts, three environments, and a couple of engineers who each "improve" prompts on their own schedule. Now the hand-managed approach stops being merely annoying and becomes the thing that breaks production at 2 a.m. with no audit trail to explain it. SPDD is an answer to operational scale, and Bedrock Prompt Management is the operational tool. Neither is worth the weight until the weight is worth carrying.

There's a tell that you've crossed the line: the first time someone asks "when did this prompt change?" and the honest answer is "no idea." That's the signal. That's when a string in a function became technical debt, in exactly the sense Fowler has written about for twenty years.

[GRAPHIC: comparison chart | hardcoded-string workflow vs. Bedrock-managed workflow across five rows — where the prompt lives, how it changes, rollback path, audit trail, review process | "Same prompt. One is an artifact, one is a liability."]

---

## The Well-Architected angle

One more connection, because it gives SPDD a home in a framework AWS teams already answer to. Treating prompts as first-class artifacts lines up cleanly with the Operational Excellence pillar of the Well-Architected Framework. Documented runbooks, reviewed changes, planned rollback. That pillar has asked for those three things about your infrastructure for years. SPDD just extends the same discipline to the prompts, which are now a load-bearing part of your system's behavior.

If you've ever justified an infrastructure practice by pointing at Operational Excellence in a review, you already have the vocabulary to justify prompt management the same way. Same argument you've been making for years. It just applies to a new kind of source file now.

---

## What I'd actually do Monday

Not a migration. A single prompt.

The low-commitment version: pick one prompt that already caused a mystery and move it into Bedrock Prompt Management as version 1. Add a `description` that says why it exists. That's it. You now have one prompt that can answer the question the rest can't.

The medium version: put a `staging` and `production` alias on it, and route your app to the alias instead of the hardcoded string. Next time you change that prompt, you cut a version and move the pointer. Feel how a prompt deploy without a code deploy changes your risk appetite.

The high version: wire an evaluation gate in front of the alias move, so no version reaches `production` without passing a regression suite. That's full SPDD on AWS, and it's the setup I'd want the day a fifty-prompt stack is keeping me up at night.

I'm building toward that last version in my own harnesses, one prompt at a time, and documenting the seams as I hit them. If you've already run prompt regression tests in production, or found where the CodePipeline integration does or doesn't exist natively, I want to hear it. Drop it in the comments. The gaps I flagged with VERIFY notes above are genuine open questions, and the fastest way to close them is someone who's already been there.

Build. Document. Share. Repeat.

---

**Sources**

- Martin Fowler, "Structured-Prompt-Driven Development," martinfowler.com, April 2026 — https://martinfowler.com/articles/structured-prompt-driven/
- AWS Bedrock Prompt Management — https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-management.html
- AWS AgentCore Evaluations (GA, March 2026) — https://aws.amazon.com/about-aws/whats-new/2026/03/agentcore-evaluations-generally-available/
- AWS CloudTrail — https://aws.amazon.com/cloudtrail/
- AWS Well-Architected Framework (Operational Excellence) — https://aws.amazon.com/architecture/well-architected/
- AWS CodePipeline — https://aws.amazon.com/codepipeline/
