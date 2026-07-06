---
title: "You're Stuck at Level 6 of the AI Maturity Pyramid. Here's the AWS Map Out."
published: false
description: "Copilot, ChatGPT, Claude in every workflow feels advanced. It's the most common plateau in AI adoption. Here's where AWS services actually move you up."
tags: aws, ai, architecture, strategy
cover_image: ""
canonical_url: ""
---

Your team uses AI every day. Devs autocomplete with Copilot. Support drafts replies with an internal chat assistant. The board deck shows adoption curving up and to the right. By every dashboard you own, you are doing AI well.

You might also be stuck, and not know it.

That's the uncomfortable read I got from Jonny Tooze's AI maturity pyramid, a 12-level model he posted in April 2026. Tooze runs ENDGAME and has a big following in the organizational-AI conversation, so the framework spread fast. The diagram is clean. But the diagram isn't what stuck with me. It was his claim that most enterprises plateau early, in the band where AI writes content, writes code, and answers questions, while a human stays in the loop for every decision. <!-- VERIFY: Tooze's exact wording is "most enterprises plateau at Level 2 AI (basic productivity)" per the LinkedIn source; the seed maps this to the generative "Level 4-6" band. Confirm the level-numbering before publish. -->

The pyramid is a good framework. What it doesn't give you is the part I care about as someone who builds on AWS: which service actually takes you from one floor to the next, and whether the climb is worth it. So I went looking for that map. Here's what I found, including where it gets expensive and where I'd want to run a test before believing my own advice.

---

## The four blocks of the pyramid

Tooze splits 12 levels into four capability blocks. The seed I worked from captured the blocks cleanly, though not all 12 individual level names, so I'll work at the block level and flag that gap. <!-- VERIFY: original Tooze post lists 12 named levels; only the 4 blocks were captured. Pull the 12 names from the source before publishing if you want per-level granularity. -->

[GRAPHIC: pyramid diagram | 4-block AI maturity pyramid — Predictive AI (1-3) at the base, Generative AI (4-6), AI Agents (7-9), Autonomous Orchestration (10-12) at the apex, with the AWS service families labeled on each block | Most teams live in the second block and mistake it for the summit.]

The base is **Predictive AI**, levels 1 through 3. Demand forecasting, ticket routing, anomaly detection. This is the mature, boring, valuable stuff. It's deterministic, it runs on your own data, and the ROI shows up on day one. On AWS this block lives in Amazon Forecast, CloudWatch anomaly detection, and SageMaker Canvas for the no-code cases. Plenty of companies sit here or above it.

Above that sits **Generative AI**, levels 4 through 6. Content, code, chat, internal assistants. This is the plateau. On AWS it's Amazon Bedrock serving Claude, Nova, and Titan, plus Amazon Q Developer for code and Amazon Q Business for chat over your internal data. Bedrock Knowledge Bases handle the domain-specific retrieval. Everything in this block works. That's the trap. It works well enough to feel like the destination.

The third block is **AI Agents**, levels 7 through 9. Here the software plans, calls tools, and acts, instead of just producing text for a human to act on. AWS puts Bedrock Agents, the AgentCore runtime, and the AWS MCP Server in this block. This is where competitive advantage actually starts, according to Tooze, and it's where the architecture changes underneath you.

The apex is **Autonomous Orchestration**, levels 10 through 12. Systems that trigger themselves, grade their own output, and improve across runs. EventBridge fires the loops, AgentCore Evaluations scores them, CloudWatch and X-Ray watch the whole thing. This block is the frontier. Few production examples exist today, and I'd treat any vendor claiming otherwise with a raised eyebrow.

---

## Why level 6 feels like the top

Here's the part that makes the plateau sticky. Being at level 6 feels like being advanced, because by most visible signals you are.

In the generative block, AI makes your people faster. A dev ships a function in less time. A marketer drafts ten variants instead of two. A support agent closes tickets quicker. Every one of those is a real gain, and every one of them keeps the human as the loop. The AI produces, the human reviews, the human decides, the human connects the output to the next step. Pull the human out and the whole thing stops.

That dependency is invisible on a dashboard. Adoption goes up. Everyone is using the tools. Nobody notices that the shape of the work didn't change, only its speed.

Tooze names the shift that breaks the plateau in one line: *stop optimizing speed, start replacing manual work entirely.* <!-- VERIFY: exact quote attribution to Tooze's April 2026 LinkedIn post. --> That shift doesn't come from a bigger model or a better prompt. It comes from a design decision you make at the start. At level 6 you use AI to go faster on a task. At level 7 you remove the task. Here's the detail that surprised me: the AWS service can be identical on both sides. Bedrock powers a chat assistant and an agent alike. What changes is the contract with the human. In the generative block the human closes the loop. In the agent block the agent closes it, and the human sets the goal instead of the steps.

If your AI initiatives all take the form "the human does X faster," you're at 6. If any of them take the form "X happens without a human touching it," you've started the climb.

---

## The gatekeepers between floors

The reason teams stall isn't ambition. It's that each transition has a hard prerequisite, and skipping it means the next floor collapses under you. The seed laid these out as gatekeepers, and they match what I'd expect from anyone who has run an agent in anger.

| Transition | What blocks you | What unlocks it on AWS |
|---|---|---|
| 6 → 7 | No tool use defined. The agent can answer but can't act. | Bedrock Agents with Lambda tools and least-privilege IAM |
| 7 → 9 | No observability. You can't see what the agent did or why. | X-Ray traces plus CloudWatch dashboards on tool calls |
| 9 → 10 | No autonomous loop. A human still triggers every run. | EventBridge scheduled rules, Step Functions for human-in-the-loop exceptions |
| 10 → 12 | No evaluation. The system runs but never learns. | AgentCore Evaluations with its evaluator suite and ground truth |

