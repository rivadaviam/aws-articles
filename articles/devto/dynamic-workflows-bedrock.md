---
title: "When Your Agent Writes Its Own Orchestration Code (and You Run It on AWS)"
published: false
description: "Claude Code stopped improvising in chat and started generating repeatable programs. Here's what changes the moment you point that at Bedrock, Lambda, and EventBridge."
tags: aws, claudecode, aiagents, generativeai
cover_image: ""
canonical_url: ""
---

Most of my Claude Code sessions used to end the same way. The agent did the thing, I got the answer, and the answer evaporated into scrollback. Next week I'd need the same thing again, and the agent would rebuild it from scratch. Different phrasing, different steps, slightly different result. Every run started at zero.

Then Dynamic Workflows landed in Claude Code v2.1.154+, and the shape of the interaction changed. The agent stopped handing me a response. It started handing me a program.

That sounds like a small thing. It isn't. A response is something you read once. A program is something you keep, review, version, and run a thousand times. And once the artifact is a program instead of a chat message, a question shows up that nobody was asking yet: what happens when that self-generated program runs on AWS instead of on my laptop?

That's the whole article. I want to be honest up front about where the ground is solid and where I'm reasoning ahead of my own hands-on. One part of this already works today. Two parts are the direction it's clearly heading, and I'll flag which is which as we go.

---

## The difference between improvising and programming

Here's the distinction that reframed how I think about agents.

An agent that improvises takes a prompt, generates an ad-hoc answer, and finishes. The knowledge lives in that one exchange. Ask again tomorrow and it does the reasoning over from the start, with no memory that it ever solved this before.

An agent that programs does something different. It generates explicit orchestration code, shows you the plan before running anything, executes in the background so your chat doesn't lock up, saves the result, and leaves the whole thing behind as a reusable slash command. That's the flow Dynamic Workflows introduced (per @pabloinpublic, who documented the feature across Instagram and LinkedIn in May–June 2026). The key line from that coverage stuck with me: the agent generates a program, not a response.

So the plan becomes visible. You read what the agent intends to do before it does it. You approve, it runs detached, and the workflow persists. Run it once, it's a program. Run it next month, it's the same program.

That persistence is the thing worth chasing onto AWS.

<!-- VERIFY: Dynamic Workflows was in research preview at INBOX capture time (per @pabloinpublic, May–June 2026). Confirm GA status and exact v2.1.154+ availability before publishing. Also confirm whether the generated orchestration artifact is Python, bash, or a custom DSL — the seed lists this as an open question. -->

---

## Three levels of depth

I found it useful to break this into three levels. Level 1 runs today. Level 2 is a manual pattern you could build now. Level 3 is where the loop starts closing on itself, and it's the most speculative of the three.

### Level 1 — Bedrock as the backend (this works today)

