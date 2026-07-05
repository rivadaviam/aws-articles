---
title: "From Demo to Production: What GA Actually Means for AWS AI Agents"
published: false
description: "My $363 Bedrock bill taught me to ask about cost before the invoice. Here's what changes when agents run 24/7 and the meter never stops."
tags: aws, amazonbedrock, aiagents, generativeai
cover_image: "https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/from-demo-to-production-aws-agent-stack/00-cover.png"
canonical_url: ""
---

Two days. ~$363. That was my Knowledge Base lesson.

I was re-syncing a Bedrock Knowledge Base over and over, tuning it, watching it fail, fixing it, syncing again. Every sync re-embedded the documents. Every re-embed spent tokens. The tokens stacked up quietly in the background while I stared at retrieval quality, not the meter. The bill showed up two days later and taught me a rule I still use: ask about cost *before* the invoice, not after.

I bring that up because the AWS story in 2026 is no longer about picking a model. It's about running agents in production, for hours, sometimes days, on their own. And production is exactly where quiet meters, hidden quotas, and "we'll figure out cost later" turn into real money. The demo is free. Production sends you a bill.

So let me look at what "GA" actually means this year, using the specifics AWS published, not the keynote adjectives.

---

## The pitch changed from "try this model" to "put agents to work"

At re:Invent 2025, the framing shifted. AWS stopped leading with "here's a better model" and started leading with agents that operate as an extension of your team: Kiro, a Security Agent, a DevOps Agent, described as "frontier agents" that work for hours or days at a time ([About Amazon](https://www.aboutamazon.com/news/aws/aws-re-invent-2025-ai-news-updates)).

That's a bigger claim than it sounds. A model call returns in seconds and you eyeball the result. An agent that runs for a day makes hundreds of decisions you never see, spends tokens you didn't budget line-by-line, and holds sessions open while it works. The moment an agent runs unattended, every question I learned to ask the hard way stops being optional. What does this cost? What are the limits? How do I know it's behaving?

The model catalog exploded in parallel. Bedrock went from roughly 60 to nearly 100 models, adding Mistral, Google, NVIDIA, OpenAI, MiniMax, Moonshot, and Qwen ([AWS News Blog](https://aws.amazon.com/blogs/aws/)). That includes GPT-5.5, GPT-5.4, and Codex running on Bedrock ([About Amazon](https://www.aboutamazon.com/news/aws/bedrock-openai-models)). Amazon's own Nova 2 line went multimodal end to end. Nova 2 Omni handles text, image, video, and voice, and Nova 2 Sonic does speech-to-speech ([AWS re:Invent 2025 top announcements](https://aws.amazon.com/blogs/aws/top-announcements-of-aws-reinvent-2025/)).

Here's what the model count doesn't tell you: none of it matters if you can't operate the thing safely. Which is why the announcements I actually care about aren't models at all.

---

## The GA list nobody puts on a slide

The interesting graduations this year are the boring-sounding ones. The plumbing that separates a prototype from something you'd let touch production data.

Three pieces of Amazon Bedrock AgentCore reached general availability:

| Component | What it does | Status |
|---|---|---|
| AgentCore **harness** | Takes you from idea to a running agent without writing your own orchestration | GA, June 2026 ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-bedrock-agentcore-harness-generally-available/)) |
| AgentCore **Evaluations** | 13 built-in evaluators plus Ground Truth to score agent behavior | GA ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2026/03/agentcore-evaluations-generally-available/)) |
| AgentCore **Policy** | Fine-grained control over which tools an agent can call, plus Guardrails | GA, March 2026 ([AWS What's New](https://aws.amazon.com/about-aws/whats-new/2026/03/policy-amazon-bedrock-agentcore-generally-available/)) |

Read that table again. Evaluation, policy, and guardrails going GA is the real signal — not another model. A market that's still doing demos doesn't ship evaluation frameworks. A market that's putting agents in front of customers does. AWS is telling you where it thinks the field is: past the "look, it works" phase and into "prove it works, and prove it stays inside the lines."

The harness matters because orchestration is where hand-rolled agents rot. You start with a clean loop: call model, run tool, feed result back. Six weeks later you're maintaining retry logic, session state, and timeout handling you never meant to own. A managed harness moves that burden to AWS. The trade-off, and I'll come back to this, is that you now depend on AWS's harness instead of your own.

Evaluations is the piece I'd reach for first, because it answers the question my $363 bill was really about: *how do you know?* Thirteen built-in evaluators plus Ground Truth means you can score an agent's output against expected answers instead of vibes. If you've ever shipped a RAG pipeline and "tested" it by asking it three questions you already knew the answers to, you know why this exists.

---

## The specifics that decide whether you can afford it

Now the part the keynote skips. Running an agent in production runs into limits, and the limits have numbers.

Two quotas stand out. AgentCore raised per-agent throughput from 25 to 200 TPS, and set the default for active session workloads at 5,000 per account. That 5,000 only applies in US East (N. Virginia) and US West (Oregon). Every other region gets 2,500 ([AgentCore release notes](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/release-notes.html)). Both are adjustable through Service Quotas, which changes the failure mode: you won't hit a wall, you'll hit a support ticket you didn't know you needed to file.

![Bar chart: AgentCore active session quota is 5,000 per account in US East and US West, but 2,500 in every other region](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/from-demo-to-production-aws-agent-stack/01-session-quota-by-region.png)
*The session cap nobody mentions in the keynote: deploy outside the two big US regions and you start with half the ceiling.*

Sit with those for a second. 200 transactions per second per agent is generous for most workloads. But if you're fanning out parallel tool calls, it's a ceiling you can hit. The active-sessions default is the one that bites quietly. An agent that "works for hours or days" holds a session the whole time. If your design spins up long-lived agents per user, per job, or per document, 5,000 concurrent isn't a lot. And if you deploy outside the two big US regions, you're working with half that. That's an architecture input, not a footnote. It's exactly the kind of number you want to know at design time — either you size under the default, or the quota-increase request goes in your launch checklist, not your incident retro.

This is where my near-miss from an earlier lab keeps echoing. In lab-01, my first design used one S3 bucket with two prefixes: input and output in the same bucket. Clean. Simple. Catastrophic. Lambda would have triggered on its own output, looped, and billed me into the thousands before I noticed. The docs saved me. I read the trigger scope carefully and split the buckets before I deployed. The lesson wasn't "S3 is dangerous." It was that autonomous systems find the failure mode you didn't design against. An agent running unattended for a day is that same class of problem, scaled up. It won't loop on an S3 event — it'll loop on a reasoning step, or hold sessions it never releases, or retry a failing tool 400 times. Same shape, bigger blast radius.

The cost picture is where I have to be honest with you, because the seed I built this from raised a question I can't fully answer yet: **what does an AgentCore agent actually cost in production versus a hand-rolled Lambda plus Bedrock call?** I don't have a real bill to show you. Anyone who hands you a confident number right now is guessing. What I *can* tell you is which meters are running: model tokens, session time, tool invocations, and any egress. The managed convenience of the harness is not free. If you take one thing from my Knowledge Base story: find the meters before you find the bill.

There's a data-residency angle worth flagging too. AgentCore's Web Search runs with zero data egress from your AWS environment ([AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/new-in-amazon-bedrock-agentcore-build-agents-with-broader-knowledge-and-continuous-learning/)), which matters if you're in a regulated shop and "the agent googled it" is not an acceptable data path.

---

## A minimum path from idea to production

If I were starting an agent on this stack today (and I'll be doing exactly that in an upcoming lab), here's the order I'd follow. Not because it's the only way, but because it front-loads the questions that cost me money when I ignored them.

![Flow diagram: the minimum path from idea to production — idea, harness, evaluate, policy, deploy, with the cost question asked at every gate](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/from-demo-to-production-aws-agent-stack/02-minimum-path.png)
*Five gates, one recurring question. The order matters: evaluation before scale, policy while the surface is small.*

Start with the idea and the harness. Get a single agent running with the managed harness before you touch anything fancy. One agent, one tool, one clear job. Prove the loop closes.

Then wire up Evaluations *before* you scale, not after. This is the reversal of how I did RAG the first time. Define your Ground Truth (the answers you expect) and let the 13 evaluators score the agent against them. If you can't measure it, you can't trust it running unattended.

Next, set Policy and Guardrails while the surface is small. Decide which tools the agent may call and what it may never do. Locking down tool access on a two-tool agent is a five-minute job. Retrofitting it onto a sprawling one is a project.

Only then think about deploy and scale. When you do, check your session math against that 5,000 cap and your throughput against 200 TPS *before* you turn on real traffic. Ask the cost question at this step, out loud, with numbers. Model tokens, session duration, tool calls, egress. Write them down.

That's the whole path: idea → harness → evaluation → policy → deploy, with the cost question asked at every gate instead of after the invoice. It's the Build-to-Learn loop applied to agents: build the smallest real thing, document the decisions, share what broke, repeat. Documentation isn't overhead. It's thinking made visible, and it's the thing that saved me in lab-01.

---

## What I'd measure in six months

So is this real infrastructure or another hype cycle? Honestly — both, and the split is the interesting part.

The plumbing is real. Evaluations, Policy, and Guardrails going GA is a market growing up. That's not marketing; that's AWS admitting the demo phase is over and betting on governance. I believe that part.

The claims I'd hold at arm's length are the autonomy claims. Nova Act is quoted at 90% reliability for browser automation ([About Amazon](https://www.aboutamazon.com/news/aws/aws-re-invent-2025-ai-news-updates)). Ninety percent sounds great until you ask ninety percent *of what*. I dug into it: the figure comes from Amazon's own internal evals of specific hard UI actions (date pickers, dropdowns, popups) plus early-customer workflows ([Amazon Science](https://labs.amazon.science/blog/nova-act)). Not an independent public benchmark. That doesn't make it fake; it makes it a number Amazon graded on Amazon's homework. And either way, a 10% failure rate on an agent that runs unattended for a day is a lot of failures nobody's watching.

There are open questions I genuinely can't answer yet, and I'd rather hand them to you than fake an answer:

- What does an AgentCore agent cost in production versus a Lambda-plus-Bedrock build you wire yourself?
- How does AgentCore compare to LangGraph, CrewAI, or the equivalents on Azure and Vertex, and what does choosing the AWS agent stack lock you into?
- Where are the *customer* case studies, not the launch posts?

If you have a real bill or a real deployment on any of these, I want to hear it. That's not a rhetorical close. Those numbers are the content I don't have yet.

---

## Your move

**Low commitment:** drop a comment with the one AgentCore quota or cost you wish AWS documented more clearly. I'll compile what comes back into a follow-up. Community-sourced FinOps beats guessing.

**Medium commitment:** if you've run AgentCore, LangGraph, or CrewAI in anything resembling production, tell me one thing that surprised you about the bill or the limits. One number is enough.

**High commitment:** build the minimum path above (one agent, evaluated, policy-locked) and write up what broke. Tag it and I'll link to it. The best answer to "is this real?" is a runnable repo, not another opinion.

I'll be putting an agent through this exact path in the next lab and publishing the meter readings, good or bad. Build. Document. Share. Repeat.
