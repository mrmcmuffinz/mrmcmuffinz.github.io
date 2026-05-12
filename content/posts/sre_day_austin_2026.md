+++
date = '2026-05-12T08:00:00-05:00'
draft = false
title = 'My Experience at SRE-Day Austin 2026'
tags = ["sre", "ai", "kubernetes", "conference"]
categories = ["devops"]
description = "Reflections from SRE-Day Austin 2026, standout talks on AI and SRE work, and questions about what the modern SRE role looks like."
layout = "post"
+++

# My Experience at SRE-Day Austin 2026

Welcome back friends. On May 12, 2026, I attended SRE-Day Austin, a DevOps-focused conference exploring the intersection of site reliability engineering and modern tooling. I recorded the full experience, and you can watch the full video recap below if you want to see the raw thoughts right after the event. This post is my written take on what stood out, what I learned, and what I wish had been there. First off, thanks to the coordinators, Miko and Mark Pawlowski, two brothers from the EU who put this together. The event had several speakers, and while some were more engaging than others, I found value in most of them. I want to talk about three presentations that particularly resonated with me, and then share some broader thoughts about what it's like trying to break into SRE work in 2026.

{{< youtube "IYuIhiKi5eE" >}}

## Three Presentations That Stood Out

## Presentation 1: Miko on the Seven Deadly Traps of SRE

Miko's talk covered the Seven Deadly Traps of SRE, and one of those traps hit particularly close to home for me: the hero trap. The setup is that there are people on SRE teams who act as heroes, always fixing things, always the ones responding to incidents. It's typically just one or two people on a team, and every time there's an on-call rotation, that person is always engaged. The team and management end up incentivizing this behavior, rewarding the hero mentality even though it creates long-term problems. The reason this resonated with me is because I was that person. I was the one always being called, always fixing the problems, and after listening to Miko's arguments and the data he presented, I realized how unhealthy that pattern was. The burnout that comes from being the hero, the lack of knowledge distribution across the team, the single point of failure you create by being the only one who knows how to fix certain issues, all of that was detrimental not just to me but to the team as a whole. Hearing it framed that way made me realize I need to approach on-call and incident response differently going forward.

## Presentation 2: Michael Forrester's AI Agent Disaster

Michael Forrester's presentation was about an incident in his homelab where an AI agent he'd given access to his Kubernetes cluster essentially tore down and rebuilt the entire system while he was cooking dinner. He'd been working on this homelab k8s cluster for months, and one day he decided to expose more services to an AI agent so it could help with tasks. The incident happened when he told the agent to handle something while he stepped away, the kind of multitasking that remote workers do all the time. The difference is that his AI agent had nearly administrator-level privileges and minimal RBAC constraints, so when it decided the best way to complete the task was to rebuild the infrastructure, it just did it. Despite having some guardrails in place, the agent circumvented them, and Michael came back to find his cluster in a very different state than when he left. You can find the full details in his [GitHub repo for the presentation](https://github.com/peopleforrester/SREday-Texas-2026), but the core lesson was about the dangers of giving AI agents direct cluster access without proper constraints.

My takeaway from this wasn't just "use better RBAC," which is obvious in hindsight but easy to miss when you're building something in your homelab and moving fast. My takeaway was questioning whether AI agents should have direct kubectl access at all, even in a homelab. Why not have the agent interact with something like Argo CD or Flux instead, where it modifies GitOps YAML files or Helm charts, submits them to a repo, and lets the GitOps tool apply the changes? That way you get version control, audit trails, and the ability to revert bad changes before they hit the cluster. It's a better safety mechanism than giving an AI direct cluster access and hoping your guardrails hold. Michael's approach wasn't incorrect for a homelab experiment, but it raises serious questions about how we safely integrate AI agents into production systems and what the boundaries should be. Those are conversations I think the industry needs to have soon, because these tools are getting more capable and more accessible, and not everyone is going to think through the failure modes before giving an agent admin privileges.

## Presentation 3: Whitney Lee on cluster-whisperer

Whitney Lee's presentation was a counterpoint to Michael's cautionary tale. She demoed an AI agent called `cluster-whisperer` (available on [GitHub](https://github.com/wiggitywhitney/cluster-whisperer)) that lets you ask questions about your Kubernetes cluster in plain English, and it investigates using kubectl, searches a vector database of cluster knowledge, and explains what it finds. It's available as a CLI tool for direct terminal use, as an MCP server for integration with Claude Code and Cursor, or as a REST API, which makes it flexible enough to fit into different workflows. This is what thoughtful AI integration in SRE work looks like: instead of manually running multiple kubectl commands to investigate an issue, you ask natural language questions and the tool figures out what to investigate and explains the findings. It's scoped, it's useful, and it shows what a modern AI-augmented SRE workflow could look like without handing over the keys to the kingdom.

## Other Notable Presentations

There were several other presentations throughout the day that stuck with me. **Aaron Hunter** built an application called "bee hunter" to help his kids practice spelling using AI. The setup is simple: your child writes out their spelling words on paper, you take a picture and upload it to the app, and it uses AWS Textract and Claude to read what they wrote, grade whether they spelled it correctly, identify misspellings, and give feedback on how to improve. I joked with him during the presentation that he'd helped us build a tool to bypass captchas, which got a laugh, but it's actually a clever example of AI integration for a very practical everyday problem. Sometimes the best use cases for AI aren't the ones you'd find in a technical roadmap, they're the ones that solve real friction in daily life.

**Eric Tschetter** gave a talk about postmortems and incident management that touched on a lot of the fundamentals: whether teams have postmortems, how to handle ongoing outages, who to engage, and what good alerting looks like. One particularly interesting point he made was about using AI during incidents. Since many teams have recorded lines to a bridge or war room during outages, you could transcribe those conversations and use AI to summarize the postmortem, capture takeaways, and identify actions that should be taken. It's another example of how AI can augment SRE work without replacing the human judgment and context that matters during an incident. Across all the talks, there were these bits and pieces about what modern SRE looks like with AI in the picture, but I kept wishing there had been one dedicated session that pulled it all together and said, "Here's what we're aspiring to build."

## What I Wish Had Been There

One thing I kept waiting for was a dedicated discussion about what it actually means to be an SRE in 2026, especially with AI tools becoming more integrated into the workflow. Not a retrospective on how we've traditionally done SRE work, but a forward-looking conversation about where the role is heading and what companies are actually looking for when they hire SREs today. There were bits and pieces throughout the day, like Eric's talk on postmortems and incident management, and Whitney's cluster-whisperer demo showing what an AI-augmented SRE workflow could look like, but I wish there had been one cohesive session or panel that pulled it all together and said, "Here's what we're aspiring to. Here's what the modern SRE role looks like now that AI agents can write kubectl commands and summarize postmortems." I'm personally looking for SRE and platform engineering roles right now, so I came to this conference hoping to understand what the market is looking for, where the trends are heading, what skills matter in 30, 60, 90 days, six months from now. The SRE I was five years ago isn't the same SRE that companies need today, and I don't want to be left behind because I'm still thinking in terms of yesterday's tooling and workflows. A panel or dedicated talk addressing that would have been valuable not just for me but for anyone trying to break into or stay relevant in this space as it evolves.

If you attended SRE-Day Austin or have thoughts on what the modern SRE role looks like with AI in the picture, I'd love to hear from you. You can find me on [LinkedIn](https://www.linkedin.com/in/abrahamcabrera/).

Well, that's all. See you in the next one.

**P.S.** I used Claude Code to help revise and polish this post. The experience, observations, and opinions are my own.
