---
title: "The CEO of Y Combinator Open-Sourced a 6-Agent Team. What Happens When You Put It on AWS Bedrock?"
published: false
description: "gstack installs in 30 seconds on your laptop. The same pattern on Bedrock takes hours and adds IAM, CloudTrail, and billing. Here's when that trade is worth it."
tags: aws, aiagents, amazonbedrock, learninpublic
cover_image: "https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/gstack-on-bedrock-multiagent/00-cover.png"
canonical_url: ""
---

Garry Tan runs Y Combinator. He also just open-sourced his Claude Code setup, and it isn't a clever alias or a tidy prompt. It's a team.

gstack is six specialized agents that turn a single conversation into a pipeline of roles. A CEO agent asks why the thing should exist before anyone writes code. An engineering manager sets the architecture. A designer generates variants and picks one. A release manager runs the ship cycle. You install it in about 30 seconds, and suddenly your one-person project has a chain of command.

That's the part worth sitting with. When the person running the most influential accelerator on earth ships his multi-agent workflow as a public dotfile, the "is multi-agent real or just demo-ware" question is settled. It's real. People do actual work this way.

I run multi-agent setups in Claude Code every day, so none of this surprised me. What caught my attention was the gap nobody talks about. "Install in 30 seconds" quietly means "on your laptop, with your credentials, no audit trail, no per-role access control, no shared bill." For a solo developer that's perfect. For a company that wants the same pattern in production, it's the start of a different problem.

This article is about that gap. What does gstack look like when you rebuild it on AWS Bedrock, and when is the extra work actually worth it?

<!-- VERIFY: seed captured only 4 of gstack's 6 roles clearly (CEO, Engineering Manager, Designer, Release Manager). Confirm the exact 6 against the repo README before publish: https://github.com/yc-oss/gstack -->

---

## gstack is the pattern, not the code

Let me be precise about what I'm porting. I'm not lifting Garry Tan's files onto AWS. gstack is written for Claude Code, and it belongs there. What travels is the idea: a small set of specialized agents, each with one job, coordinated so the output of one feeds the next.

There's a useful frame going around for this. Someone mapped out five levels of Claude Code usage, roughly Prompt, Skill, Skill Chain, Agent, and Agent Team, with the payoff climbing at each step. Level five, the agent team, is where the big multiplier lives. gstack is a level-five artifact. It's a team, not a trick.

<!-- VERIFY: the "5 levels / payoff multiplier" claim comes from an Instagram post (@leadgenman) cited in the seed. Treat as a framing device, not a benchmark. Keep the language qualitative, no specific multiplier numbers stated as fact. -->

So the real question was never "how do I copy gstack." The question is: I like level five, so how do I run it where a team of engineers, an auditor, and a finance lead can all live with it? That's the Bedrock conversation.

---

## Mapping the six roles to AWS services

Bedrock has a multi-agent collaboration model built for exactly this shape: one orchestrator agent that routes work to specialized sub-agents. That maps onto gstack's roles almost one to one. The interesting part is which AWS service backs each role, because that's where the local pattern gains its production teeth.

Here's how the four well-documented gstack roles line up.

| gstack role | What it does | AWS Bedrock equivalent |
|---|---|---|
| CEO | Validates the business case before any code ("why does this need to exist?") | Sub-agent fronted by Bedrock Guardrails to filter requests with no business justification |
| Engineering Manager | Sets architecture up front | Sub-agent backed by a Knowledge Base of your internal architecture, plus the Well-Architected Tool |
| Designer | Generates 4-6 variants, picks the best | Parallel model invocations, scored with AgentCore evaluations |
| Release Manager | Runs the release cycle | Sub-agent with CodePipeline and CodeBuild access through an MCP server |

Forget the specific service names for a second. What the table really shows is a shift in what "a role" means. On your laptop, the CEO agent is a system prompt. On Bedrock, that same agent is a system prompt plus a guardrail policy plus an IAM role that literally cannot touch your deploy pipeline. The role stops being a suggestion. It becomes an enforced boundary.

![Bedrock orchestrator agent routing to CEO, Engineering Manager, Designer, and Release Manager sub-agents, with a CloudTrail, CloudWatch, and IAM audit floor underneath](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/gstack-on-bedrock-multiagent/01-orchestrator-architecture.png)

*The orchestrator routes to each specialized sub-agent, and every one of them runs on top of the same audit floor: CloudTrail for decisions, CloudWatch for metrics, IAM for least-privilege access.*

---

## The architecture, in one picture

Strip away the service logos and the Bedrock version looks like this:

```
User / Event
     │
     ▼
Bedrock Orchestrator Agent  (the coordinator)
     ├── CEO Agent                  → Guardrails + Claude
     ├── Engineering Manager Agent  → internal KB + Claude
     ├── Designer Agent             → parallel invocations + evaluations
     ├── Release Manager Agent      → CodePipeline via MCP + Claude
     └── [roles 5 and 6]            → see VERIFY note above
     │
     ▼
CloudTrail   → every tool call and agent decision, logged
CloudWatch   → latency and token metrics per agent
IAM          → each agent gets its own least-privilege role
```

The three lines at the bottom are the whole reason to do this. On a laptop, all six agents run as *you*. They share your credentials, your shell, your access to everything. On Bedrock, the Release Manager agent can reach CodePipeline and nothing else. The CEO agent can read the request and write a verdict, but it can't deploy. If someone asks later "why did the system approve this change," CloudTrail has the answer with a timestamp.

