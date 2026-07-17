---
title: "Loop Engineering on AWS: Stop Writing Prompts, Design the System That Writes Them"
published: false
description: "Addy Osmani named the shift from prompter to loop architect. Here's each of his six components mapped to a real AWS service."
tags: aws, aiagents, amazonbedrock, architecture
cover_image: "https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/loop-engineering-on-aws/00-cover.png"
canonical_url: ""
---

Stop being the person who writes prompts. Become the person who designs the system that writes them for you.

That's the whole idea behind a phrase Addy Osmani (engineering lead on Chrome DevTools at Google) dropped on LinkedIn on June 9, 2026. He called it **loop engineering**, and the post landed with 1,277 likes and 124 comments. His definition is short enough to memorize:

> "Loop engineering: replacing yourself as the person who prompts the agent. You design the system that does it instead. A loop is a recursive objective where AI iterates until complete."

Osmani didn't invent the practice. The best teams were already doing it. He named it, and naming is what turns a scattered practice into shared vocabulary. Once someone with his credibility gives it a word, the industry starts using that word to argue about architecture.

I build agent loops every day in my own tooling. Not a demo, not a weekend toy. A running system of scheduled tasks, skills, sub-agents, and memory that runs discovery and drafting on a cron tick, then hands me decisions instead of asking me to type. So when Osmani listed the six pieces of a loop, I recognized every one of them. What his post doesn't give you is the part I care about most: where does each piece actually *live* when you run it on AWS?

That's the gap this article fills. Six components, six concrete services, one minimal loop you can trace end to end. And a warning at the end that I think matters more than any of the wiring.

---

## The six components, and why the timing is not an accident

Osmani lists five components: automations, worktrees, skills, plugins (MCP), and sub-agents, plus memory as what he calls the sixth thing, state that lives outside the single conversation. Read that list as an architecture. Each part answers a different question. What triggers the work. Where the work runs in isolation. What the agent knows. How it reaches tools. Who does the specialized labor. And what it remembers between runs.

The reason this is worth writing *now* is that the AWS side matured on almost the same calendar as Osmani's post. AWS MCP Server hit GA in May 2026. AgentCore Policy went GA in March, AgentCore Evaluations the same month, and the AgentCore harness reached GA in June 2026, per the [AgentCore release notes](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/release-notes.html). Two threads, the mental model and the infrastructure, arrived at the same knot in the same month.

Here is the map I keep coming back to.

![Six loop components mapped to AWS services: Automations to EventBridge plus Lambda, Worktrees to AgentCore Runtime microVMs, Skills to Bedrock Knowledge Bases, Plugins (MCP) to AWS MCP Server, Sub-agents to AgentCore harness, Memory files to S3 plus DynamoDB](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/loop-engineering-on-aws/01-osmani-aws-map.png?v=2)

*Osmani's six loop components on the left, the AWS service each one lives on when you run it in production on the right.*

| Component (Osmani) | What it does | AWS service | Key config |
|---|---|---|---|
| **Automations** | Scheduled discovery. The loop fires itself, no human prompt. | EventBridge (scheduled rules + event bus) → Lambda | Cron expression or event pattern; Lambda as dispatcher |
| **Worktrees** | Isolated branch per agent so parallel runs don't collide | AgentCore Runtime (microVM per session) | Each session gets its own `/mnt/workspace`; git stays the coordination layer |
| **Skills** | Reusable project knowledge the agent loads into context | Bedrock Knowledge Bases (Smart Parsing) | Pre-computed chunking + embeddings; structured S3 too |
| **Plugins / MCP** | Standard connection to external tools | AWS MCP Server (GA May 2026) | The new default; replaced custom integrations |
| **Sub-agents** | Specialists orchestrated by a coordinator | AgentCore harness (GA June 2026) | Orchestrator harness routing to specialist agents |
| **Memory files** | State that survives between sessions | S3 (long-term) + DynamoDB (short-term) | S3 for context files; DynamoDB for fast state |

