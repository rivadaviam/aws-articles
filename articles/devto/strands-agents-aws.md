---
title: "AWS Built Its Own Agent Framework. Almost Nobody Is Writing About It."
published: false
description: "Strands sits in week 3 of AWS's most-followed agent curriculum. Search for it and you find the docs and silence. Here's what I found."
tags: aws, aiagents, generativeai, learninpublic
cover_image: ""
canonical_url: ""
---

Search "Strands agents AWS" today. You get the official docs, a GitHub repo, and then a wall of nothing.

That is strange for a framework AWS put at the center of its own agent curriculum.

Here's how I ran into it. Prasad Rao, a Principal Solutions Architect at AWS, posted a six-week roadmap in May 2026 for learning to build AI agents on AWS. The post pulled 693 reactions. He is one of the most-followed AWS voices on LinkedIn for agent work, so when he lays out an ordered learning path, it reads less like an opinion and more like a map of where AWS thinks this is going.

Week 3 of that map says one thing: **Strands agents framework.**

Week 4 builds on it: Bedrock AgentCore, Strands, and the Nova Pro model, stacked together.

So Strands is not a footnote. It shows up before AgentCore in the sequence, which means AWS's own SA treats it as something you learn *first*, then build on. And yet the community coverage sits at roughly zero. LangChain has thousands of articles. LlamaIndex has thousands. AutoGen had thousands before it got folded into Microsoft's Agent Framework. Strands has the docs and a quiet corner of GitHub.

Full disclosure before we go further. I have not shipped a production system on Strands. Nobody I can find has published one either. What I did was read the documentation, poke at the SDK, and follow the roadmap so you don't start from a blank page. Call it the "I went and looked, here's the lay of the land" article, written before the topic gets crowded. When I'm guessing, I'll say so.

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

<!-- VERIFY: confirm current import paths and package names (strands / strands_tools) against https://strandsagents.com/ and https://github.com/strands-agents/sdk-python before publishing -->

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

<!-- VERIFY: confirm the @tool decorator API, argument-passing behavior, and that docstring-as-schema is accurate per current SDK docs -->

The interesting design choice is that the docstring is not documentation sitting next to the code. It *is* the interface the model sees. Sloppy docstring, sloppy tool use. That single detail tells you the whole framework leans on the model's judgment instead of your control flow.

---

## Why AWS built this instead of pointing you at Bedrock Agents

This was my first real question. AWS already has Bedrock Agents, a managed service where you define an agent in the console, attach action groups, and let Bedrock run it. So why ship a separate SDK?

The honest answer, from what I can piece together, is that they solve different problems. Bedrock Agents is managed and console-first. It's great until you want your agent logic to live in your codebase, in version control, tested like the rest of your application, running wherever your code runs. Strands is code-first and model-agnostic. It works with Bedrock and Nova, but the SDK is not locked to a single model provider.

Here's the mental model I landed on, and I'll flag it as my interpretation rather than gospel. Think of it the way infrastructure tooling splits. The runtime that actually executes your workload is one layer. The framework you write to define that workload is another. Strands is the framework you write. AgentCore, which shows up in week 4, is closer to the managed runtime and operational plumbing that hosts and scales what you built. You author with Strands. You deploy and run at scale with AgentCore.

