---
title: "AWS Runs Kiro and Amazon Q on Strands. Why Isn't It in Your Stack Evaluation?"
published: true
description: "25M downloads. Cox Automotive runs 17 agents on it. AWS ships its own products on it. So why does every framework comparison skip Strands?"
tags: aws, aiagents, generativeai, learninpublic
cover_image: "https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/strands-agents-aws/00-cover.png"
canonical_url: ""
---

Open any "which agent framework should I use" post from the last six months. You'll see LangGraph. You'll see CrewAI. Maybe LlamaIndex, maybe OpenAI's Agents SDK. You almost never see Strands.

That's odd, because Strands is the framework AWS trusts with its own products. Kiro runs on it. So do Amazon Q, AWS Glue, and AWS Transform for .NET ([AWS's own announcement](https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/) names them). It crossed [25 million downloads](https://aws.amazon.com/blogs/opensource/) by its one-year anniversary in May 2026, hit v1.7.0 on June 25, and sits at 6.4k GitHub stars. Cox Automotive runs 17 production agents on it. This is not a science project.

So the gap isn't coverage anymore. A year ago you could barely find a tutorial. Now there's a Medium series, an InfoQ writeup, YouTube playlists, the works. The gap moved. It's an *evaluation* gap. Everyone benchmarks LangGraph against CrewAI and quietly leaves out the framework AWS runs Kiro on. If you're picking an agent framework on AWS and Strands isn't even on your shortlist, that's worth a second look.

Full disclosure before we go further. I have not shipped a production system on Strands. I have run every code example in this article against Bedrock Nova (they work as printed; 2-3 seconds per agent call, under a cent total), read the documentation, and pulled together what the adopter evidence actually shows so you don't start from a blank page. When I'm guessing, I'll say so.

---

## What Strands actually is

Strands is an open-source SDK from AWS for building AI agents. The pitch, once you strip the marketing, is model-driven agents: you give the model a set of tools and a prompt, and the model itself decides how to plan, when to call a tool, and when it's done. You are not hand-wiring a state machine of "if the model says X, call Y." The loop that connects the model to its tools is the framework's job.

That framing matters because it's the opposite of how a lot of people first build agents. My own instinct, and probably yours, is to reach for something explicit. Step Functions. A big orchestration graph. Nodes and edges you can point at. Strands bets that the model is good enough now that most of that scaffolding is wasted effort.

The core of a Strands agent is small enough to fit in your head:

```python
from strands import Agent
from strands_tools import calculator

# The agent is: a model + a set of tools + a prompt.
# You don't write the plan-act-observe loop. The SDK runs it,
# and the MODEL decides which tools to call and when to stop.
agent = Agent(tools=[calculator])

agent("What is 3111696 divided by 48?")
```

That's the whole thing. No graph. No explicit routing. Defining your own tool is a decorator on a Python function, which is the part that clicked for me. The docstring and type hints become the tool's contract that the model reads:

```python
from strands import Agent, tool

@tool
def get_invoice_total(account_id: str) -> float:
    """Return the current month's invoice total for an AWS account.

    Args:
        account_id: the 12-digit AWS account number
    """
    # Your real logic goes here. The docstring above is what the
    # model actually reads to decide when to call this tool, so the
    # docstring IS the interface. Write it for the model, not just for humans.
    return 1284.50

agent = Agent(tools=[get_invoice_total])
agent("How much did account 123456789012 spend this month?")
```

The interesting design choice is that the docstring is not documentation sitting next to the code. It *is* the interface the model sees. Sloppy docstring, sloppy tool use. That single detail tells you the whole framework leans on the model's judgment instead of your control flow.

---

## Why AWS built this instead of pointing you at Bedrock Agents

This was my first real question. AWS already has Bedrock Agents, a managed service where you define an agent in the console, attach action groups, and let Bedrock run it. So why ship a separate SDK?

The honest answer, from what I can piece together, is that they solve different problems. Bedrock Agents is managed and console-first. It's great until you want your agent logic to live in your codebase, in version control, tested like the rest of your application, running wherever your code runs. Strands is code-first and model-agnostic. It works with Bedrock and Nova, but the SDK is not locked to a single model provider.

Here's the mental model, and this one I can back with AWS's own words. Think of it the way infrastructure tooling splits. The runtime that executes your workload is one layer. The framework you write to define that workload is another. Strands is the framework you write. AgentCore is the managed runtime that hosts and scales it. The [AgentCore FAQ](https://aws.amazon.com/bedrock/agentcore/faqs/) says it directly: AgentCore Runtime deploys and scales agents built with "any open-source framework (such as CrewAI, LangGraph, LlamaIndex, Google ADK, OpenAI Agents SDK, or Strands Agents)." You author with Strands. You deploy and run at scale with AgentCore.

That split explains why the two names keep showing up together. You author with the framework, then deploy with the runtime. It also explains why "Strands vs. AgentCore" is the wrong question. They're not competing. One writes the agent, the other runs it at scale.

![Layered diagram: Bedrock models at the bottom, Strands SDK as the authoring layer you write, AgentCore as the managed runtime that runs it](https://raw.githubusercontent.com/rivadaviam/aws-articles/main/articles/assets/strands-agents-aws/01-strands-agentcore-layers.png?v=2)
*The split that makes "Strands vs. AgentCore" the wrong question: you write one, the other runs it.*

---

## Getting to a first agent

The setup path is standard Python, and standard AWS credentials, which is a relief after some agent frameworks that ask you to adopt a whole platform first.

```bash
# Strands is just a Python package. No console clicking to get started.
pip install strands-agents strands-agents-tools

# It talks to Bedrock through your normal AWS credentials.
# Same profile, same region, same IAM you already use for boto3.
export AWS_PROFILE=your-profile
export AWS_REGION=us-east-1
```

Here's a detail that will save you a confused hour: most tutorials still tell you to go enable model access in the Bedrock console first. That advice is stale. Since [October 2025](https://aws.amazon.com/about-aws/whats-new/2025/10/amazon-bedrock-automatic-enablement-serverless-foundation-models/), Bedrock enables serverless foundation models automatically in every commercial region. The manual "Model access" page is gone. The one exception: Anthropic models still ask for a one-time usage form before first use. Nova models need nothing. If a guide sends you hunting for a console page that no longer exists, that's the guide's age showing, not your mistake.

With credentials sorted, pointing the agent at a specific model is explicit:

```python
from strands import Agent
from strands.models import BedrockModel

# Being explicit about the model is a cost and capability decision,
# not a detail. Nova Pro is AWS's own model and is cheaper per token
# than the top-tier Claude models; a strong Claude model reasons harder
# but costs more. For an agent that loops and calls tools many times
# per task, that per-call cost multiplies fast.
model = BedrockModel(
    model_id="us.amazon.nova-pro-v1:0",  # US cross-region inference profile
    temperature=0.3,  # lower temp: more deterministic tool selection,
                      # which is usually what you want for agents that act
)

agent = Agent(model=model, tools=[get_invoice_total])
```

That `temperature=0.3` is a deliberate choice worth explaining. For a chatbot you might want warmth and variety, so you run it hot. For an agent that picks tools and takes actions, you want it boring and repeatable. High temperature means the model might improvise a different tool sequence run to run, and non-determinism in something that touches your systems is a debugging nightmare. Lower temperature, more predictable actions.

Pairing Strands with Nova Pro is the same kind of decision made at the model layer. Nova is AWS's own model family, and it tends to cost less per token than the premium models. When your agent loops, plans, calls a tool, reads the result, and plans again, you pay for every hop. A model that's marginally smarter but several times more expensive can quietly turn a cheap task into a real bill. Running the agent on Nova reads as an intentional cost posture, not just AWS promoting its own model.

---

## Where I got skeptical

I don't want to write the breathless "this changes everything" piece, because that's exactly the content this topic doesn't need. So here's my honest doubt.

Model-driven control flow is elegant right up until you need to know *why* the agent did something. When the model owns the plan, your debugging surface changes. You are no longer reading a state machine. You are reading a transcript of what the model decided, and asking why it decided that. That's a real trade-off, and the docs are understandably optimistic about it. I'd want to see observability and tracing in a genuine multi-tool workload before I trusted it with anything that spends money or mutates data.

The other open question is what kind of production evidence exists. There's plenty of it now, and that's the good news. Smartsheet presented a Strands session at re:Invent 2025 (BIZ210). Cox Automotive runs 17 production agents on it. Jit, Landchecker, and Verisk show up in [AWS's own writeup](https://aws.amazon.com/blogs/machine-learning/enabling-customers-to-deliver-production-ready-ai-agents-at-scale/) on delivering production agents at scale. And AWS runs Kiro, Amazon Q, and Glue on it, which is about as strong a bet as a vendor can place on its own tooling.

Here's the caveat I'd keep front and center. Almost all of that evidence is AWS-authored. It's the vendor telling you its framework works, which is exactly what a vendor does. The independent post-mortem, the "we picked Strands over LangGraph for these reasons and here's where it hurt," barely exists yet. Setup tutorials, yes. Marketing vignettes, yes. A neutral engineer walking through six months of running it in anger, no. So the coverage gap closed, but a depth gap opened in its place. Being early to that depth still cuts both ways. You get the clean runway. You also get to find the sharp edges first.

---

## Why this belongs in your evaluation

Step back from the code for a second. The reason to put Strands on your shortlist isn't that it beats LangGraph. I haven't run that comparison honestly enough to claim it, and neither has most of the internet yet. That's the part worth sitting with.

Frameworks are consolidating fast in 2026. AutoGen already got absorbed into Microsoft's Agent Framework. The market clearly wants fewer, cloud-native options. A first-party AWS framework with 25M downloads, a year of releases, and AWS running its own flagship products on it is a strong candidate to be one of the survivors. Leaving it out of a framework comparison because you hadn't heard of it a year ago is a decision you're making by accident.

Documentation isn't overhead. It's thinking made visible. Right now the thinking that's visible about Strands is AWS's own. Nobody outside the vendor has weighed it honestly against the frameworks everyone already benchmarks. That's the messy middle, and the messy middle is where learning lives.

---

## Try it, then tell me where I'm wrong

If you have ten minutes: `pip install strands-agents`, run the calculator example, and watch the model pick the tool on its own. That's the whole "aha" in one command.

If you have an afternoon: build one real tool for something you actually use, an internal API, a cost lookup, a status check, and see how far the docstring-as-interface idea carries you before it strains.

If you have a weekend and you already run agents in production: put Strands next to LangGraph or CrewAI on a task you actually care about, and write down where each one hurt. That comparison is the missing artifact. Every framework roundup skips Strands, and every Strands writeup skips the comparison. Whoever publishes an honest head-to-head fills the one hole in the whole conversation.

I'll be doing the same, in the open, wrong turns included. If you've already built something on Strands, or you think my framework-versus-runtime read of the AgentCore relationship is off, tell me in the comments. I'd rather be corrected early than confident and wrong. Build. Document. Share. Repeat.

---

*Sources: [Introducing Strands Agents, an open-source AI agents SDK](https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/) (AWS, names Kiro, Amazon Q, Glue, Transform for .NET) · [Enabling customers to deliver production-ready AI agents at scale](https://aws.amazon.com/blogs/machine-learning/enabling-customers-to-deliver-production-ready-ai-agents-at-scale/) (AWS ML Blog, named adopters) · [Strands Agents documentation](https://strandsagents.com/) · [Strands Agents SDK on GitHub](https://github.com/strands-agents/sdk-python) · [Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) · [Amazon Nova](https://aws.amazon.com/ai/generative-ai/nova/)*
