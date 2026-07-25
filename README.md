# pi-cover-image

A [pi](https://pi.dev) package that creates a cover image for an article on a target **surface** (X Article, 微信公众号头条/图片消息, 小红书, or a generic ratio).

## Pipeline

idea-extractor (reads the article in a clean context; develops 3 idea+rhetoric candidates; scores them on the six poster tests, with freshness judged against current AI-image tropes; outputs the best + the article's real title verbatim) → title-link-reflector (gates track selection with the title-swap test — would this image fit a different title just as well? — and sends all candidates back when none links to the title) → [parallel: palette-planner (recommends palette artworks) + layout-planner (recommends films — layout sources are film posters from a film poster database ONLY, never artworks; the planner also typesets the title)] → works/poster lookup → download → feature extraction → prompt-composer (candidates from TWO independent art-direction tracks: 2 from the chosen rhetoric + 1 from the runner-up; each rhetoric owns its palette, layout, seed artworks, and extracted features, so one weak rhetoric cannot poison the set) → **generation via [`pi-image-gen`](../pi-image-gen)** (Codex / Antigravity / Grok) → [parallel: blind slop jury — one juror per candidate, image-only, advisory verdicts; findings feed a local gitignored slop memory that future idea extractions read as banned tropes] → final-checks (light visual self-check).

Every stage's output is a file in the run dir, so a run can **resume from any stage** — useful both for debugging which stage's output went wrong and for recovering from a slop verdict (resume from the idea stage with the rejected tropes as exclusion constraints).

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
agents/                    10 subagents (idea-extractor, title-link-reflector, palette-planner, layout-planner, artist-works, artwork-feature-extractor, layout-generator, prompt-composer, final-checks-verifier, slop-juror)
scripts/seed-library/      fetch.py, palette.py, recall.py, manifest.json, setup.zsh
package.json               pi.skills + pi.subagents.agents; bundles pi-image-gen; declares pi-subagents as the subagent runtime
```