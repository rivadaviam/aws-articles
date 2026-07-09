---
title: "How Claude Code Went From \"We Can't\" to \"When Do We Start\" (via AWS Bedrock)"
published: false
description: "Your security team already blocked Claude Code. The AWS Bedrock path turns four hard no's into approvals they'll actually sign."
tags: aws, claude, security, devops
cover_image: ""
canonical_url: ""
---

The same conversation happens in every company that wants Claude Code. An engineer demos it, the room lights up, and then someone from security asks four questions in a row. Where does our code go? Who can reach the model? How do we audit the calls? Whose credit card is this on? Four questions. Four reasons the answer defaults to no.

I run Claude Code every day. Not in a sandbox, in my actual daily work. So when I started reading about the enterprise path, I wasn't asking "is this useful." I already knew. I was asking a narrower question: what changes when the model calls route through AWS Bedrock instead of straight to Anthropic? Because that routing choice is the entire difference between a personal tool and something a security team will sign off on.

Prasad Rao, an AWS Principal Solutions Architect, documented the technical answer back in May 2026, in a LinkedIn write-up that made the rounds. <!-- VERIFY: exact Prasad Rao LinkedIn URL not captured in seed; confirm before publishing --> His framing stuck with me because it maps one-to-one onto those four security questions. Bedrock doesn't add features to Claude Code. It answers objections. That's a different thing, and it matters more.

---

## The four no's, and what actually flips them

Let me take the questions in the order security usually asks them, because the order is itself the story. Each one sounds like a wall. Each one has a control your AWS team already runs for a dozen other services.

**"Our code can't leave our VPC."** This is the first wall and usually the tallest. With Claude Code pointed at the public Anthropic API, requests go out to the internet. Full stop. On Bedrock, inference runs inside your chosen AWS region, and you can add a VPC interface endpoint so the traffic never touches the public internet at all. The data stays where your other regulated workloads already live. Same region, same residency story you already told your auditors.

**"We don't know who can reach the model."** On the direct API, access is an API key. Keys get shared, keys leak, keys are hard to scope. Bedrock replaces the key with IAM. That means access is a role, not a secret. You grant `bedrock:InvokeModel` per team, per project, per environment, with the same condition keys and boundaries you use everywhere else. Revoking access becomes a policy change, not a key rotation scramble.

**"We have no audit trail."** Every inference call through Bedrock lands in CloudTrail. Timestamp, principal, which model, request metadata. Your SIEM already ingests CloudTrail, so the model calls show up next to your S3 reads and your Lambda invocations with no new pipeline to build. The auditor asking "who called Claude at 2am on the payments repo" now has a real answer.

**"We can't manage another subscription."** This one sounds like procurement whining. It isn't. A separate Anthropic bill means a new vendor, a new card, a new line item outside the AWS budget your finance team already approved. On Bedrock it's pay-as-you-go inside the account you already have. No new contract, no new card, no new vendor review. The spend lands under tags you already track.

That's the flip. Four no's become four yeses, and none of the yeses required anyone to invent a new control. They required moving the same call through a door your security team already trusts.

[GRAPHIC: comparison table | Direct Anthropic API vs. Bedrock across the four security controls — data path, access model, audit, billing | The routing choice, not the model, is what unblocks the enterprise]

---

## What Bedrock does NOT fix (the honest part)

The announcement posts skip this part. Routing through Bedrock is a trade, and you pay for it.

You pay in tokens. Bedrock's per-token pricing for Claude can run higher than the direct Anthropic API for the same model, and there's no Claude Pro flat rate hiding on the other side. <!-- VERIFY: confirm current Bedrock per-token rates vs. direct API at https://aws.amazon.com/bedrock/pricing/ before publishing --> If you're a solo developer comparing a $20/month Pro plan to metered Bedrock inference, Bedrock will usually lose on raw cost. That comparison is real, and if someone tells you Bedrock is free, they're selling.

The point is that the comparison changes shape at the enterprise level. When the alternative to Bedrock isn't "$20 Pro plan" but "we can't use the tool at all because security blocked it," the total cost of ownership math inverts. You're not paying extra for tokens. You're paying for the audit trail, the IAM boundary, and the VPC residency that make the tool legal to run in the first place. That's cheaper than a compliance finding.

You also pay in regions. Not every Claude model lands in every AWS region on day one, and the model identifiers you point Claude Code at are region-specific. <!-- VERIFY: confirm which Claude models are available in which Bedrock regions and current model IDs before publishing --> If your compliance story requires a specific region, check model availability there first. I've watched teams design a whole IAM setup and then discover the model they wanted wasn't in the region they needed.

Latency is the open question I can't close from documentation alone. Whether Bedrock adds meaningful round-trip time versus the direct API depends on your region, your endpoint config, and the model. <!-- VERIFY: no benchmarked latency comparison in seed; measure Bedrock vs. direct API round-trip before making any latency claim --> I won't hand you a number I didn't measure. If latency matters for your workflow, benchmark it in your own account before you commit.

---

## The setup, roughly

The seed's raw notes sketch a minimum viable path, and I want to show it while being honest that IAM policies and model IDs drift faster than any blog post can keep up. Treat this as the shape of the work, not a copy-paste script.

