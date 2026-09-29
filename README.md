# made-in-code

A free Claude Code skill for making ambitious creative work where every picture and sound is drawn in code: no image models, no stock art, no binary assets.

It distils two real builds:

- **A 23-minute animated explainer film** in a linocut storybook style: SVG puppets, GSAP and HyperFrames, TTS narration timed to the word, and a music and SFX mix.
- **[Ashen Throne](https://ashen-throne-game.vercel.app)**, a Souls-like 3D boss fight in the browser: procedural rigs and animation, a castle, VFX and a synthesised orchestral score in three.js, playtested by a bot.

What you get is the process, not the source code: the pipeline, drop-in snippets, the brief template for running parallel subagents, calibration numbers (time, cost, scale), and about 50 traps, each written as symptom, cause and fix.

## Install

```bash
git clone https://github.com/Jeremy-Sharpe/made-in-code ~/.claude/skills/made-in-code
```

Then ask Claude Code something like "make me an animated explainer of this essay, all drawn in code" or "build me a browser boss fight". The skill loads on its own.

For films, also install [product-launch-motion](https://github.com/AbubakrChan/product-launch-motion), which covers motion craft. This skill builds on it and does not copy it.

## Files

- `SKILL.md`: shared laws, and which playbook to use
- `references/film.md`: the animated film playbook
- `references/game.md`: the browser game playbook

## Licence

MIT. Third-party tools (GSAP, HyperFrames, three.js, TTS providers, music and SFX libraries) keep their own terms; check them before you ship anything commercial.
