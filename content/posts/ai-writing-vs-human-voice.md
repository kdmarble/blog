---
title: "AI Writing vs Human Voice"
date: 2026-04-19
draft: false
tags: ["ai", "writing", "public-voice", "red-line"]
---

Steven Levy's latest Backchannel in WIRED puts two reporters side by side: Alex Heath, who's trained an AI to sound like him and now one-shots columns from his notes, and Nick Lichtenberg, who's filed 600 stories for Fortune since July and hit seven bylines in a single February day via a headline → Perplexity draft → light edit → publish pipeline.[^1] Each has a defense ready: "the hard work is the thinking, not the typing"; "AI-assisted, not AI-written", and both phrases are doing a lot of (_ahem_) labor.

The generational read is interesting, too. Heath himself says the people in media who most hate what he's doing are 25-29.[^1] Gen Z frames AI as a thief coming for their careers before those careers can start. Yet another once-in-a-lifetime event, stacked on a pandemic, a climate that keeps breaking its own records, and a media economy that's been eating itself for a decade. And for most people outside the bleeding edge, AI equates to slop. Low-effort grabs for attention in an attention economy.

There's a certain type of person who loves building on the forefront of technology. The intellectually curious. The ones using a bit of technical know-how to squeeze the best out of their agents right now. I like to imagine a world where everyone builds their own personal agent that augments their best abilities. How do we help each other use these tools to be more human? To better ourselves, not just replace drudgery? Writing is thinking made legible, which is suddenly becoming harder to see in a world with more and more powerful "reasoning" models. Optimizing the writing away optimizes the thinking away with it.

In practice, I use my agents for research, summarization, and as a processing layer for the firehose of information the modern web has become. Hermes[^2] is legit. It's been my personal AI agent for months now and it's held up: chat runs on qwen3.5-122b on a Strix Halo, cron tiers run qwen3.6-35b on a 7900 XTX. It processes RSS, Reddit, and HN feeds so I don't have to. It scans arXiv, watches logs, handles admin tasks (docker pulls, web research, building and extending itself). The single most useful thing it does is keep me off Reddit. I stay informed without getting overwhelmed.

It'll never write articles for me. I'm always responsible for the final output. Human in the loop, built by both, but *for* humans. This isn't boomer affectation: it's about who we are as communicators, and how we help one another through an era of immense upheaval and immense potential. I'm intellectually curious and staunchly humanity-first. I haven't figured this all out, and I want to keep exploring it. This is where I sit as of 2026-04-19. The line might move, but I'm drawing it now.

What's your red line?

[^1]: Steven Levy, "The Problem With Letting AI Do the Writing," WIRED Backchannel, 2026. https://www.wired.com/story/backchannel-the-problem-with-letting-ai-do-the-writing/

[^2]: Hermes Agent, an open-source personal AI framework by Nous Research (MIT). I run a lightly patched fork on my homelab. https://github.com/NousResearch/hermes-agent
