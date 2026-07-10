---
title: "I think I was part of a model distillation attack"
date: 2026-07-10T10:39:05-06:00
summary: "Someone drained a legacy OpenAI API key I made back in 2023 and used it to run their own AI agent training pipeline. Here is what I found digging through the logs."
tags: ["LLMs", "Security"]
author: "Sebastian Marines"
categories: [" "]
---


Last night I received an email from OpenAI saying that they detected inconsistent usage on one of my API keys. It was late and I was already preparing to go to sleep, so I didn't pay much attention to it.

But a couple of minutes after that first email, I received a second one saying I had violated the "Prohibited Biological Use" policy and that's when I really started to worry.

![OpenAI Policy Violation Email](/i-think-i-was-part-of-a-model-distillation-attack/openai-policy-violation-email.png)

When I opened my OpenAI platform portal, I didn't see anything unusual. In fact, spend was $0, no requests showed up, so I was very confused until I opened the logs tab and saw the 114 interactions with gpt-5.5 all in quick succession, plus two stray requests from two days earlier that just said `ping` and `pong`.

I immediately went to disable the compromised API key, and found out that it was a legacy API key I had created in 2023 to use with bettergpt.chat, a UI for ChatGPT that let you bring your own key.

To be honest, I am not completely sure the leak came through that website, since they claim **"If you use your own API key, it is stored exclusively in your browser and is never shared with any third-party entity. It is used solely for the exclusive purpose of accessing the OpenAI API and not for any other unauthorized use."**.

This was a long time ago, and I don't remember using that specific key for something else. So if it looks like a duck, swims like a duck, and quacks like a duck, then it probably is a duck.

## What my API key was used for

After making sure all my API keys were disabled, I started going through the logs, and there was a bit of everything in there. Someone was writing an audio+visual QA benchmark for an "omni" video model, running coding agents that build RL and benchmark environments, doing LLM-as-judge scoring, translation and quality review, PDF→Markdown OCR of Chinese textbooks, and the one task that probably set off the Prohibited Biological Use email, exporting the chains of an insulin molecule.


Most of the coding traffic came out of a repo they call `ga-synthesis`. In each of these conversations an agent gets dropped into a task folder and told to write the scripts that set the task up, solve it, and check the result. Here is a trimmed down one for a PyMOL task, where the agent has to export the chains of an insulin molecule.

```bash
<system-reminder>
As you answer the user's questions, you can use the following context:
# currentDate
Today's date is 2026/07/10.

      IMPORTANT: this context may or may not be relevant to your tasks. You should not respond to this context unless it is highly relevant to your task.
</system-reminder>


Read the environment profile at /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/profiles/pymol_env_profile.json and the single task spec at /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator/task_spec.json. A READ-ONLY reference copy of the env's helper library is at /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator/task_utils.sh — open it ONLY to learn its function names/signatures. In the scripts you write, ALWAYS source it from its real VM location: `source /workspace/scripts/task_utils.sh` (NEVER source the local reference path, /tmp/, or /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator). Reuse its functions for all DB access (e.g. magento_query for reads AND writes: magento_query "INSERT ... ON DUPLICATE KEY UPDATE ..."). Do NOT parse env.php or build your own 'mysql -h...' command; the app/DB run in containers the host cannot reach. Write THREE files — setup_task.sh, export_result.sh, golden_solve.sh — DIRECTLY into this directory (/cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator). export_result.sh MUST write its result JSON to the EXACT path /tmp/insulin_chain_sequence_export_result.json (the verifier reads exactly that path). Do NOT write task.json (the orchestrator builds it) and do NOT modify task_utils.sh. Do NOT create a 'insulin_chain_sequence_export/' subdirectory and do NOT ask questions; produce the files now. Follow your agent prompt exactly. A previous attempt left feedback in /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator/REVIEW.md — address every FAIL line.

...

Read({"file_path": "/cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/.adv_work/pymol_env/insulin_chain_sequence_export/generator/REVIEW.md", "limit": 2000, "offset": 0, "pages": "1"})

Claude requested permissions to read from /cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/profiles/pymol_env_profile.json, but you haven't granted it yet.

0	#!/bin/bash
1	# Shared utilities for PyMOL environment tasks
2	
3	export DISPLAY=:1
4	export XAUTHORITY=/home/ga/.Xauthority
5	
6	# Take a screenshot
7	take_screenshot() {
8	    local path="${1:-/tmp/screenshot.png}"
9	    DISPLAY=:1 scrot "$path" 2>/dev/null || \
10	    DISPLAY=:1 import -window root "$path" 2>/dev/null || true
11	}
12	
...
79	# Launch PyMOL with a specific PDB file
80	launch_pymol_with_file() {
81	    local pdb_file="$1"
82	    if [ ! -f "$pdb_file" ]; then
83	        echo "ERROR: PDB file not found: ${pdb_file}" >&2
84	        return 1
85	    fi
86	    launch_pymol "$pdb_file"
87	    sleep 3
88	    maximize_pymol
89	    sleep 1
90	}
91	
```

The part that surprised me is near the bottom of that helper library. It sets up an X display, grabs screenshots with `scrot`, and has a function to launch PyMOL with a given PDB file. So this was never a text task, the agent is clicking around a real GUI app inside a virtual desktop and getting graded on whatever ends up on screen.

A few of the other system prompts show how different the jobs were. This first one runs an agent that does nothing but optimize CUDA kernels, told to work on its own and never stop.


