---
title: "The Software Factory Blueprint Is Real. The AWS Translation Is Missing."
published: false
description: "freeCodeCamp's 5-layer, 7-agent Software Factory is production-grade architecture. Nobody wrote the AWS version with service names, ARNs, and the 3 decisions that break it."
tags: aws, ai, aiagents, architecture
cover_image: ""
canonical_url: ""
---

There's a diagram going around. A London Tech Lead published it through freeCodeCamp under the name "Software Factory Architecture with AI Agents," and it's good. Five layers, stacked clean: Context, Knowledge, Agent, Workflow, Delivery. Seven specialized agents. Three points where a human has to approve before the machine keeps going.

I read it twice. The abstraction holds up. This is not tutorial fluff, it's how a real multi-agent system should be organized. But then I hit the wall every architect hits with a good diagram: it tells you *what* to build and says nothing about *what to build it with*.

Which Knowledge Base implements the Knowledge layer? What routes tasks between seven agents? How do you pause a workflow indefinitely while a human clicks approve, without paying for a Lambda that spins in a loop? The diagram is silent. That silence is the whole reason this article exists.

One thing up front. I'm not critiquing the freeCodeCamp model. I'm extending it. The blueprint is sound. What's missing is the AWS layer underneath it, with real service names and the three design decisions I've watched teams get wrong. Let's build the translation.

<!-- VERIFY: freeCodeCamp article title "Software Factory Architecture with AI Agents" and author framing "London Tech Lead, 2026". Confirm exact title and URL before publish; seed lists it as captured reference. -->

---

## The five layers, mapped to real services

Start with the layer map, because everything else hangs off it. Each abstract layer becomes something concrete once you actually deploy it on AWS.

[GRAPHIC: architecture diagram | The 5 layers (Context, Knowledge, Agent, Workflow, Delivery) each labeled with its AWS service(s), arrows showing data flow top to bottom | "Every layer has a home on AWS. This is which service lives where."]