Sources: [Addy Osmani, LinkedIn (June 9, 2026)](https://www.linkedin.com/posts/addyosmani_ai-softwareengineering-programming-activity-7469999258658791425-cTvy); [AWS MCP Servers announcement](https://aws.amazon.com/blogs/machine-learning/introducing-aws-mcp-servers-for-code-assistants/); [Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html).

Let me walk each row, because a table hides the interesting decisions.

---

## Automations: the loop has to fire itself

This is the line that separates a loop from a chatbot. If a human still has to type "go," it's not a loop. It's a prompt with extra steps.

On AWS, the trigger is EventBridge. A scheduled rule handles the "run discovery every morning" case with a cron expression. An event bus handles the "react when something lands in this bucket" case with an event pattern. Either way, EventBridge doesn't run your logic. It wakes up a Lambda function, and that Lambda is your dispatcher, the thing that decides what the loop should do this cycle and calls the orchestrator.

```python
# Lambda dispatcher — invoked by an EventBridge scheduled rule.
# Job: decide what this cycle's objective is, then hand it to the orchestrator.
# IAM: this function's role needs bedrock-agentcore:InvokeAgentRuntime on the runtime ARN.
import json
import boto3

agentcore = boto3.client("bedrock-agentcore")

def handler(event, context):
    # EventBridge gives us the trigger; we set the recursive objective.
    objective = build_objective_from(event)   # cron tick or event payload
    response = agentcore.invoke_agent_runtime(
        agentRuntimeArn="ORCH_RUNTIME_ARN",
        runtimeSessionId=session_for(event),   # ties back to memory, see below
        payload=json.dumps({"prompt": objective}),
    )
    return {"status": "dispatched", "objective": objective}
```

Why put a Lambda between EventBridge and Bedrock at all? Because the dispatcher is where you set the objective and, later, where you'll bolt on a circuit breaker. That thin layer is cheap insurance. I'll come back to why I never skip it.

---

## Worktrees: isolation so parallel agents don't step on each other

Osmani's "worktrees" borrows from git: each agent gets its own branch so two agents editing the same tree don't corrupt each other's state. When agents run in parallel, shared mutable state is how you get a mess that's impossible to debug.

The AWS translation is isolation per agent session: AgentCore Runtime runs each session in its own microVM. Each run gets a clean, isolated environment and tears down after.

I flagged this row as a seam in an early draft, and running it to ground made the map better. Osmani's worktrees are literally git worktrees inside a running coding agent, and AWS doesn't reproduce them. It replaces them. Each AgentCore session gets its own microVM with its own `/mnt/workspace`, so the isolation that git worktrees fake at the directory level happens at the machine level, with git left in place as the coordination layer. [AWS's own write-up on hosting coding agents](https://aws.amazon.com/blogs/machine-learning/its-safe-to-close-your-laptop-now-hosting-coding-agents-on-amazon-bedrock-agentcore/) is explicit: no worktree management needed, because the filesystem itself is per-session. Treat this row as "isolation, upgraded," not "git worktrees, reproduced."

---

## Skills: what the agent knows before it starts

Skills are reusable project knowledge the agent pulls into context, so it doesn't relearn your conventions every cycle. On AWS that's Bedrock Knowledge Bases with Smart Parsing: you pre-compute chunking and embeddings, and the agent retrieves against them at run time. For flatter, more structured knowledge, a well-organized S3 layout does the job without the retrieval overhead.

The decision I'd surface here is *when* Knowledge Bases earns its cost. Retrieval isn't free, and for a small, stable skill set, structured S3 files the agent reads directly can beat a vector round-trip. I default to S3 for anything the agent needs every single run, and reserve Knowledge Bases for the large, changing corpus where retrieval actually saves tokens.

---

## Plugins and MCP: the part that quietly became the standard

This is the row that changed the most in 2026. The Model Context Protocol went from "interesting" to the default way agents reach tools. The production-stack write-up from @codewithbrij (Instagram, ~May 2026, 649 likes) put it bluntly: MCP replaced custom integrations, and the whole stack stopped growing taller and instead thinned out in the middle and thickened at the edges. Fewer bespoke connectors, more standard ones.

AWS MCP Server reaching GA in May 2026 is what makes this real on AWS, and per Prasad Rao (AWS Principal SA, LinkedIn, May 2026) it's pay-as-you-go with no extra charge for the server itself, which [AWS's GA announcement](https://aws.amazon.com/about-aws/whats-new/2026/05/aws-mcp-server/) confirms: you pay only for the resources your agents use. Practically, MCP is how your orchestrator calls out to external tools without you writing and maintaining glue code for each one.

---

## Sub-agents: specialists with a coordinator

A single agent doing everything is a single agent doing everything badly. Osmani's loop uses sub-agents: specialists, each good at one thing, orchestrated by a coordinator.

On AWS this now lives in one place: the AgentCore harness, GA since June 2026, holds an orchestrator and its specialists together in production. (If you were about to reach for Bedrock Agents for this row, don't. That service is now [Bedrock Agents Classic](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html) and closes to new customers on July 30, 2026; AWS points migrations at AgentCore.) And "in production" comes with numbers you should know before you design: AgentCore's June 2026 quotas are **200 TPS per agent** and **5,000 active sessions per account** in us-east-1 and us-west-2, 2,500 in other Regions ([AgentCore release notes](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/release-notes.html)). Agents hold sessions, sometimes for a long time, so the session ceiling is the one I'd model first. It's easy to blow past 5,000 concurrent sessions with a fleet of long-lived loops and never touch the TPS limit.

This is also where Osmani's *other* framing clicks into place. A month before the LinkedIn post, he wrote on O'Reilly Radar about "agent harness engineering," and the line worth taping to your monitor is: *"A decent model with an excellent harness outperforms an excellent model with a poor harness."* The harness and the loop are the same idea named for two audiences. The AgentCore harness is AWS's answer to exactly that sentence.

---

## Memory files: state that outlives the session

The last component is memory: what the agent carries across runs so cycle two knows what cycle one did. The seed splits this cleanly, and I agree with the split. S3 for long-term context, the files and artifacts the agent should still see next week. DynamoDB for short-term, fast state: the session pointer, the last checkpoint, the "where was I" record that every dispatch reads and writes.

That session ID in the dispatcher code above is the thread. It ties EventBridge's trigger to DynamoDB's state to S3's context, and it's how a set of independent Lambda invocations becomes one continuous loop instead of six strangers.

---

## The minimal viable loop, wired end to end

Put the six together and you get a loop small enough to draw and real enough to run.

```
EventBridge rule (cron / event)
  -> Lambda (dispatcher, sets objective + circuit breaker)
    -> AgentCore harness (orchestrator)
        |- MCP Server (external tools)
        |- Knowledge Base (context / skills)
        |- Sub-agents (specialists)
  -> S3 / DynamoDB (memory files)
  -> CloudWatch (loop observability)
```

The permission chain is short. EventBridge needs invoke on the Lambda, and the Lambda's role needs `bedrock-agentcore:InvokeAgentRuntime` on the orchestrator's runtime ARN ([invoke docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-invoke-agent.html)). That's the whole chain.

Notice observability is on the diagram, not a footnote. The @codewithbrij write-up made the point that stuck with me: in 2026 you don't bolt observability on at day 100. It runs vertically through the whole thing from day 1, as a rail. On AWS that rail is X-Ray for distributed traces across the full loop (trigger to agent to tool to response), CloudWatch for latency and token metrics, and CloudTrail for the audit log of what the agent decided. If you're doing this for an enterprise, CloudTrail is the row compliance will ask about first.

Which brings up the reason a serious team picks Bedrock at all. Per Prasad Rao, running Claude Code over Bedrock rather than straight against the model adds IAM-based access control, data residency inside your VPC and region, CloudTrail audit on every model call, and unified billing that lands in your existing AWS budget instead of a separate card. None of that is glamorous. All of it is what stands between a demo and something legal will let you ship.

---

## The warning, which is the actual point

Here's where I part ways with most of the "build agents" content. The wiring is the easy part. The hard part is not surrendering your own judgment to a system you built specifically to act without you.

Osmani names three risks, and each one has an AWS-shaped answer, but the answers are guardrails, not cures. Verification stays a human job; the loop does not check itself. AWS gives you AgentCore Evaluations (13 built-in evaluators plus Ground Truth) and you can drop a checkpoint Lambda before any final output, but a human still reads the checkpoint. Comprehension debt piles up faster when the loop is automated, because you stop watching the steps; X-Ray traces and a CloudWatch decision dashboard exist so you *can* watch, not so you don't have to. And the deepest one, what he calls **cognitive surrender**, accepting outputs passively because the system sounds confident: AgentCore Policy plus Guardrails and a Step Functions human-approval step before anything irreversible are the technical brakes. The judgment behind them is still yours.

I feel this one personally, because I run these loops daily. The temptation is real. When the system has been right forty times, you stop reading the forty-first output. That's the moment the engineering fails, and no service on the diagram catches it. Osmani says it better than I can, so I'll give him the close:

> "Build the loop. But build it like someone who intends to stay the engineer, not just the person who presses go."

---

## What I'd tell you to do next

Start smaller than the diagram. One EventBridge rule, one Lambda dispatcher, one AgentCore agent, and DynamoDB for state. Skip the sub-agents, skip the Knowledge Base, skip MCP until the two-box loop runs clean. Add each component only when the loop's actually straining without it. A loop you understand beats a loop that impresses.

If you want to go deeper: take the six-row table above and, for one component, write the ADR. Automations is the easiest place to start. Why EventBridge scheduled rule over a Step Functions timer? What's your circuit breaker in the dispatcher? Write down the decision and the trade-off you rejected, because that's the artifact that makes the loop yours instead of a copy of mine.

And if you build one, come tell me where the six-component map broke for you. Two seams I flagged in an early draft (worktrees, and the exact IAM chain) got run to ground during verification, and both answers made the map stronger. Find me the next one. I want the correction more than I want to be right. That's the whole point of building in public.

Build the loop. Stay the engineer.

---

*Sources: Addy Osmani, ["Loop engineering," LinkedIn (June 9, 2026)](https://www.linkedin.com/posts/addyosmani_ai-softwareengineering-programming-activity-7469999258658791425-cTvy) and ["Agent Harness Engineering," O'Reilly Radar (May 2026)](https://www.oreilly.com/radar/agent-harness-engineering/); @codewithbrij, "The AI Production Stack — May 2026 Edition" (Instagram); Prasad Rao (AWS Principal SA), Claude Code on Bedrock enterprise setup (LinkedIn, May 2026); [AWS MCP Servers](https://aws.amazon.com/blogs/machine-learning/introducing-aws-mcp-servers-for-code-assistants/), [Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html), [Bedrock Agents Classic maintenance mode](https://docs.aws.amazon.com/bedrock/latest/userguide/agents-classic-maintenance-mode.html), [AgentCore release notes](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/release-notes.html), [AgentCore Evaluations GA](https://aws.amazon.com/about-aws/whats-new/2026/03/agentcore-evaluations-generally-available/).*