Read that table top to bottom and a pattern shows up. Every gate is a piece of engineering discipline, not a model upgrade. Tool definitions. Traces. Schedules. Evaluations. None of it is glamorous. All of it is the difference between a demo and a system you'd let run overnight.

The 6-to-7 gate is the one most teams hit first, so it's worth sitting on. To move from a chat assistant to an agent, the model needs tools it can call, and each tool needs a permission boundary. In AWS terms that's Bedrock Agents wired to Lambda functions, each function scoped by an IAM role that grants the minimum it needs. Get the IAM wrong here and you've built an autonomous thing with broad permissions. That's exactly the failure mode you don't want to discover in production. This is the floor where "let me test the blast radius first" turns from paranoia into your actual job.

---

## What the 7-to-9 climb actually costs you

The plateau story is clean. Reality rarely is, and the seed left a few parts open that deserve saying out loud.

The jump from a working agent (7) to a trustworthy one (9) is mostly observability, and observability isn't free. Every tool call your agent makes is a Lambda invocation, a Bedrock inference charge, an X-Ray trace, and CloudWatch log volume. An agent that loops through five tools to answer one request costs more than the single Bedrock call it replaced at level 6. The value can be higher too, since it's doing work a human used to do. But anyone selling you the climb as a pure win is skipping the bill.

I couldn't pin down a firm number for the minimum cost of one production agent on Bedrock, and I'm not going to invent one. <!-- VERIFY: seed open question — what is the minimum monthly cost of a single production Bedrock agent? No sourced figure available; needs a real pricing estimate or a small benchmark before any cost claim is published. --> If I were sizing this before committing, the first thing I'd test is the cost of one representative agent run end to end: count the tool calls, price the Bedrock tokens per call, add the Lambda and trace overhead, then multiply by expected volume. That single measurement tells you more than any maturity model. It's also the kind of test worth publishing, because AWS docs are thin on what a real agent costs to operate.

The top of the pyramid gets the same treatment. AgentCore Evaluations reached general availability in March 2026 with a suite of evaluators and ground-truth support, which is what a level-10-to-12 system needs to grade and improve itself. <!-- VERIFY: AgentCore Evaluations GA date (March 2026) and the "13 evaluators + Ground Truth" claim, per AWS What's New. --> The AWS MCP Server hit GA in May 2026, which is what standardizes how agents pull in external tools. <!-- VERIFY: AWS MCP Server GA date, May 6 2026, per AWS News Blog (Sébastien Stormacq). --> The tooling to build autonomous systems now exists. Whether a given team should build one is a separate question, and for most the answer at levels 10 through 12 is "not yet." The seed's own open question stands: I couldn't find a named AWS customer running publicly at level 9 through 12, so treat that top block as a direction, not a case study. <!-- VERIFY: open question — is there a public AWS case study of a customer operating at pyramid level 9-12? Search aws.amazon.com/solutions/case-studies before citing any example. -->

---

## The three jumps that matter

Twelve levels is a lot to hold in your head. If I compress the pyramid to the moments where the architecture genuinely changes, there are three.

The first is adding AI to the stack at all, roughly level 1 into 4. Predictive models and generative assistants join a system that was pure software before. Most readers of this are already past it.

The second is the 6-to-7 jump, from tools to agents. This is the real one. It's where the human stops being the loop, where IAM and tool design suddenly carry production weight, and where "faster" turns into "replaced." If your team makes one deliberate move this year, this is the move.

The third is 9-to-12, from agents to self-running systems. EventBridge triggers, AgentCore evaluations, the full observability stack. It's frontier work, high-value and high-risk, and it's the one place I'd tell most teams to wait and watch rather than lead.

You don't climb this pyramid because the diagram says so. You climb the one floor where your current dependency on humans is costing you the most, and you climb it only after the gatekeeper for that floor is actually met.

---

## Where I'd start, and where I'd wait

Straight up: this is analysis, not a war story. I haven't run a fleet of production agents through all twelve levels, and I won't pretend I have. What I have is a map, a set of gatekeepers that hold up to scrutiny, and a clear read on where the map goes quiet.

If I were placing my own team on this pyramid today, I'd do three things. I'd figure out which level we're actually at by one test, not by the dashboard: is there any workflow where AI acts without a human closing the loop? If not, we're at 6, full stop. Then I'd pick the single workflow where being the loop hurts most and build one Bedrock agent for it, tools scoped tight, IAM tighter, X-Ray on from the first run. And I'd measure what that one agent costs before scaling it, because that number is the thing nobody publishes and everybody needs.

The questions I can't answer yet are the useful ones. What does a minimum production agent actually cost per month on Bedrock? Is there a real AWS customer operating at level 9 or above that we can learn from? Does Tooze's maturity model line up with the AWS Well-Architected guidance for AI workloads, or do they disagree in ways worth knowing? I'd rather leave those open and honest than fill them with a confident guess.

If you want to check your own team against this map, here's the tiered invite. Low effort: run the one-question test above and tell me in the comments what level it puts you at. I'm curious how many land on 6. Medium effort: if you've built a Bedrock agent in production, share what it cost you to run, because that's the data gap this whole space has. High effort: if you can point at a public AWS case study operating at the autonomous levels, drop the link. I'll update this piece and credit you.

Build. Document. Share. Repeat. The map is only as good as the tests we run against it, so let's run some.