The **Context layer** holds project information, history, and current state. That splits into two AWS homes, not one. Static, durable context (specs, prior decisions, the stuff that doesn't change mid-run) lives in S3. Session state, the working memory each agent reads and writes during a task, belongs in DynamoDB. Keep those separate. I'll come back to why, because collapsing them is one of the three mistakes.

The **Knowledge layer** is domain knowledge: documentation, patterns, the things your agents need to reason well. This is Bedrock Knowledge Bases, with Smart Parsing to chunk your documents sensibly and the Agentic Retriever to fetch across them. One fork in the road here. If your knowledge is document-shaped and internal, a Knowledge Base backed by S3 is the straight path. If it's tabular, with structured records you need to query semantically, you're better off with Aurora PostgreSQL and the `pgvector` extension. Pick based on the shape of your data, not the marketing page.

The **Agent layer** is the seven specialists. Each one becomes a Bedrock Agent running on Claude, and each carries its own tools as Lambda functions exposed through Tool Use. For production runtime you wrap them in the Bedrock AgentCore harness, which handles the session, memory, and identity plumbing you'd otherwise write yourself. One agent, one Bedrock Agent, one set of Lambda tools. Clean mapping.

The **Workflow layer** is the orchestrator, the thing that decides which agent runs next. This is the layer people get most wrong, so it gets its own section below. Short version: it's a hybrid of Bedrock Agents multi-agent collaboration for the reasoning-heavy routing and Step Functions for the parts that must be deterministic.

The **Delivery layer** pushes output to the user or the next system. API Gateway plus Lambda for synchronous responses, EventBridge for anything asynchronous. If a downstream system reacts to "architecture approved" or "tests passed" as an event, that's EventBridge. If a caller is waiting on a response, that's API Gateway.

The same map reads better as a table when you're implementing, so here it is as a decision matrix.

| Layer | Function | AWS service | The design call |
|---|---|---|---|
| Context | Project info, history, current state | S3 (static context) + DynamoDB (session state) | Split them. S3 for durable, DynamoDB for per-agent working memory |
| Knowledge | Domain knowledge, docs, patterns | Bedrock Knowledge Bases (Smart Parsing + Agentic Retriever) | Documents → KB on S3. Tabular data → Aurora `pgvector` |
| Agent | 7 specialists with their own tools | Bedrock Agents (Tool Use) + AgentCore runtime | One agent = one Bedrock Agent + its own Lambda tools |
| Workflow | Orchestrator routing between agents | Bedrock multi-agent + Step Functions | Bedrock for agentic routing, Step Functions for deterministic gates |
| Delivery | Output to user or external system | API Gateway + Lambda + EventBridge | EventBridge async, API Gateway sync |

---

## The seven agents, as Bedrock Agents

The blueprint names seven specialists. Each maps to a Bedrock Agent on Claude, distinguished by the tools you hand it. The tools are the interesting part, because that's where the agent touches real AWS services and real external systems through MCP.

| Specialist | AWS form | Tools it holds |
|---|---|---|
| Requirements Analyst | Bedrock Agent (Claude) | Bedrock KB with specs, JIRA via MCP |
| Architect | Bedrock Agent (Claude) | AWS Diagrams via MCP, Well-Architected Tool API |
| Backend Developer | Bedrock Agent (Claude) | CodeBuild, Lambda, CodeCommit via MCP |
| Frontend Developer | Bedrock Agent (Claude) | CloudFront, Amplify via MCP |
| QA Tester | Bedrock Agent (Claude) | CodeBuild CI, test runners on Lambda |
| Security Reviewer | Bedrock Agent (Claude) | Inspector, GuardDuty, Security Hub via MCP |
| Orchestrator | Bedrock Agents multi-agent | The other six as sub-agents |

<!-- VERIFY: The seven agent names and their exact roster come from the freeCodeCamp article. Seed flags this as tentative ("verificar los 7 nombres exactos en el artículo original"). Confirm the real seven before publish; the AWS tool mappings are my own and are safe. -->

The MCP threading matters more than it looks. The AWS MCP Server reached GA, which means these agents don't hit JIRA or CodeCommit through brittle custom glue. They call standardized MCP tools with IAM governing every call. When your Backend Developer agent commits to CodeCommit, that action is an IAM-scoped, CloudTrail-logged operation, not a rogue script with a stored token. That's the difference between a demo and something you'd let near production.

<!-- VERIFY: AWS MCP Server general availability. Seed cites the AWS blog "Introducing AWS MCP servers for code assistants." Confirm GA status wording before publish. -->

---

## Decision one: don't route everything through one brain

This is the mistake I see most, and it's seductive because it feels elegant. Bedrock supports multi-agent collaboration, so the temptation is to make the Orchestrator a single Bedrock super-agent that reasons about every hop. Requirements done? Let the model decide to call the Architect. Architecture done? Let the model decide to call the Backend Developer. All routing, all LLM.

It works in the demo. Then it drifts.

The problem is that some transitions in a Software Factory are not judgment calls. "After the Architect finishes, a human must approve before code starts" is not something you want a language model deciding to skip because the context window got crowded. That's a rule. Rules belong in deterministic control flow, which is Step Functions, not in a probabilistic router.

The correct shape is a hybrid. Use Bedrock multi-agent collaboration where the routing genuinely needs reasoning, like deciding whether a task needs the Backend or Frontend specialist. Use Step Functions for the deterministic spine: the fixed sequence, the retries, the approval gates. Bedrock does the thinking. Step Functions enforces the law.

The catch, and I won't pretend otherwise, is that the hybrid costs you design time upfront. You have to decide which transitions are reasoning and which are rules before you write a line. The all-Bedrock version is faster to stand up and slower to trust. Pay the design cost early or pay the debugging cost forever.

---

## Decision two: give the agents a shared store, not a growing prompt

The second mistake hides until your third agent, and then it explodes.

The tempting pattern goes like this. Agent one produces output. You take that output, stuff it into the prompt for agent two as a string. Agent two produces more. Now the prompt for agent three carries agent one's output *and* agent two's output, both as text. By the fourth handoff you're passing a wall of accumulated context that's expensive to send, easy to truncate, and impossible to query.

State passed as strings in prompts doesn't scale past a couple of agents. The fix is the DynamoDB session store from the Context layer. Every agent reads the current state and writes its contribution back, keyed by session. Agent three doesn't inherit a swelling prompt. It reads exactly the fields it needs from a shared table.

This is why the Context layer split into S3 and DynamoDB back at the start. S3 holds the durable stuff. DynamoDB holds the live session state that all seven agents share. Collapse those two and you either bloat your prompts or lose your durability. Keep them apart and each agent stays lean, reading a store instead of parsing a paragraph.

---

## Decision three: pause with a task token, not a polling loop

The three human approval gates are the heart of a Software Factory. They're the reason a leadership team will actually let this thing near production. Miss the implementation and you've built something clever that nobody trusts.

The three gates sit at natural risk boundaries:

- **Gate 1**, after the Architect: a human approves the design before any code gets written.
- **Gate 2**, after QA: a human approves test results before deploy.
- **Gate 3**, after the Security Reviewer: a human signs off before production.

The wrong way to build a gate is to poll. You write a Lambda that wakes up, checks "did the human approve yet," goes back to sleep, wakes up again. It works. It also burns invocations, adds latency you can't predict, and turns a simple pause into a scheduling problem you now own.

Step Functions gives you the pattern for free. It's the `WaitForTaskToken` integration, and it does exactly what a human gate needs. The flow looks like this:

```
State Machine (per gate):
  1. Task          → the agent produces its output
  2. SendTaskToken → notify the human (SES email, Slack, or SNS)
                     the workflow now holds a task token and PAUSES
  3. .waitForTaskToken → Step Functions waits. Indefinitely. Zero polling.
  4. Human decides → approves or rejects via a pre-signed URL or a UI
  5. SendTaskSuccess → flow continues
     SendTaskFailure → flow aborts cleanly
```

The word that matters is *indefinitely*. Step Functions holds that paused state for you at no per-second compute cost, waiting for the token to come back. A human can approve in four minutes or four days, and nothing is spinning or polling in the meantime. When `SendTaskSuccess` fires with the token, the machine picks up exactly where it left off. That's the entire difference between a gate that scales and a gate that quietly runs up a bill.

Wire all three gates the same way, each as a `.waitForTaskToken` step in the Step Functions spine from decision one. Now the deterministic backbone and the human control points are the same mechanism. That's not a coincidence. That's the design working.

---

## What the blueprint gave us, and what it didn't

The freeCodeCamp model earned its reputation. Five layers, seven agents, three gates: that's a genuinely useful way to think about a multi-agent build, and I'd point anyone starting from scratch at that diagram first.

What it doesn't give you is the part where the abstraction meets a bill and an IAM policy. The Knowledge layer is a Bedrock Knowledge Base or an Aurora `pgvector` table, and the choice depends on your data shape. The Workflow layer is a hybrid of Bedrock reasoning and Step Functions determinism, and getting the split wrong costs you trust. The gates are a task-token pause, not a polling loop, and that one choice is the line between a workflow that scales and one that leaks money.

I still have open questions I couldn't close from the diagram alone, and I'll be honest about them rather than paper over them. At what exact point does the orchestrator decide which agent to activate, LLM routing or hardcoded rules? What does this full stack actually cost per month for a small team? Has anyone published running this exact pattern in production on AWS? If you've built any piece of this, those are the answers I want.

<!-- VERIFY: Open questions above are unresolved in the seed (section 7). Keeping them as honest open questions rather than inventing figures. No cost estimate fabricated per authenticity rule. -->

The blueprint is thinking made visible. The AWS translation is that thinking made deployable. If you're holding the diagram and staring at the AWS console wondering how the two connect, start with the layer table, get the three decisions right, and the rest follows.

If this was useful, the lowest-effort thing you can do is drop the one AWS service you'd swap in my mapping. Somewhere I picked S3 and you'd pick EFS, or I picked Step Functions and you run this on EventBridge Pipes. I want to know. If you want to go deeper, take one layer, the Knowledge layer is a good start, and build just that piece with your own data, then tell me where my mapping broke. And if you've already got Bedrock Agents in a demo and you're trying to grow it into this structured shape, that's exactly the migration I'm working through too. Let's compare notes in the comments.

Build. Document. Share. Repeat.
