# pi-cover-image

A [pi](https://pi.dev) package that creates a cover image for an article on a target **surface** (X Article, 微信公众号头条/图片消息, 小红书, or a generic ratio).

## Pipeline

idea-extractor (reads the article in a clean context; develops 3 idea+rhetoric candidates; scores them on the five poster tests; outputs the best + the article's real title verbatim) → [parallel: palette-planner + layout-planner (the planner also typesets the title)] → artist-works → download → feature extraction → prompt-composer (3 layout variants) → **generation via [`pi-image-gen`](../pi-image-gen)** (Codex / Antigravity / Grok) → final-checks (light visual self-check).

Generation is delegated to the `image-gen` skill from `pi-image-gen`, which this package bundles (`bundledDependencies`). No image is sent to the generator — only text features.

Subagents run on [`pi-subagents`](https://github.com/nicobailon/pi-subagents) (the `subagent` tool family), also wired in as a bundled dependency — pi loads its extension from `node_modules/pi-subagents/`, and the runtime discovers this package's agents through the `pi.subagents.agents` manifest key (no host-side setup). If the host already installs `pi-subagents` itself (e.g. globally), keep exactly one copy loaded by excluding the bundled one through the host's package filter:

```json
{
  "source": "git:github.com/WeZZard/pi-cover-image",
  "extensions": ["-node_modules/pi-subagents/index.ts"]
}
```

## Install

```bash
pi install git:github.com/WeZZard/pi-cover-image
```

`pi-image-gen` and `pi-subagents` are pulled in automatically as bundled dependencies.

## Use

The skill takes `surface` + `article-path` + `output-dir` and writes the cover PNG (the surface's `filename`) to `output-dir`. It does not touch the article file or any post directory — a host project wires those (see the blog repo's `cover-image-ship` shim).

## Layout

```
skills/cover-image/        SKILL.md + references/ (surfaces.md, surfaces/<id>.md, poster-principles.md, visual-rhetoric.md, titling.md, final-checks.md)
agents/                    8 subagents (idea-extractor, palette-planner, layout-planner, artist-works, artwork-feature-extractor, layout-generator, prompt-composer, final-checks-verifier)
scripts/seed-library/      fetch.py, palette.py, recall.py, manifest.json, setup.zsh
package.json               pi.extensions (bundled pi-subagents) + pi.skills + pi.subagents.agents; bundles pi-image-gen and pi-subagents
```