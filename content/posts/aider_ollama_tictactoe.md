+++
date = '2026-04-03T12:00:00-05:00'
draft = false
title = 'I Tested Aider with a Local AI Model and It Was... Fine'
tags = ["ai", "aider", "ollama", "ubuntu"]
categories = ["devops"]
description = "I ran Aider with qwen2.5-coder:14b via Ollama on my desktop to build a tic-tac-toe game, although it worked I certainly didn't have a complete and cohesive experience."
layout = "post"
+++

Welcome back friends. I've been using Claude Code for a while now and I wanted to know what the local, self-hosted version of that experience looks like. Not a cloud API, not a subscription, just a model running on my own GPU, connected to a CLI tool that reads and edits my project files. The tool I landed on was **Aider**, and the model was `qwen2.5-coder:14b` served through **Ollama** on my Ubuntu 24.04 desktop. I gave it a simple task (build a tic-tac-toe game in Python), recorded the whole thing, and walked away with a working game and a list of caveats. This post is about what happened.

{{< youtube "-tr-xig1PPw" >}}

## Why This Experiment

I've been curious about local AI-assisted coding for a while. Cloud-hosted tools like Claude Code are powerful, but they cost money per token, they require an internet connection, and your code leaves your machine. There's also the question of what happens if the service goes down, or the provider raises prices after you've built your workflow around them, or worse, goes under entirely. The AI space is moving fast and not every company offering coding tools today will be around in five years. Having a local option that works, even if it's not as capable, means you're not completely dependent on any single provider. There's an appeal to running everything locally, especially when you already have capable hardware sitting on your desk. My desktop has a Ryzen 9 9900X, 64GB of RAM, and an RTX 5060 Ti with 16GB of VRAM, which is enough to run a 14-billion parameter coding model comfortably. I wanted to see how far that setup could take me with a real coding task, even a simple one.

Aider caught my attention because it works similarly to Claude Code in concept. It's a CLI tool that connects to a language model and can read, write, and edit files in your project. You chat with it, describe what you want, and it makes the changes. The difference is that Aider supports local model backends through Ollama, which means you can point it at a model running on your own hardware instead of hitting an API.

## The Prep Work

Before I recorded my video I spent some time getting everything ready. I wrote an initial prompt (v1) describing the tic-tac-toe game I wanted, then took that prompt to claude.ai for feedback. There's some irony in using a cloud AI to improve a prompt destined for a local AI, but here we are. Claude suggested adding input validation and error handling, specifying the board display with a numbered reference grid, including a turn indicator, and adding a "play again" option. I went through a few rounds of formatting with Claude, restructuring the improvements into my original format and adding the context section back in. That became v2, the prompt I actually used in the video.

