# How I Use LLMs: Working Notes & Recommendations

A short field manual for working with coding LLMs (Claude Code in particular). These are personal rules of thumb, written down.

The common thread: **control the inputs**. LLMs are sensitive to context noise: what's loaded, what fires, how stale the conversation is. Most of the wins below are about reducing surprise, not about cleverness.

---

## 0. It's all just prompting

Every "capability," "smart," command, skill, agent, tool, persona, MCP server, memory entry: under the marketing terms, it's all the same thing. **Text that gets prepended or appended to your prompt before the model sees it.** There is no hidden second engine. There is no separate reasoning path. A skill is a file with instructions. A command is a file with instructions. A persona is a file with instructions. The model reads all of it as one conversation.

Once you understand this, the rest of the doc follows almost mechanically:

- More skills = more text in the prompt = more chances for the wrong text to bias the answer.
- A command "switching modes" is just a chunk of instruction text being injected. Nothing magical.
- Context limits matter because everything (skills, prompts, your messages, and every byte the model reads or runs) shares one window.
- "The model decided to use X" really means "the loaded instructions made X the most likely continuation."
- Switching topics mid-chat doesn't reset the prompt; the earlier topic is still in front of the model, shading every next answer.

The corollary: if you treat prompting as the actual primitive, you stop asking "which skill should I install?" and start asking "what text do I want in front of the model right now, and what text do I want to keep out?" Every rule below is an answer to that second question.

And one level deeper: an LLM is a stochastic, statistical machine that puts one word after another based on patterns in its training data. There is no understanding underneath, just very good guessing about what token comes next.

Practically: `~/CLAUDE.md` is where you put the text you want in front of the model in *every* chat: ground rules, persona, tone, formatting preferences, anything you'd otherwise retype. Project-level `CLAUDE.md` is for project facts; user-level is for *you*. Both are just the prompt by another name.

## 1. Keep your own skills and commands

I maintain a personal set of slash-commands outside of any project repo. The repo's skills are fine for the team; mine are for me.

**Why:** I want to *know* what's loaded. Skills fire on prompts in ways that surprise me, and even when the LLM picks a skill that's a reasonable match for the problem, if I haven't read that skill's intent I don't actually know from what angle it's making recommendations. Keeping my own set means every skill that can fire is one I've written or reviewed. The model may still align a third-party skill with the task perfectly well; I just can't verify that, and "looks right" is not the same as "is right."

## 2. Disable most skills. Enable the minimum

I run with skills *off* by default and enable only what I actually use. Currently that's roughly:

- `claude-md-management`: keeps `CLAUDE.md` honest
- `skill-creator`: for building and editing the skills I do keep
- Plus a small set of my own

Everything else is off.

**Why:** when too many skills are installed, you lose the ability to predict what the LLM will do. The model picks a skill based on a fuzzy match against the prompt, and the more skills you have, the more often it picks the wrong one. The output looks competent; it just isn't doing what you asked.

A concrete example: after installing Claude Desktop, some weeks later I noticed I had 20-odd extra skills installed that I'd never reviewed. They were firing on prompts across all my projects. I had no idea what most of them did. A "finance overview" skill firing on a Terraform question is funny exactly once.

My rule: if I can't name the skill, what it does, and roughly when it fires, it should not be enabled.

## 3. Prefer commands over skills

Skills fire opportunistically. Commands fire when I say so.

For overlapping concerns (code review, architecture review, doc review) I keep my own slash-commands and update them when I see a good idea in someone else's:

- `/architect`
- `/code-review`
- `/debug-problem`
- `/review-doc-logic`
- `/review-doc-arch`

**Why:** a skill can trigger on almost any prompt and rewrite the working context underneath me. That's useful when I want it; it's a hazard when I don't. A command is an explicit handshake: I'm saying "switch into this mode now," and the context shift is deliberate rather than ambient.

Think of skills as ambient automation and commands as deliberate mode switches. For anything that materially changes how the model reasons, I want the mode switch.

## 4. Mind the context budget

**Try to not let the chat go past ~30% of the 1M-token context window.**

What actually fills the window is not the static furniture (skills, system prompts, `CLAUDE.md`, memory). That's a fixed tax of a few thousand tokens taken once at the start, and then it doesn't move. The dominant cost is the model's own investigation: every file it reads, every command it runs, every build log returned, every subagent report. A single failing test or a verbose `bazel build` can drop tens of thousands of tokens in one tool result. The messier the codebase or the noisier the tooling, the faster you arrive at the limit.

In my experience, past that point, the model starts hallucinating noticeably more often, contradicts itself, forgets earlier decisions, and the quality of reasoning drops faster than the token count would suggest. This is the published **"context rot"** effect: every model tested (Claude included) degrades continuously as input length grows.
The damage isn't only about *finding* information; reasoning *over* longer context gets harder even when retrieval is perfect. Multi-hop work (the kind real coding tasks actually require) collapses much faster than the single-needle retrieval benchmarks suggest.

The shape is a slope, not a cliff; degradation starts almost immediately and accelerates with length. 30% is a working heuristic for coding sessions, not a hard threshold; the right number depends on task complexity and how clean the context is.

When you're approaching the limit: don't push through. Take what you've learned in the current chat and use it to seed a new one with better, tighter context. A fresh chat with a good summary outperforms an exhausted chat with full history.

Further reading:

- [Context Rot: How Increasing Input Tokens Impacts LLM Performance (Chroma, 2025)](https://research.trychroma.com/context-rot): 18 frontier models tested; every one degrades at every length increment.
- [Context Length Alone Hurts LLM Performance Despite Perfect Retrieval (arXiv 2510.05381)](https://arxiv.org/html/2510.05381v1): 13.9–85% performance drops as input grows, even when retrieval is perfect.
- [Context rot: the emerging challenge (UnderstandingAI)](https://www.understandingai.org/p/context-rot-the-emerging-challenge): accessible writeup with Anthropic's own MRCR numbers.

## 5. Keep memory OFF

I actively try to keep Claude's auto-memory *off*. Turning it off is infuriatingly harder than it has any right to be: different surfaces (Code, Desktop, web, project-level settings) each have their own toggles, defaults can flip back on after updates, and "off" in one place doesn't mean off elsewhere.

**Why:** auto-memory sounds great until it quietly remembers something wrong and acts on it later. The failure mode isn't a loud error; it's a confident answer based on a fact that was true once and isn't any more. You won't catch it unless you're already suspicious.

If you do leave memory on, expect the LLM to have opinions based on what may have been said in a short chat three days ago.

## 6. Don't change the problem space mid-chat

If you've been deep in a Terraform issue for an hour and now you want to talk about a Rust service, start a new chat. The accumulated context from the first problem will silently bias answers about the second.

**Why:** the model predicts each token conditioned on *everything* already in the conversation. An hour of Terraform vocabulary, error patterns, and reasoning style is now part of the input; it shades how the model interprets your Rust question: which patterns feel salient, which solutions get reached for first. It doesn't get confused in the obvious way (writing HCL in a Rust file); it gets confused in the subtle way: reaching for infra-shaped abstractions when the right answer is a plain function. The bias is invisible in the output.

New problem, new chat.