```bash
# 1. Enable Claude models in the Bedrock console for your region
#    (us-east-1 / us-west-2 are common starting points; check model
#     availability in YOUR compliance region first)

# 2. Attach an IAM policy. Start least-privilege, not FullAccess.
#    You want InvokeModel on the specific model ARNs you approved,
#    not a blanket grant. The scoped policy is the whole security story —
#    don't undo it with a wildcard.

# 3. Add a VPC interface endpoint for Bedrock so inference traffic
#    stays off the public internet. This is what lets you tell the
#    auditor "it never leaves the VPC" and mean it.

# 4. Confirm CloudTrail is on for the account. It usually is.
#    Now every model call is logged with principal + timestamp.

# 5. Point Claude Code at Bedrock instead of the direct API:
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
export ANTHROPIC_MODEL='<region-specific Bedrock model id>'
# plus standard AWS credentials (role, SSO, or profile)
```

<!-- VERIFY: confirm exact env var names and current Bedrock model ID format against https://docs.anthropic.com/en/docs/claude-code/bedrock — the seed flagged CLAUDE_CODE_USE_BEDROCK vs ANTHROPIC_MODEL naming as changeable -->

Why least-privilege in step 2 and not the managed full-access policy? Because the IAM boundary is the entire reason security said yes. A `BedrockFullAccess` grant is the fast path in a demo and a finding in an audit. Scope `bedrock:InvokeModel` to the specific model ARNs you approved and nothing more. The scoped policy is the thing you're actually selling to the security team. Don't hand it back to them broken.

Here's the decision I'd write down as an ADR if I were setting this up for a team:

```
ADR: Route Claude Code through Bedrock, not the direct Anthropic API

Status: Accepted
Context: Security blocks direct-API tools that egress to the public
  internet, use shared API keys, and bill outside AWS.
Decision: Point Claude Code at Bedrock via VPC endpoint + scoped IAM.
Consequences:
  (+) IAM access control, CloudTrail audit, in-VPC residency,
      unified AWS billing — the four blockers resolved.
  (-) Higher per-token cost than direct API; region-limited model
      availability; latency to be benchmarked per account.
Trade accepted because: the alternative is not "cheaper Claude Code,"
  it's "no Claude Code."
```

---

## The stack is actually complete now

The timing is the part I'd underline. AWS MCP Server reached general availability in the same month as Rao's write-up. <!-- VERIFY: confirm AWS MCP Server GA date (seed says May 2026) against https://aws.amazon.com/blogs/machine-learning/introducing-aws-mcp-servers-for-code-assistants/ --> There's no extra charge for the MCP server itself. You pay for the model usage underneath it, same pay-as-you-go meter. That closes the loop: you get Claude Code, sub-agents, and MCP-based tool access, all inside the AWS boundary your security team already governs.

That's why "when do we start" is a fair reaction and not hype. The pieces that were missing six months ago are shipped. What used to be a roadmap reads like a setup guide now.

If you want the concrete on-ramps, Rao points at four, and they're the right four in the right order. Start with the Anthropic Academy course to get the fundamentals of API calls, tool use, RAG, and MCP servers before you touch any AWS config. Then the Claude Code documentation for the Bedrock-specific IAM setup and troubleshooting. Then the AWS Solutions GitHub sample for real auth patterns, deployment, and usage monitoring you can actually run. Then the AWS Workshop Studio lab for hands-on practice with the Bedrock integration, MCP servers, and sub-agent deployment. <!-- VERIFY: confirm AWS Workshop Studio lab is open to any account vs. event/partner-gated; seed flagged this as an open question --> Fundamentals, then docs, then runnable code, then hands-on. That sequence saves you from configuring things you don't yet understand.

---

## What I'd actually tell a team lead

If you're the person who has to walk into the security review, don't lead with the model. Lead with the controls. The security team doesn't care that Claude Code is good, they care that it's governable. Bedrock lets you answer their four questions with four services they already run.

And keep the trade honest on your side of the table too. You will pay more per token. You may hit a region limit. Benchmark latency before you promise anything. Say all of that in the review. The credibility you build by naming the downsides is what makes the approval hold up three months later, when finance looks at the bill.

Documentation isn't overhead. It's thinking made visible. Writing that ADR before the meeting is how you turn "we can't" into a decision someone can actually sign.

---

Pick the depth that fits where you are right now:

**Low:** Open the [Claude Code Bedrock docs](https://docs.anthropic.com/en/docs/claude-code/bedrock) and check whether the model you want is available in your compliance region. Five minutes, and it's the fact most likely to sink a setup later.

**Medium:** Run the [AWS Workshop Studio lab](https://catalog.us-east-1.prod.workshops.aws/) or the [AWS Solutions GitHub sample](https://github.com/aws-samples/) in a scratch account. Wire up the scoped IAM policy and watch a real inference call show up in CloudTrail. That single log line is the whole security pitch in one screenshot.

**High:** Take the four-objection framing into your next architecture review and write the ADR first. Then come back and tell me where it broke. Which of the four no's was hardest to flip at your company? Did the token cost land where you expected? I read every comment, and the wrong turns other people hit are the most useful thing in this whole thread.

Build. Document. Share. Repeat.