```
You are an AI agent optimizing CUDA kernels. You have a single tool: bash. Use it for everything — reading files, writing files, running commands, git, profiling. Read TASK.md for your full instructions. Read KERNEL_GUIDE.md for optimization techniques. Work autonomously. NEVER STOP.

Context management: Older messages are automatically cleared when the window is full. This is routine — your workspace files are unaffected. Use MEMORY.md to persist important notes across clearings. When you see <context_info>, messages before it may be cleared later. Messages after it are kept. Save key state to MEMORY.md when you see it.

```

Also, a request leaked an `x-anthropic-billing-header`, a line saying the agent is built on Anthropic's Claude Agent SDK, and a path to a `ga-adv-verifier.md` agent file.

```
x-anthropic-billing-header: cc_version=2.1.193.6f1; cc_entrypoint=sdk-cli;
You are a Claude agent, built on Anthropic's Claude Agent SDK.
/cpfs01/data/shared/Group-m6/jiangyizhen.jyz/repos/ga-synthesis/synthesis/agents/ga-adv-verifier.md
```

The single biggest chunk of traffic was not code at all. Around 74 of the conversations were writing questions for a video understanding benchmark. The model never sees the video itself, it gets a text timeline of everything that happens and has to turn that into quiz questions. Here is the start of that system prompt.

```
You are an expert audio-visual question writer building a benchmark-style QA dataset for an omni (audio + vision) video model. The target model watches AND listens to the full video to answer.

You do NOT see the video. Instead you are given a **deterministic event chain** extracted from it — a text timeline of what happens.
```

Every one of those conversations also shipped with a tool called `lookup_source`, and its whole description was written in Chinese. It tells the model to go back and check every piece of evidence against the original annotation before it writes anything, so it does not make things up.

```
回查视频某处的原始标注以核实事实、防止幻觉。你在出题时依赖的**每一个证据、每一跳推理**都必须先用本工具读到原始内容再使用，不得只凭事件链摘要下结论。三选一传参：event_id（用事件链里的 eN 取该事件在原始标注中的完整内容）；entity_id（取某人物/物体/地点的注册信息与出现原文）；time + 可选 source（取某时间窗内某信息源的原始内容——source∈{vision=视觉描述, audio_synthesis=音频综合, asr=逐字语音, music=音乐, sound=声音事件}；省略 source 则返回该窗口逐字 ASR + 各通道摘要概览）。
```

The video they were annotating in most of these was a Cantonese documentary (according to Claude since I asked it to translate the text), which is part of why I think this is a Chinese lab.

## Interesting things from the chat interactions

A few things stood out while I was reading through the logs.

**They are running Claude Code against GPT.** A lot of the system prompts say things like "You are a Claude agent, built on Anthropic's Claude Agent SDK", the tool outputs kept saying "Claude requested permissions to read from ...", and one request even leaked an `x-anthropic-billing-header` with `cc_version=2.1.193` and `cc_entrypoint=sdk-cli`. So the harness itself is Anthropic's Claude Agent SDK (aka Claude Code), but the model that actually ran on my key was gpt-5.5. It looks like they built one agent setup and just point it at whatever model they feel like, and this time that was my stolen OpenAI key.

**They were not generating text, but building a computer-use benchmark.** The coding tasks all live under a repo called `ga-synthesis`, and each task launches a real desktop app inside a sandbox. You can see the X server (`DISPLAY=:1`), `scrot` taking screenshots, and helpers like `launch_pymol_with_file`. I counted around 11 different environments: PyMOL, PyCharm, QBlade, OpenVSP, QGroundControl (drones), OpenRocket, OpenEMR, OpenICE (infusion pumps), MedinTux, OnlyOffice and ProjectLibre. For each one an agent writes a `setup_task.sh`, a `golden_solve.sh` and a verifier, plus a `vlm_checklist.json` where a vision model checks a screenshot of the app against a list of expected changes. Basically a factory for teaching and grading agents that click around real software.

**It is probably a Chinese lab running on Alibaba Cloud.** The `/cpfs01/...` paths are Alibaba Cloud's Cloud Parallel File Storage, `jiangyizhen.jyz` looks like an internal Alibaba style account, a good chunk of the tool descriptions were written in Chinese, and the video they were annotating was in Cantonese.

**They tested the key before using it.** The very first requests were two days earlier, on July 8. One sent `ping` and got `pong` back, the other sent `pong` and got `ping`. Then nothing for about 33 hours, until the real batch showed up at midnight. My guess is the key got scooped into some pool, someone confirmed it was live, and it sat in a queue until they had work to run.

The funniest part is that the "Prohibited Biological Use" email that freaked me out at 1am was probably triggered by an agent opening insulin and enzyme structures in PyMOL, a molecule viewer, for one of these benchmark tasks. Not exactly a bioweapon.

## The full logs

Claude helped me pull all the conversations back out of the API, organize them, and upload the whole set to a GitHub gist. If you want to read through the raw logs yourself, they are all there.

https://gist.github.com/sebastianmarines/2e6d06ff0a58af5577ad3560d234d563

## Learnings

The attack only lasted around 10 minutes before I disabled the API key, and in that time the attackers used 4M tokens, costing $23. This is the first time I'm glad I have trouble sleeping. It happened around midnight, and I can only imagine how much I would have owed OpenAI if I hadn't realized what was going on until the next morning.

![OpenAI Bill](/i-think-i-was-part-of-a-model-distillation-attack/openai-bill.png)

I guess from now on I'm setting spending limits and deleting keys I stopped using years ago.
