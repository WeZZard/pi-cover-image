# pi-cover-image

A [pi](https://pi.dev) package that creates a cover image for an article on a target **surface** (X Article, 微信公众号头条/图片消息, 小红书, or a generic ratio).

## Pipeline

idea-extractor (reads the article in a clean context; develops 3 idea+rhetoric candidates; scores them on the five poster tests; outputs the best + the article's real title verbatim) → [parallel: palette-planner + layout-planner (the planner also typesets the title)] → artist-works → download → feature extraction → prompt-composer (3 layout variants) → **generation via [`pi-image-gen`](../pi-image-gen)** (Codex / Antigravity / Grok) → final-checks (light visual self-check).

Generation is delegated to the `image-gen` skill from `pi-image-gen`, which this package bundles (`bundledDependencies`). No image is sent to the generator — only text features.

Subagents run on [`pi-subagents`](https://github.com/nicobailon/pi-subagents) (the `subagent` tool family). The package declares it in `dependencies` (version-pinned) but deliberately does **not** load the extension itself: pi fails hard when two extensions register the same tool, so the running copy must be exactly one install owned by the host — globally or per project:

```bash
pi install npm:pi-subagents
```

Any running `pi-subagents` discovers this package's 8 agents through the `pi.subagents.agents` manifest key — no other host-side setup.

## Install

```bash
pi install git:github.com/WeZZard/pi-cover-image
```

`pi-image-gen` is pulled in automatically as a bundled dependency. Install `pi-subagents` alongside (see above).

## Use

The skill takes `surface` + `article-path` + `output-dir` and writes the cover PNG (the surface's `filename`) to `output-dir`. It does not touch the article file or any post directory — a host project wires those (see the blog repo's `cover-image-ship` shim).

## Layout

```
skills/cover-image/        SKILL.md + references/ (surfaces.md, surfaces/<id>.md, poster-principles.md, visual-rhetoric.md, titling.md, final-checks.md)
agents/                    8 subagents (idea-extractor, palette-planner, layout-planner, artist-works, artwork-feature-extractor, layout-generator, prompt-composer, final-checks-verifier)
scripts/seed-library/      fetch.py, palette.py, recall.py, manifest.json, setup.zsh
package.json               pi.skills + pi.subagents.agents; bundles pi-image-gen; declares pi-subagents as the subagent runtime
```