That's not a feature you notice on day one. It's the feature you're grateful for during the incident review six months later.

---

## What actually changes: local vs. Bedrock

This is the part the migration blog posts skip. The pattern is identical. The operational reality is not.

| Dimension | gstack local (Claude Code) | gstack pattern on Bedrock |
|---|---|---|
| Audit | No trail. You can't reconstruct what each agent decided | CloudTrail logs every tool call and decision |
| Access control | Everything runs with the developer's credentials | Each agent gets its own IAM role, least privilege |
| Billing | Individual subscription | Centralized in AWS, visible by team, project, environment |
| Scale | One developer, one session | Many developers, parallel sessions, no collision |
| Governance | No formal guardrails | AgentCore policy plus Guardrails on each agent |
| Integration | The developer's local tools | AWS MCP server for any AWS service |

Read that table as a translation of one word: *ownership*. Local gstack is owned by a person. Bedrock gstack is owned by an organization. Everything in the right column is what "owned by an organization" costs and buys.

---

## The 30 seconds vs. the several hours

gstack's pitch is "install in 30 seconds." That's true, and it's the right pitch for what it is.

The Bedrock version has no such pitch. You're defining IAM roles, writing agent definitions, standing up a Knowledge Base, wiring MCP integrations, and turning on CloudTrail. That's not a 30-second job. Anyone who tells you otherwise is selling something.

I'm not going to hand you a fake stopwatch number here, because I haven't run this exact port end to end, and inventing a duration would be the kind of thing this whole series exists to push back against. What I can tell you from running multi-agent work daily is where the time goes. It goes into IAM. It always goes into IAM. The agents are the easy part. The least-privilege role for each one, tested so it can do its job and nothing more, is the work.

<!-- VERIFY: no time estimate stated on purpose (brand rule: no time estimates; author has not run this specific port). If a real hands-on port is done later, add measured setup time here. -->

So is it worth it? That depends entirely on who's asking.

If you're a solo developer or a two-person startup, almost certainly not. The audit trail protects you from a compliance question nobody is asking yet. The per-agent IAM roles guard against a blast radius that, at your size, is just your own laptop. Stay on gstack. Ship. The 30 seconds is the correct answer.

If you're a team of five or more engineers, or anyone with a compliance obligation, the math flips. The moment "which agent approved this deploy" becomes a question a real human has to answer, CloudTrail stops being overhead and starts being the cheapest insurance you own. The moment two developers run the pattern at once, shared IAM and centralized billing stop being bureaucracy and start being the thing that keeps the wheels on.

![Two-column comparison: for a solo developer or early startup, Bedrock's audit, access control, billing, and scale features are overhead; for a team of five or more with compliance needs, the same features are insurance](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/gstack-on-bedrock-multiagent/02-solo-vs-team.png)

*Same six agents, same four dimensions. On the left they read as overhead; on the right, as insurance. The scale is what flips the answer.*

The dishonest version of this article sells Bedrock as the universal upgrade. It isn't. It's the right tool at a specific scale, and naming that scale out loud is more useful than another "why you should migrate everything" post.

---

## What I'd tell you before you start

A few things I'd want to know before rebuilding this pattern, learned from running agent teams rather than from this specific port.

The orchestrator is where your design lives or dies. gstack's magic is the handoff between roles, and on Bedrock that handoff is the orchestrator's routing logic. Get that wrong and you have six agents talking past each other instead of a pipeline. Spend your design budget there, not on the individual agents.

Guardrails are not a nice-to-have on the CEO agent, they're the entire point of it. The whole reason a "business validation" agent earns its slot is that it says no. If its guardrail is loose, it's a rubber stamp with a token bill attached.

And least privilege is the feature, not the tax. It's tempting to give every agent broad permissions to get things working, then tighten later. Later never comes. If you're going to pay the setup cost of Bedrock at all, the per-agent IAM boundary is the thing you're paying for. Don't skip it and keep the overhead.

Build. Document. Share. Repeat. This series was never here to tell you Bedrock wins. It's here to show the actual trade, so you can make the call for your own team, at your own scale.

---

## Your turn

Three ways to take this further, pick your commitment level.

**Low:** Go read gstack. It installs in 30 seconds and it'll change how you think about a single Claude Code session, whether or not you ever touch Bedrock. Then tell me in the comments which of the six roles you'd actually keep. I suspect not everyone needs all six.

**Medium:** If you've mapped a multi-agent pattern onto Bedrock's multi-agent collaboration, I want the part I left out: what did the setup actually cost you in time and dollars? I skipped the stopwatch on purpose. Yours would make this article better.

**High:** Do the real port. Take the pattern, build the orchestrator plus the four documented agents on Bedrock, log every decision to CloudTrail, and write up where the design fought you. That's the article I'd read next, and if you write it, I'll link it here.

The pattern is Garry Tan's. Full credit to him and to gstack for making level-five agent teams something you can install instead of theorize about. The question of where to run it is the one worth arguing over.

---

*Sources: Garry Tan's gstack, covered by Charlie Hills on LinkedIn; the five-levels framing from @leadgenman on Instagram; [Amazon Bedrock Agents](https://aws.amazon.com/bedrock/agents/), [AgentCore](https://aws.amazon.com/bedrock/agentcore/), [CloudTrail](https://aws.amazon.com/cloudtrail/), and [IAM](https://aws.amazon.com/iam/).*

#BuildToLearn #AWSCommunity #LearnInPublic