<!-- VERIFY: the Strands-as-framework / AgentCore-as-runtime relationship is my inference from the roadmap ordering and product docs, not an official AWS statement. Confirm against AgentCore docs (https://aws.amazon.com/bedrock/agentcore/) or soften the claim before publishing. -->

The roadmap ordering backs this up, at least circumstantially. If Strands were just a thin wrapper over AgentCore, you would not teach it first. You teach the authoring layer before the deployment layer because you need something to deploy. That is why I read week 3 (Strands) coming before week 4 (AgentCore) as a signal, not an accident.

[GRAPHIC: layered diagram | Nova/Bedrock models at the bottom, Strands SDK as the authoring layer in the middle, AgentCore as the managed runtime on top, with "you write this" pointing at Strands and "this runs it" pointing at AgentCore | the layers are complementary, not competing]

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

<!-- VERIFY: confirm exact package names on PyPI (strands-agents vs strands) and that Bedrock model access must be enabled in the console for the target region/model -->

The one thing that will bite you here has nothing to do with Strands and everything to do with Bedrock. If you have never turned on model access for your account, your first agent call fails with an access error, not a code error. You request model access in the Bedrock console per region, per model. I've watched people burn an hour debugging their Python when the fix was three clicks in the console. Check that first.

Once credentials and model access are sorted, pointing the agent at a specific model is explicit:

```python
from strands import Agent
from strands.models import BedrockModel

# Being explicit about the model is a cost and capability decision,
# not a detail. Nova Pro is AWS's own model and is cheaper per token
# than the top-tier Claude models; a strong Claude model reasons harder
# but costs more. For an agent that loops and calls tools many times
# per task, that per-call cost multiplies fast.
model = BedrockModel(
    model_id="amazon.nova-pro-v1:0",
    temperature=0.3,  # lower temp: more deterministic tool selection,
                      # which is usually what you want for agents that act
)

agent = Agent(model=model, tools=[get_invoice_total])
```

<!-- VERIFY: confirm the BedrockModel import path, the exact Nova Pro model_id string, and current relative pricing of Nova Pro vs Claude models on Bedrock before stating cost claims -->

That `temperature=0.3` is a deliberate choice worth explaining. For a chatbot you might want warmth and variety, so you run it hot. For an agent that picks tools and takes actions, you want it boring and repeatable. High temperature means the model might improvise a different tool sequence run to run, and non-determinism in something that touches your systems is a debugging nightmare. Lower temperature, more predictable actions.

The Nova Pro choice in week 4 of the roadmap is the same kind of decision made at the model layer. Nova is AWS's own model family, and it tends to cost less per token than the premium models. When your agent loops, plans, calls a tool, reads the result, and plans again, you pay for every hop. A model that's marginally smarter but several times more expensive can quietly turn a cheap task into a real bill. Pairing Strands with Nova reads as an intentional cost posture, not just AWS promoting its own model.

---

## Where I got skeptical

I don't want to write the breathless "this changes everything" piece, because that's exactly the content this topic doesn't need. So here's my honest doubt.

Model-driven control flow is elegant right up until you need to know *why* the agent did something. When the model owns the plan, your debugging surface changes. You are no longer reading a state machine. You are reading a transcript of what the model decided, and asking why it decided that. That's a real trade-off, and the docs are understandably optimistic about it. I'd want to see observability and tracing in a genuine multi-tool workload before I trusted it with anything that spends money or mutates data.

The other open question is production evidence. I went looking for companies running Strands in production and came up mostly empty. That's not damning. It's a young framework. But it means anyone adopting it right now is early, and being early cuts both ways. You get the clean search-traffic runway and the AWS-notices-its-own-framework upside. You also get to be the one who finds the sharp edges first.

<!-- VERIFY: check for any published production case studies or notable adopters of Strands before publishing; if found, cite them; if not, the "mostly empty" claim stands -->

---

## Why this is worth your attention now

Step back from the code for a second. The reason to care about Strands is not that it's objectively better than LangChain. I have not run that comparison honestly enough to claim it, and neither has most of the internet yet. The reason to care is the gap.

AWS assigns Strands enough weight to put it in week 3 of its flagship agent roadmap. The community has assigned it almost none. That gap is the whole story. It's the same gap that existed around every framework the year before it got crowded. Frameworks are consolidating fast in 2026, AutoGen already absorbed, LangChain under pressure, and the market clearly wants fewer, cloud-native options. A first-party AWS framework sitting in the official curriculum is a strong candidate to be one of the survivors.

Documentation isn't overhead. It's thinking made visible. Right now the visible thinking about Strands is a docs site and a handful of GitHub examples. That's the messy middle, and the messy middle is where learning lives. Going through it in public, before the topic is picked clean, is the entire point of building to learn.

---

## Try it, then tell me where I'm wrong

If you have ten minutes: `pip install strands-agents`, run the calculator example, and watch the model pick the tool on its own. That's the whole "aha" in one command.

If you have an afternoon: build one real tool for something you actually use, an internal API, a cost lookup, a status check, and see how far the docstring-as-interface idea carries you before it strains.

If you have a weekend and you're following Prasad Rao's roadmap anyway: do week 3 properly, wire Strands to Nova Pro, and write down what surprised you. That article does not exist yet. You could be the one who writes it.

I'll be doing the same, in the open, wrong turns included. If you've already built something on Strands, or you think my framework-versus-runtime read of the AgentCore relationship is off, tell me in the comments. I'd rather be corrected early than confident and wrong. Build. Document. Share. Repeat.

---

*Sources: [Prasad Rao's six-week AWS agents roadmap](https://www.linkedin.com/posts/kprasadrao_want-to-gain-deep-hands-on-experience-on-activity-7462182719771549698-24n9) (LinkedIn, May 2026, 693 reactions) · [Strands Agents documentation](https://strandsagents.com/) · [Strands Agents SDK on GitHub](https://github.com/strands-agents/sdk-python) · [Amazon Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) · [Amazon Nova](https://aws.amazon.com/ai/generative-ai/nova/)*
