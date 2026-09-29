# Copy-paste prompt

Paste everything in the box below into Claude Code or Codex. You don't need to know how to code: the agent installs what it needs, asks you what you want to make, and builds it.

```text
I want to make something creative where every picture and sound is drawn in code: no AI image models, no stock art.

Use the free "made-in-code" playbook at https://github.com/Jeremy-Sharpe/made-in-code

1. Set it up:
   - Claude Code: clone the repo into ~/.claude/skills/made-in-code so it loads as a skill.
   - Codex or any other agent: clone it into a folder called made-in-code next to our project, then read SKILL.md.
   Read SKILL.md first, then the playbook that fits what I want: references/film.md for an animated video, references/game.md for a browser game.

2. I'm not technical, so:
   - Install any tools you need yourself (Node, ffmpeg, and so on), telling me in one line what each is for. Stop and ask before anything that costs money or needs an account or API key, and give me the cheapest or free option first.
   - Explain decisions in plain English, not code.
   - Show me my work as you go: a rendered still, a short clip, or a link I can open in my browser.

3. Before building anything, interview me in one round: what I want to make, who it's for, how long or big it should be, the look and mood I'm imagining (a film, artist or game it should feel like), and any rules I have. If my answer is thin, push back and suggest options.

4. Then follow the playbook: write the one-page direction and get my OK, build a small working version first, and only then build out the rest. Check your own work with the playbook's methods (render and look at frames, play-test with the bot), and read its traps list whenever something looks wrong.

5. Before starting a big build, tell me roughly how long it will take and what it will cost.
```