I also set up the supporting infrastructure in a [GitHub repo](https://github.com/mrmcmuffinz/ai-station/tree/main/aider). That includes a `compose.yml` for running Ollama in a container with GPU passthrough, a healthcheck script, and the README documenting the setup steps. All of that was in place before recording started so the video could focus on the actual Aider interaction rather than environment setup.

## The Setup

The stack is straightforward. Ollama runs in a container launched with `nerdctl compose up -d`, configured with CUDA device access and flash attention enabled. Before starting Aider I preloaded `qwen2.5-coder:14b` into VRAM with a `curl` request that sets a 3-hour keep-alive, so the model stays warm and responsive throughout the session. Aider itself runs in a pyenv virtualenv on Python 3.13.9, pointed at the local Ollama endpoint through environment variables. At the time of recording I was using Aider v0.86.2.

## What Happened

I loaded the v2 prompt into Aider's ask mode and let it generate the tic-tac-toe game. The model produced a working game, which is the good news. Two players can take turns on the same terminal, the board displays after each move, and it detects wins and draws. The core game logic came out correct on the first pass, and for a 14B parameter model running on consumer hardware, that's genuinely impressive. A year or two ago you would have needed a cloud API for that kind of output quality. But "working" and "done" are different things, and the gap between them is where this experiment got interesting.

## Where It Fell Short

The output formatting wasn't quite right. The game worked, but the terminal output didn't look the way I wanted. I could have prompted Aider to fix it, but honestly it was faster to just open the file and make the changes myself. Sometimes the most efficient path with AI coding tools is knowing when to stop prompting and start typing, and this was one of those cases.

Input validation was the first real problem. The game accepts numbers 1 through 9 for board positions, but the initial implementation would accept an out-of-range input and tell you the position was "already taken" instead of telling you the input was invalid. That's the kind of bug that seems minor but would confuse any actual player. I had to prompt specifically for that fix, and it did eventually get it right, but the fact that v2 of my prompt explicitly asked for input validation and error handling and the model still got it incorrect on the first pass is worth noting.

The unit tests were where things really got bumpy. I asked Aider to generate simple unit tests for the game, and the tests it produced had a bug. That took multiple rounds of prompting to resolve, which is a frustrating experience in any AI coding tool but especially in a local one where you're waiting for each response to generate on your GPU. More importantly, the tests expected to import the game code as a Python module, but Aider had never set up the project with a proper module structure. It may have mentioned it briefly at the beginning of the session, but it never circled back to it or made the necessary changes. I ended up copying the dependent code directly into the test file just to get the tests running. It works, but it's not how you'd want to structure a real project, and the fact that the AI didn't flag the disconnect between its test code and the actual project layout is a gap in its reasoning.

## The Honest Take

A 14B local model through Aider can write a working Python game from a well-crafted prompt, and that's genuinely useful. But the experience has a texture to it that's different from using a frontier cloud model, and the difference isn't where I expected it to be. The core game logic came out correct on the first pass, which means the model understands Python and can reason about basic algorithms just fine. Where it struggled was everything around the code: setting up a proper project structure, keeping its test imports consistent with how the files were actually organized, handling edge cases in input validation that the prompt specifically asked for. Those secondary concerns require the model to hold a bigger picture of the project in its head, tracking not just the current file but how all the pieces fit together, and a 14B model running locally is going to have a harder time with that than a much larger model running in the cloud.

Prompt quality matters more with smaller models too, and that's something I need to own. The jump from v1 to v2 of my prompt, informed by Claude's feedback, made the output noticeably better. With a frontier model you can afford to be vaguer because it'll fill in the gaps and infer what you probably meant. With a 14B model you need to be much more specific about what you want, and even then you might need to intervene when it misses something. That tells me I should probably invest more time in improving my own prompting skills rather than defaulting to blaming the model for everything that goes sideways. A better prompt might have avoided some of the issues I ran into, particularly the input validation one where I asked for it and still didn't get it right on the first pass. That's partially on the model and partially on how I framed the requirement, and I think being honest about that split is more useful than pretending it's all one or the other.

I also want to be fair about the scope of what I tested. This was one experiment with one task, and a tic-tac-toe game with unit tests is a specific kind of challenge that hits some of the harder problems for local models (multi-file reasoning, project structure, test infrastructure) while completely skipping others. The issues I ran into might not show up the same way if I asked it to write a web scraper, build a CLI utility, do some data processing, or interact with an API. Before I draw any firm conclusions about what local models can and can't do through Aider, I need to try a wider variety of use cases and see whether the pattern holds or whether tic-tac-toe just happened to expose a particular set of weaknesses.

Would I use this setup for real work today? For quick, self-contained scripts and utilities where I don't want to burn cloud tokens and the task fits in a single file, absolutely. For anything involving multiple files, test infrastructure, or project structure decisions where the AI needs to reason about how pieces connect, I'd still reach for a more capable tool. The local setup is a good complement to cloud-based AI coding rather than a replacement for it, at least not yet and not at 14B parameters. But the fact that it works at all on consumer hardware, that I can sit down with a GPU I bought for gaming and have a coding assistant that produces functional Python without an internet connection, is worth paying attention to. This space is moving fast, and what's "good enough for simple tasks" today might be "good enough for most tasks" in another generation or two of models.

All the code from this experiment is in the [ai-station repo](https://github.com/mrmcmuffinz/ai-station/tree/main/aider) if you want to try it yourself.

Well, that's all. See you in the next one.

**P.S.** I used Claude AI to help me refine the prompt that I used in the actual experiment. The ideas, the experience, and the final edits are my own.
