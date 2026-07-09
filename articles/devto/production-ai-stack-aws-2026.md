---
title: "The 2026 Production AI Stack Stopped Growing Taller. Here's the AWS Map."
published: false
description: "Observability and governance stopped being layers and became vertical rails. Here's every node of the 2026 AI stack mapped to the AWS service that covers it."
tags: aws, ai, architecture, cloud
cover_image: ""
canonical_url: ""
---

For two years the production AI stack grew by getting taller. Every six months, another layer landed on top. Vector databases. Orchestration frameworks. Then agents. Then evaluation. You could watch the tower rise in every architecture diagram that made the rounds.

Then it stopped.

The May 2026 edition of the stack map didn't add a new floor. It changed shape. Observability and governance, which used to sit as boxes somewhere near the bottom, turned sideways. They became vertical rails that run through every single layer, from the model call at the top to the audit log at the bottom. That's the shift I want to walk through, because it changes how you build on AWS, not just how you draw the picture.

The map I'm working from comes from @codewithbrij, who redraws the production AI stack every six months and posts it. The May 2026 edition (649 likes, for what a like is worth) marked a clean before-and-after. I'm not here to take credit for the map. I'm here to do the part the map skips: replace every node with the AWS service that actually covers that position, and call out the two places where I'd want to run a test before trusting my own mapping.

One disclaimer up front, because it matters. This is my interpretive mapping, not AWS documentation. The stack shapes belong to @codewithbrij. The AWS-service picks are mine, and I'll flag the ones I'm less sure about. The value here is the curation and the judgment calls, not any claim to authority.

---

## What changed between the last redraw and this one

Five things moved in the May 2026 edition, and each one tells you where the industry stopped arguing.

MCP replaced custom integrations. What used to be a pile of proprietary adapters, one per tool, is now a single protocol. The Model Context Protocol went from "interesting" to the default way agents talk to tools. AWS shipped its own MCP servers to GA in May 2026, so this isn't a fringe bet anymore.

Vector databases got absorbed into Postgres. The specialized vector-DB layer, the one everyone stood up a separate service for, mostly disappeared into `pgvector`. On AWS that means Aurora PostgreSQL now covers the case that used to justify a whole extra box on the diagram.

Frameworks consolidated, and some died. AutoGen is gone, discontinued. LangChain and LlamaIndex are both under real pressure. I'll come back to what that means for what you should adopt, because "the framework is under pressure" is not the same as "rip it out tomorrow."

AI gateways had their first real security moment. In April 2026, an AI gateway vulnerability landed on the CISA Known Exploited Vulnerabilities catalog. That's the line where a category stops being a convenience and becomes a security requirement. More on this below, including one thing I couldn't fully pin down.

And the big one. Observability and governance stopped being layers. They became rails. @codewithbrij put it as a rule: not on day 100, on day 1.

---

## The map: every layer, and the AWS service that covers it

Start with the core of it: the classic horizontal layers of the stack, each with the AWS service I'd reach for.

[GRAPHIC: architecture diagram | The 2026 production AI stack as horizontal layers (Model, Orchestration, MCP, Memory, RAG, Gateway, Prompt Mgmt, Eval, Guardrails) with two vertical rails — Observability and Governance — running through all of them, each node labeled with its AWS service | The stack stopped growing taller; the rails now run top to bottom.]

| Layer / Component | AWS service | What it actually does here |
|---|---|---|
| **Model / Inference** | Amazon Bedrock | ~100 models behind one endpoint: Claude, Nova, plus OpenAI, Google, Mistral, Qwen and more |
| **Orchestration** | Step Functions / Bedrock Agents | Step Functions for deterministic flows; Bedrock Agents when the flow is genuinely agentic |
| **MCP / Integrations** | AWS MCP Servers (GA May 2026) | The standard replacement for custom per-tool adapters |
| **Memory / Vector store** | Aurora (`pgvector`) / OpenSearch Serverless | Aurora `pgvector` absorbs the specialized vector-DB use case |
| **Knowledge / RAG** | Bedrock Knowledge Bases | Managed RAG with an agentic retriever; Kendra as the alternative |
| **AI Gateway** | API Gateway + AWS WAF | Rate limiting, auth, throttling; WAF for the prompt-injection surface |
| **Prompt management** | Bedrock Prompt Management | Versioning and A/B of prompts in production |
| **Evaluation** | AgentCore Evaluations (GA Mar 2026) | 13 evaluators with Ground Truth built in |
| **Policy / Guardrails** | AgentCore Policy + Bedrock Guardrails | Fine-grained agent-to-tool control; content filters |

A few of these deserve a sentence more than a table cell.

Bedrock at the model layer is the pick that ages best. The whole point of the 2026 stack is that you no longer bet on one model provider. With around 100 models behind a single endpoint, including OpenAI and Google models, the inference layer becomes a routing decision, not a vendor lock-in. That's the receipt behind "one endpoint": you swap `anthropic.claude` for `amazon.nova` in a parameter, not in your architecture.

The memory layer is where the map earns its "stopped growing taller" thesis most clearly. Two years ago you'd stand up a dedicated vector database. Now the honest answer for most teams is Aurora with `pgvector`. The specialized layer didn't get better. It got absorbed. OpenSearch Serverless is still the right call when your vector workload is large or latency-sensitive enough to justify a purpose-built engine, but that's now the exception, not the default.

Orchestration is the one place I'd push back on treating the map as gospel. "Orchestration" is one node on the diagram and two very different jobs in practice. If your flow is deterministic, a fixed sequence of steps with branching you can draw, that's Step Functions, and reaching for an agent there just adds cost and nondeterminism. If the flow genuinely needs the model to decide what happens next, that's Bedrock Agents. Same box on the map. Opposite tools.

