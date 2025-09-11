---
title: Two Weeks, One App - My Experience with Vibe Coding
date: 2024-09-10
categories: CS
tags:
- AI
- Claude Code
- Cursor
- Codex
- ChatGPT
---
I spent the last two weeks experimenting with AI coding agents — Cursor, Claude Code, and Codex.

I’m the type of person who comes up with a new idea every day and often gets the urge to build it right away. I love creating things, but I don’t enjoy grinding through the low-level details of coding. In the AI era, that’s no longer a blocker — tools like these are a dream for builders like me.

Here’s what I learned about efficiency, design, prompt engineering, agents, and even my own mindset after building an app in just two weeks.

### Efficiency Gains
- Writing new code: A true 10x speedup. Seven years ago, I last built an iOS app — and I had forgotten everything. This time, with AI, the core features were running in just a few days.
- Debugging: Weaker. Probably because that context windows are limited, hallucinations are common, and agents rarely ask for help directly. With the right instructions, it gets better, but honestly — sometimes I debug faster myself.
- Testing: I let AI handle all my unit tests (finally!). I still had to think up edge cases and instruct it carefully, but my hands were freed from the part of coding I find the most time consuming.
- Learning: By reading AI-generated code, I re-learned Swift’s syntax and design patterns. Reading code + asking targeted questions beat any course, video, or manual.

### Design Workflows
Two approaches worked well for me:
- Rough wireframes in Figma, then implementation with Claude Code + Figma Dev MCP.
- Stitch together Dribbble designs into a screen and hand them to Claude Code.

The results matched my vision 100%. But aesthetics still matter — you need at least a sense of layout, fonts, and colors. Otherwise, the UI ends up with that faint “AI flavor.”

### Claude Code and Prompts
I optimized my workflow mostly around Claude Code. At first, I gave it no rules — just asked questions and got answers. For example, I asked it to fix a compile error. It kept spinning, trying five different ways to guess what the logs said. I wondered: why can’t it just see the error I can see? Eventually, I politely asked, “Do you need me to provide something to help you locate the error log?” Then it admitted: “Your Xcode version is too low, I can’t build. Could you upgrade?” 🤦 This is very non-human behavior — rather than reaching out, it blindly tries random things.

So, I built some meta-rules for our collaboration. Before every task, I remind it:

- Collaborate, don’t solo. “If you can’t solve something, reach out and ask what resources I can provide.”
- Don’t rush into coding. “Do more problem exploration first. Analysis > coding. Think step by step and show your reasoning. Confirm your plan with me before writing code. Don’t modify the codebase too fast — every line should be thoughtful and useful.” **This saved me tons of time and is where humans shine: defining problems, setting processes, and choosing solutions.**
- Manage confidence. Claude always says “This time it’s fixed!” — and then it isn’t. That hurt my soul 😅. So I told it: “For every attempt, give me a confidence level.” Now I get fewer false hopes.
- Performance check. By default, its code wasn’t performance-optimized. I opened a separate session to specifically ask about performance issues and improvements. Iterating here was also crucial.

### Comparing Agents
I rotated between Cursor, Claude Code, and Codex:
- Tone: Codex = cold and blunt. Claude = people-pleaser. Cursor = balanced, often explains.
- Ease of use: All three can integrate into IDEs. I sometimes run Claude Code directly from the terminal.
- Use cases:
> Cursor → quick fixes, code explanations
> Codex → sharper debugging (esp. backend/non-UI), but when editing code, it sometimes randomly added a “{” or forgot a “},” which annoyed me.
> Claude → my go-to for writing new code and unit tests.
> Best part: Cross-checking. When one agent failed, I switched. Stubborn bugs eventually gave in.

### Cognitive Shifts
Using coding agents has greatly changed my mindset.

Before, I thought being a “code god” meant writing code super fast, and I held myself to that standard. But now AI can do that. So where does that leave me?

I realized the freed-up time lets me focus on higher-value questions: product positioning, iteration, identifying which problems are truly worth solving, and where engineering direction matters most. The “screwdriver work” (like debugging a complex chart display in my app) can just be handed to AI — it would’ve taken me days otherwise, and I don’t even want to learn it deeply.

AI liberates me from detail traps. That’s a huge relief. And honestly, I enjoy the moments when I give it a task, then go shower or sleep. It feels great to see technology directly improving my quality of life, turning labor hours into better rest.

I also asked some ex-colleagues — turns out nobody really writes code from scratch anymore. Everyone starts with AI scaffolding. For me too, it now feels strange to open a blank file and manually type import or require.

### Final Thoughts
This whole two-week experiment was fascinating and super fun. AI tools didn’t just speed things up — they reshaped how I think about building, coding, and what’s worth my attention.