You can already run Claude Code with Amazon Bedrock as the model backend instead of the Anthropic API directly. Prasad Rao, an AWS Principal SA, walked through this on LinkedIn in May 2026. The mechanics: IAM plus SigV4 for auth, [CloudTrail](https://aws.amazon.com/cloudtrail/) logging every model call, and billing that lands in your AWS account alongside everything else.

The Dynamic Workflows the agent generates still run in your local dev environment at this level. The model just gets served from [Bedrock](https://aws.amazon.com/bedrock/). That alone solves two problems enterprise teams actually care about: the audit trail (every call is a CloudTrail event) and data residency (the traffic stays inside your AWS boundary).

Nothing exotic here. It's Claude Code, wired to a backend your security team already trusts. If you're evaluating Claude Code for a regulated environment, this is the entry point, and it's real.

### Level 2 — The workflow becomes a Lambda (the jump)

This is the conceptual jump, and I want to be clear that this part is a pattern I'm describing, not a production system I've shipped.

The orchestration program that Claude Code generates is code. Regular code. So there's nothing stopping you from taking that code, reviewing it (you already saw it before approving the run), and packaging it as an [AWS Lambda](https://aws.amazon.com/lambda/) handler. Deploy it into your account, and instead of triggering it by typing a slash command, you trigger it with [EventBridge](https://aws.amazon.com/eventbridge/).

The path looks like this:

```
Claude Code (v2.1.154+, via Bedrock)
  → generates orchestration_program.py
  → you review and approve
  → you package it as a Lambda handler
  → CDK or Terraform deploys the Lambda
  → an EventBridge Rule triggers it on schedule or event
  → CloudWatch logs every execution
  → the workflow is now permanent infrastructure
```

Follow that chain and something clicks. What started as a Claude Code session ended up as a piece of production infrastructure that runs without you in the loop. The agent's reasoning got frozen into a deployable artifact.

Here's a sketch of what the wrapping looks like. The agent hands you the workflow logic. You wrap it in a handler and give it a schedule.

```python
# lambda_function.py
# The workflow body is what Claude Code generated and I approved.
# I'm only wrapping it in a handler + wiring it to a trigger.
from workflow import run_workflow  # the agent's orchestration program

def handler(event, context):
    # event carries whatever EventBridge sends (schedule tick,
    # S3 event, custom bus event). The workflow stays trigger-agnostic.
    result = run_workflow(payload=event)
    return {"status": "ok", "result": result}
```

```python
# infra.py — CDK, trimmed to the load-bearing parts
# Why EventBridge instead of a slash command? A slash command needs me
# at a keyboard. A rule runs at 6am whether I'm awake or not.
schedule_rule = events.Rule(
    self, "WorkflowSchedule",
    schedule=events.Schedule.cron(minute="0", hour="6"),  # daily 06:00 UTC
)
schedule_rule.add_target(targets.LambdaFunction(workflow_fn))
```

The human review step is not decoration. You read the generated program before it becomes a Lambda that fires every morning. That's the gate, and I'll come back to why the gate matters.

### Level 3 — The loop that propagates itself (the frontier)

Now the speculative part. This is reasoning projected forward, not something I've run, and I'd rather say that plainly than dress it up as certainty.

The [AWS MCP Server went GA](https://aws.amazon.com/blogs/machine-learning/introducing-aws-mcp-servers-for-code-assistants/) in late May 2026 (reported by Renato Losio at InfoQ, May 24). It lets an agent deploy AWS resources directly, gated by IAM. Put that next to an agent that can already generate orchestration programs, and the manual steps in Level 2 start to collapse.

At that point the agent could, in principle: generate the workflow, deploy it as a Lambda through the AWS MCP Server, create the matching EventBridge Rule, and leave it running while it reports to CloudWatch. No human packaging step. The agent generates its own execution infrastructure and stands it up.

That's a real step past "an agent that does tasks." It's an agent that manufactures the machinery that does the tasks, then walks away. I find that equal parts exciting and slightly alarming, which brings us straight to the question I care about most.

---

## Why this matters more than it looks

Go back to the improvising agent for a second. Its real weakness is that every execution is disposable. The agent solves something, finishes, and the knowledge is gone or trapped in a chat log nobody will reopen.

Dynamic Workflows breaks that. The agent's knowledge crystallizes into code. And code, on AWS, gets to live the same life as any other piece of infrastructure. You version the workflows in your source control. You test them in CodeBuild. You watch them in [CloudWatch](https://aws.amazon.com/cloudwatch/). You audit them in CloudTrail. A workflow the agent generated once inherits the entire lifecycle you already built for everything else you deploy.

That's the payoff, and it's quieter than "the agent is smarter." The agent's output finally sticks around long enough to compound.

---

## The governance question I keep circling back to

If an agent can generate and potentially deploy its own infrastructure in your account, who reviews it? Who approves it?

This is the part that would keep me up at night if the answer were bad. It isn't bad, and the answer is reassuringly boring: the gate was never supposed to be the agent's judgment. The gate is [IAM](https://aws.amazon.com/iam/).

The agent can generate whatever it wants. It can only deploy what its IAM role permits. On top of that, [AgentCore Policy](https://aws.amazon.com/bedrock/agentcore/) (AWS What's New, March 2026) controls which tools each agent is even allowed to call. So you get two layers. AgentCore Policy decides what the agent may attempt, and IAM decides what actually lands. Neither of them trusts the model's good intentions, which is exactly right.

That reframes the whole Level 3 anxiety. The scary version is "the agent deploys anything it dreams up." The real version is "the agent proposes, IAM disposes." A Lambda that self-deploys is only as dangerous as the role you scoped for it. If the role can't touch production, the agent can't either, no matter how confidently it generated the code.

So the question stops being "can I trust the agent?" and becomes "did I scope the role correctly?" That's a question I already know how to answer. I've been answering it for every human on the team for years.

---

## Where the receipts run out

I said I'd be honest about the boundary, so here it is in plain terms.

Level 1 is proven. Claude Code on Bedrock, IAM auth, CloudTrail logging, billing in your account. That's documented and running today.

Level 2 is a pattern I've reasoned through but not yet shipped as a real deployment. Every piece exists (Lambda, EventBridge, CDK, the generated program). I haven't personally taken one Dynamic Workflow all the way to a scheduled Lambda in a real account and watched it fire for a week. I looked for a published example of someone doing exactly that and came up empty, which is honestly part of why I wanted to write this: the gap is where the interesting work lives.

Level 3 is projection. The primitives are GA. The clean end-to-end loop is where things are heading, not a thing I can point at and say "here, run this."

Dressing projection up as certainty would be the easy move and the wrong one. The value here is drawing the line clearly. This works. This should work. This is coming. Treating all three as the same confidence level is how you end up shipping hype with extra steps.

---

## What I'm doing next

The obvious follow-up is the hands-on: take one real Dynamic Workflow, package it as a Lambda, wire the EventBridge Rule, and report back with the actual bill and the actual failure modes. That's the Part 2 I want to write, and it'll have receipts instead of reasoning.

If you want to move on this before I get there, here are three on-ramps by effort.

Low: run Claude Code against Bedrock instead of the Anthropic API this week. Level 1 is real and it costs you an afternoon of IAM setup. You'll get the CloudTrail audit trail for free.

Medium: next time a Dynamic Workflow generates something you'd want to run on a schedule, don't let it evaporate. Save the program, read it properly, and sketch the Lambda wrapper. Even if you don't deploy, you'll feel the shift from "chat output" to "deployable artifact."

High: build the Level 2 path end to end and publish it. Scope a tight IAM role, deploy one workflow, schedule it, watch it in CloudWatch for a week. If you beat me to the real receipts, link it in the comments and I'll point people at your work instead of my theory.

I'm chasing the same thing you are here, so if you've already taken a self-generated workflow to production infrastructure, I genuinely want to read it. Build. Document. Share. Repeat.

---

*Sources: Dynamic Workflows coverage by @pabloinpublic (Instagram/LinkedIn, May–June 2026). Claude Code on Bedrock via Prasad Rao, AWS Principal SA (LinkedIn, May 2026). AWS MCP Server GA via Renato Losio, InfoQ (May 24, 2026). AgentCore Policy via AWS What's New (March 2026). Service references: [Bedrock](https://aws.amazon.com/bedrock/), [Lambda](https://aws.amazon.com/lambda/), [EventBridge](https://aws.amazon.com/eventbridge/), [CloudTrail](https://aws.amazon.com/cloudtrail/), [CloudWatch](https://aws.amazon.com/cloudwatch/), [IAM](https://aws.amazon.com/iam/), [AgentCore](https://aws.amazon.com/bedrock/agentcore/).*