---

## The rails: the part nobody adds until it's too late

This is the actual thesis, so it gets prose, not a table.

The reason observability and governance turned from boxes into rails is that an AI system fails differently than a normal service. When a REST endpoint breaks, the stack trace usually points at the problem. When an agent gives you a wrong answer, nothing crashed. The model returned tokens. The tool ran. The output was confident and wrong. There's no exception to catch. The only way you find out what happened is if you were already recording the whole path: which prompt, which model, which tool call, which retrieved document, which guardrail decision.

That's why the rule is day 1, not day 100. If you add tracing after something goes wrong in production, you're debugging a failure you have no record of. You get to reproduce it live, in front of whoever noticed.

On AWS, the observability rail has three pieces. X-Ray gives you the distributed trace end to end, from the trigger through the model call through the tool and back to the output. CloudWatch carries the metrics that tell you it's degrading before it breaks: latency, token usage, error rate per component. And AWS Distro for OpenTelemetry is the piece I'd argue for hardest, because it keeps your telemetry portable. If you instrument with OTel instead of straight to CloudWatch, you're not rebuilding your observability the day you add a non-AWS component. The standard outlives the vendor choice.

The governance rail is the one that turns compliance from a scramble into a query. CloudTrail logs every model call, which is what makes an audit answerable at all when someone asks "what did the system do on this date." AWS Config catches configuration drift, so the guardrail that was on last month doesn't quietly turn off. IAM conditions do the fine-grained "what is allowed to call what," which in an agent world is most of your security posture. And Security Hub pulls the whole picture into one place instead of four dashboards.

None of this is exciting. That's the point. Governance is boring until an auditor or an incident makes it the only thing in the room, and by then you either have the logs or you don't.

---

## The security moment, and the one thing I couldn't verify

The April 2026 CISA KEV listing is worth sitting with, because it's the moment the AI gateway stopped being optional.

KEV means "Known Exploited Vulnerabilities." It's not a theoretical CVE list. It's the list of things that are being exploited in the wild, right now, which is why US federal agencies are required to patch KEV entries on a deadline. When a category shows up there, the conversation changes. The AI gateway went from a nice throttling convenience to a security control you're expected to have.

On AWS, the gateway rail is API Gateway for the plumbing, auth, rate limiting, throttling, with AWS WAF in front for the LLM-specific attack surface, including prompt-injection patterns. At the enterprise end, Bedrock Guardrails plus AgentCore Policy give you the content filtering and the agent-to-tool control that a raw gateway doesn't.

Now the part I can't clean up. I could not pin down exactly which AI gateway product landed on the KEV catalog in April 2026, or whether the listing was a specific product or a broader category entry. That distinction changes how you'd respond, so I'm marking it as an open question instead of papering over it.

<!-- VERIFY: Identify the specific AI gateway CVE/product listed on the CISA KEV catalog in April 2026. Confirm whether it was a single product or a category. Source: @codewithbrij May 2026 post referenced this but did not name the CVE. Cross-check against https://www.cisa.gov/known-exploited-vulnerabilities-catalog before publishing. -->

---

## What not to adopt (the framework consolidation)

The other useful thing a stack map does is tell you where not to spend your time. May 2026 was rough on the framework layer.

| Framework | Status, May 2026 | Where I'd go on AWS instead |
|---|---|---|
| AutoGen | Discontinued | Bedrock Agents multi-agent |
| LangChain | Under pressure | Step Functions + native Bedrock |
| LlamaIndex | Under pressure | Bedrock Knowledge Bases |
| Custom vector DB | Absorbed by Postgres | Aurora `pgvector` |

"Under pressure" does not mean "delete it Monday." If you have LangChain running in production and it works, read this table as guidance for the greenfield decision, not a migration order. When you're starting something new in mid-2026, the native Bedrock path has caught up enough that reaching for a heavy framework is a choice you should have to justify, not a default. AutoGen is the only clear-cut one. It's gone, so building new on it is building on a dead dependency.

---

## What I'd double-check before trusting this map

The whole value here is curation you can trust, so this is where I'd want a test before betting a production decision on my own mapping.

I'd confirm the AgentCore Evaluations and AgentCore Policy GA claims against the AWS console, not just the launch posts, because "GA" and "GA in your region with the quota you need" are different facts, and I've been burned by that gap before. I'd verify the CISA KEV specifics, as flagged above. And I'd treat the framework-consolidation calls as a snapshot with a six-month shelf life. This map gets redrawn in November. Some of what looks settled in May won't be.

<!-- VERIFY: Confirm AgentCore Evaluations (GA Mar 2026, 13 evaluators + Ground Truth) and AgentCore Policy region availability in the AWS console before presenting as production-ready. Launch-post GA ≠ regional availability. -->

That's the map. Nine horizontal layers, each with an AWS service that covers it, and two vertical rails, observability and governance, that the best teams wire in on day 1 and everyone else adds on day 100 after something breaks. The stack stopped growing taller because the hard problem moved. It's no longer "what layer do we add." It's "does the whole thing tell us the truth when it fails."

If you're building any part of this on AWS, three ways to take it further, pick your commitment level. Low: save the table, and next time someone shows you an AI stack diagram, ask where the observability rail is. If it's a box and not a rail, you found the gap. Medium: pick one rail and wire it in before your next feature. OpenTelemetry via ADOT is the single highest-return move. High: take one layer where you disagree with my AWS pick, build both, and tell me which one won and why. I'll update the map, with credit.

I map these publicly and get them wrong sometimes, which is the whole point of doing it in the open. If you'd map a node differently, the comments are the best part.

Build. Document. Share. Repeat.
