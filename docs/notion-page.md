# Made in code: make a film or a game with AI, no AI art

A free playbook for Claude Code and Codex. Copy one prompt, paste it in, and the agent builds an animated film or a 3D browser game where every picture and sound is drawn in code.

## What people have made with it

- **Ashen Throne**, a Souls-like boss fight you can play in your browser: https://ashen-throne-game.vercel.app . The knight, the boss, the castle and the music are all code, with no image or audio files. It took about 5.5 hours.
- **A 23-minute animated explainer** in a hand-cut linocut storybook style, with 27 scenes, a cast of 24 characters and full narration. It took about 14 hours.

## What you need

- Claude Code (https://claude.com/claude-code) or Codex, on a paid plan. These are big builds, so a subscription works out much cheaper than paying per API call.
- About an hour of your attention for a small version. The agent does the rest.
- For narrated films only: a text-to-speech voice. The agent offers a free local option first.

## The prompt

Copy everything in the box and paste it into Claude Code or Codex.

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

## What happens next

1. The agent installs what it needs and asks you what you want to make.
2. It writes a one-page direction for the look and feel, then waits for your OK.
3. It builds a small working version and shows it to you.
4. It builds out the rest, checking its own work as it goes.

## Source

Everything is open and MIT licensed: https://github.com/Jeremy-Sharpe/made-in-code
