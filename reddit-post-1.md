TL;DR: Claude-auto-pause is a free, open-source (MIT) Claude Code plugin. When you're down to 8% of your 5-hour limit or 3% of your weekly limit, Claude stops at a clean point, saves a note (handoff) about where it left off, and gives you a "resume" prompt to paste into a new chat after the reset.
https://github.com/k45sle/Claude-auto-pause


> **This plugin was written entirely by AI.** Claude wrote and tested all of the plugin's code, and helped me edit this post. I'm not a programmer. The idea was mine (before I knew similar plugins existed), and I did the research on other plugins after I had the idea, decided what it should do, and reviewed the work.


The usage limit has no sense of timing. It can land in the middle of a change and leave your files half-edited, with no record of what Claude was doing. Then you come back after the reset, unknowingly burn your fresh tokens, and play detective.

With auto-pause:

- At 15% left of the 5-hour limit, Claude is quietly told to wrap up and not start anything new.
- At 8% left, it stops, saves the note in a `.handoffs` folder, and tells you when your limit resets.
- After the reset, you open a new chat, paste the resume prompt, and it carries on.

The weekly limit gets the same treatment at 10% and 3%. Under the hood it's a skill that tells Claude how to write the note, plus a hook that watches your usage.

I looked at nine similar plugins (they're all in a comparison table in the README). Plenty of them warn you or write a note. My Claude-auto-pause is the only one I found that locks Claude's tools until the note passes a check: it has to match the current state of your project, list every file Claude changed, and contain no passwords or keys. So Claude can't write "did some stuff, good luck" and clock out.

Why a new chat instead of the old one? Claude's cache of your chat expires after an hour, so continuing the old chat after the reset makes it re-read the whole conversation at full price. A short note costs much less. It's plain text, so you can also start the new chat on a cheaper model.


The catch:

- Writing the note eats a small part of your last usage allowance.
- I haven't measured how much usage it saves yet.
- You need a Pro or Max plan. It works in the Desktop app's Code tab, the terminal and VS Code.
- It's version 0.1.5 and still being tested.


Install:

```
claude plugin marketplace add k45sle/Claude-auto-pause
claude plugin install auto-pause@k45sle --scope user
```

Then restart Claude and see the README for one setting to change. Bug reports and roasting welcome, here or in GitHub issues.


P.S. While I was drafting this post, the plugin warned Claude that I was down to 7% of my weekly limit, so it works on me at least.
