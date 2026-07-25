---
name: cover-image
description: Create a cover image for a target surface (X Article, 微信公众号头条/图片消息, 小红书, or a generic ratio). An idea-extractor reads the article and outputs idea + rhetoric + orientation; palette-planner and layout-planner develop palette and layout directions in parallel and recommend artists; artist-works downloads works; features are extracted and combined into a pure-text generation prompt — no image is sent to the generator.
---

# Cover Image

Create a cover image for a target **surface**. The flow splits into:
- **Concept** (from the article): idea → rhetoric → orientation — all in one clean context.
- **Palette** (parallel): palette-planner develops a palette direction + recommends palette artists.
- **Layout** (parallel): layout-planner develops a layout direction + recommends layout artists.
- **Download**: artist-works finds and downloads works for all recommended artists.
- **Extract + Generate**: palette features from palette works + layout features from layout works → pure-text generation prompt (no image sent).

References under `references/`:
- `surfaces.md` — the **surface registry**: one row per surface with `aspectRatio`, `safeArea`, `bleed`, `text`, `cropBehavior`, `filename`, and a `detail` path. Read this first to offer the surface choice and to load each surface's config.
- `surfaces/<id>.md` — per-surface detail: composition rules, crop behavior, element list (the `detail` each registry row points at).
- `poster-principles.md` — the five tests a cover must pass; read by `idea-extractor` before it extracts anything.
- `visual-rhetoric.md` — catalog of visual rhetorical devices (read by idea-extractor).
- `titling.md` — how the article's single title is typeset on a banner, including when to split into kicker + main (read by layout-planner on text-carrying surfaces).
- `final-checks.md` — criteria for the visual subagents.

Subagents (spawned via the `subagent` tool from `pi-subagents`, wired in as a bundled dependency — see **Subagent runtime** below):
- `idea-extractor` — reads the article in a clean context; extracts idea (3 words) + creates rhetoric (1 device + target) + orientation (color/tone).
- `palette-planner` — given idea + rhetoric + orientation, develops a palette direction + recommends 2–3 palette artists.
- `layout-planner` — given idea + rhetoric + orientation + surface constraint, develops a layout direction + recommends 2–3 layout artists.
- `artist-works` — agentically searches the web for all recommended artists' works; records work + reason + source URL.
- `artwork-feature-extractor` — views downloaded works and extracts visual key features as text (one per image).
- `layout-generator` — views downloaded works + surface constraint, extracts spatial tendencies, writes layout rules.
- `prompt-composer` — takes all intermediate products (rhetoric, palette direction + features, layout direction + features, surface constraints) and composes the final generation prompt as markdown text. Kimi K3.
- `slop-juror` — blind-judges one generated candidate against a single criterion (does it read as AI slop) with no knowledge of the pipeline; one juror per candidate. Kimi K3.

Tools: `../../scripts/seed-library/` — `fetch.py` (download with Wikimedia verification + GAP fallback), `palette.py` (exact palette), `recall.py` (gallery + provenance). Generation backends are provided by the `image-gen` skill (Codex / Antigravity / Grok).

## Subagent runtime

The pipeline's subagents run on **`pi-subagents`** (the `subagent` tool family). The package declares it in `dependencies` but does **not** load the extension itself — pi fails hard when two extensions register the same tool, so the single running copy must be the host's own install (global or project): `pi install npm:pi-subagents`. Any running `pi-subagents` discovers this package's agents through the `pi.subagents.agents` manifest key; there is no other host-side setup. If the `subagent` tool or the 9 agents are missing, that install is what is absent.

## Slop memory

The pipeline keeps a **local, per-host memory of blind-jury slop findings** at `<host-project-root>/.pi/cover-image/slop-memory.jsonl` — one JSON object per line:

```json
{"ts": "<UTC ISO>", "article": "<article-path>", "run": "<run-dir>", "candidate": "<candidate file>", "verdict": "yes|borderline", "tropes": ["<noun phrase>", ...], "justification": "<two sentences>"}
```

- **Write:** after the jury step, append one line per candidate whose verdict is `yes` or `borderline`. Never append `no` verdicts.
- **Read:** pass the memory file to `idea-extractor` (step 2) whenever it exists — recorded tropes are banned rhetoric targets. Never pass it to a juror: jurors stay blind.
- **Ignore:** the memory is local learning, not source. It MUST be gitignored in the host project (add `.pi/cover-image/` to the host's `.gitignore`); never commit it.

## Run modes

Resolve the execution mode **before** creating or selecting a run dir. The mode is part of the run contract, recorded in `meta.json`; do not infer it from the presence of an old run dir.

### 1. New task — default

When the caller does not explicitly name a mode, this is a **new task**. Create and use a fresh run dir under the normal timestamped naming rule. Never read, copy, or alter an earlier run's artifacts.

### 2. Rerun

When the caller explicitly asks to run an earlier task again, this is a **rerun**. Create and use a fresh timestamped run dir, exactly as for a new task. Set `"mode": "rerun"` and `"source-run": "<prior run dir>"` in the new run's `meta.json`; retain the article path, surface, output directory, and any user-specified title override from the source run, but execute every stage again. The source run is immutable provenance — never write into it.

### 3. Checkpoint resume

When the caller explicitly asks to continue an earlier task **from stage N**, this is a **checkpoint resume**. This is the only mode allowed to use the original run dir. Before re-executing, verify the artifacts from stages `1..N-1`, record `"mode": "checkpoint-resume"`, `"resume-from-stage": N`, and a UTC timestamp in that run's `meta.json`, then re-execute stage N and every following stage in the same dir.

If any outputs at or after stage N already exist, copy them to `revisions/<UTC>/` inside the same run dir before overwriting them. The checkpoint remains the active run, while the snapshot preserves the discarded downstream attempt for diagnosis.

Restart far enough back to cover the stage whose output is suspect: suspect rhetoric → stage 2; suspect palette/layout → stages 3/4; suspect prompt → stage 9; suspect generation only → re-run `image-gen` on the existing `08-generation-prompts/`. When a checkpoint resume follows a slop verdict, append the rejected tropes to the spawning prompt of the resumed stage as exclusion constraints (they are already in the slop memory).

## Concurrency

The runtime gives three execution shapes — pick per step by the step's shape, not by habit:

- **Parallel (one call):** a single `subagent({ tasks: [ ... ] })` call runs its tasks concurrently and returns all results together (add `async: true` to background the whole group). Use this when a step fans out independent sub-tasks whose results are all needed together.
- **Background (async):** `async: true` on any run returns a run id immediately and wakes the session on completion; several async calls may be issued in one message. Use this to overlap independent work with other work.
- **Blocking (foreground):** a foreground `subagent` call blocks the turn and returns the result inline. Hard limit: **exactly one foreground execution call per message** — the runtime rejects a second one. Never issue two blocking calls in one message; use the parallel form or async instead.

When a step below is marked **[parallel]**, its sub-tasks are independent and SHOULD run concurrently (prefer the one-call parallel form); when marked **[sequential]**, each sub-task consumes the previous one's output and must not be parallelized. Steps without a mark are single-spawn steps where either blocking or background is correct.

## Working directory

Use a fresh temp run dir per run (e.g. `$(mktemp -d -t cover-image-XXXX)`). Write every step's output there before the next step runs — nothing is held only in conversation:

```
<temp-run-dir>/
  meta.json                     # article-path, surface, output-dir, timestamp
  01-idea-extractor.json        # idea (3 words) + rhetoric (device + target) + orientation + article_title
  02-palette-planner-<rhetoric-id>.json # one palette direction + artists per selected rhetoric
  03-layout-planner-<rhetoric-id>.json  # one layout direction + title block + artists per selected rhetoric
  04-artist-works.json          # all works, grouped by selected rhetoric and palette/layout role
  05-downloads/                 # downloaded artworks (all)
  06-palette-features-<rhetoric-id>.json # palette features for one rhetoric's palette artists
  07-layout-features-<rhetoric-id>.json  # layout features for one rhetoric's layout artists
  08-generation-prompts/        # one .md per candidate, written before generation
  09-candidates/                # generated candidate PNGs (2-3)
  10-jury/                      # one blind-jury verdict .md per candidate
  11-provenance.json            # per candidate: palette source + layout source + rhetoric + features + jury verdict
  12-final.png                  # the user's pick
```

The final image (`12-final.png`) and `11-provenance.json` are copied to `output-dir` with the surface's `filename` at the end. The temp run dir may be kept for provenance or discarded.

## Input

- `surface` — a surface `id` from `references/surfaces.md` (e.g. `x-article`, `wechat-headline`, `wechat-image-message`, `xiaohongshu`, `square`, `4x3`, `9x16`, `16x9`).
- `article-path` — absolute path to the article file (the text to make a cover for).
- `output-dir` — absolute path of the directory where the final cover PNG is written.

If the surface is missing, ask once which surface, listing the `label` column from the registry; cache the answer in the session (it stays in the transcript). If `article-path` or `output-dir` is missing, ask once for each. **Default to New task only when no execution mode is stated; never infer rerun or checkpoint resume from existing files.**

## Workflow

1. Resolve `surface`, `article-path`, `output-dir`, **and the execution mode** (see **Run modes**). Read `references/surfaces.md`, look up the surface row, and read its `detail` file (`references/surfaces/<id>.md`). Build the **surface constraint** from the row: `aspectRatio` + `safeArea` + `bleed` + `text` + `cropBehavior` + `filename`.

   - **New task / rerun:** create the fresh timestamped run dir with subdirs `05-downloads/`, `08-generation-prompts/`, `09-candidates/`, `10-jury/`; write `meta.json` with the mode. A rerun also records its source run.
   - **Checkpoint resume:** select the named existing run dir; snapshot every existing output from the restart stage onward into `revisions/<UTC>/`, update its `meta.json`, and do not create a sibling run dir.

   Continue with the selected run dir.
2. **Idea + rhetoric + orientation + article title.** Spawn `idea-extractor` with the article path AND `references/poster-principles.md` AND `references/visual-rhetoric.md` AND the slop memory file when it exists (see **Slop memory**). It reviews the poster principles, surveys the article, develops THREE idea+rhetoric candidates, scores them on the five poster tests, and outputs the highest-scoring one (with per-test breakdown and the two runners-up). It also carries the article's frontmatter `title` **verbatim** as `article_title` — it does NOT split or rephrase it; splitting is a layout decision. **Write** the full output to `01-idea-extractor.json`. Select two rhetoric tracks for this run: `chosen` and `runner-up-1`. Each track owns its own palette, layout, seed artworks, features, and generation prompt.
3. **[parallel] Palette directions.** For EACH selected rhetoric track (`chosen`, `runner-up-1`), spawn `palette-planner` with that track's idea, rhetoric, and orientation. **Write** one result per track to `02-palette-planner-<rhetoric-id>.json`.
4. **[parallel] Layout directions.** For EACH selected rhetoric track, spawn `layout-planner` with that track's idea, rhetoric, orientation, surface constraint, article title, and `references/titling.md`. **Write** one result per track to `03-layout-planner-<rhetoric-id>.json`. All four planners are independent — run them concurrently.
5. **Find works.** Spawn `artist-works` with every artist from both palette/layout planners, tagged with its rhetoric track, plus that track's rhetoric target. **Write** to `04-artist-works.json`.
6. **Download works.** For each found work, download with `fetch.py` into `05-downloads/`, retaining its rhetoric-track tag.
7. **[parallel] Extract palette features.** Spawn one `artwork-feature-extractor` per downloaded palette work, across BOTH tracks, all concurrently. **Write** one grouped result per track to `06-palette-features-<rhetoric-id>.json`.
8. **[parallel] Extract layout features.** Spawn one `artwork-feature-extractor` per downloaded layout work, across BOTH tracks, all concurrently. **Write** one grouped result per track to `07-layout-features-<rhetoric-id>.json`. Steps 7 and 8 may overlap; the runtime's queue handles overflow.
9. **Generate.** Each rhetoric track is independently art-directed. Spawn `prompt-composer` once per track, passing ONLY that track's rhetoric, palette direction/features, layout direction/features, and the shared surface constraint; write its candidate prompt(s) before generation. Generate two candidates from the `chosen` track and one from `runner-up-1`, each from the matching prompt, to `09-candidates/`. This prevents one weak rhetoric from poisoning the whole set and prevents a runner-up's content from being forced into the chosen rhetoric's layout.
10. **[parallel] Blind jury.** Spawn one `slop-juror` per candidate — all concurrently, each in a fresh context, each given ONLY the candidate's image path and the criterion (no article, no rhetoric, no pipeline context: jurors stay blind). **Write** each verdict verbatim to `10-jury/candidate-<n>.md`, then append every `yes`/`borderline` finding to the slop memory (see **Slop memory**). The jury is **advisory**: verdicts are presented with the candidates; the user rules.
11. **Record provenance.** Write `11-provenance.json` mapping each candidate to its rhetoric (chosen / runner-up), palette source + layout source + features, the backend that generated it (with any ratio deviation), and its jury verdict.
12. **Ship + check.** Present the candidates WITH their jury verdicts; copy the user's pick to `12-final.png`, then to `output-dir` with the surface's `filename` (and copy `11-provenance.json` alongside if useful). Run `final-checks-verifier` (passing `article_title` as ground truth for the on-image text) for a light visual self-check. If the user rejects every candidate, choose the explicit next mode: a new **rerun** for a fresh attempt, or a **checkpoint resume** from stage 2 for slop-driven rhetoric repair / stage 9 for a taste-driven generation repair. The rigorous SAM-based crop-safety check is the host project's job (e.g. its preflight) — this skill does not ship a SAM stage. The host project handles any post-placement (e.g. copying into a post directory) and its own preflight — this skill stops at writing the cover to `output-dir`.

## Stacking rules

**MUST:**

- Let `idea-extractor` read the article in a clean context; it creates idea + rhetoric + orientation — do not hardcode any of them.
- Select the chosen rhetoric and first runner-up as separate tracks; run the palette/layout planners, artwork lookup, and feature extraction once **per track**. Do not use the chosen track's art direction to render the runner-up.
- Let `artist-works` find works for every artist from every track; keep each work tagged with its track and palette/layout role.
- Do NOT send any artwork image to the generator — the generator works from text only.
- Let `prompt-composer` compose each track's prompt from only that track's intermediate products — do not compose it yourself.
- Render the title **verbatim** from each track's `layout-planner` title block (`article_title` / `kicker` / `main`). Do not invent, rephrase, or re-split cover text.
- **Write each candidate's full prompt to `08-generation-prompts/candidate-<n>.md` before generating, and pass that file's content verbatim to the `image-gen` skill.**
- Generate 2–3 candidates concurrently — two from the chosen track, one from runner-up-1 — with each candidate using its matching track prompt.
- Keep jurors blind: a `slop-juror` spawn carries only the image path and the criterion — never the article, rhetoric, prompt, or other verdicts.
- Write every jury verdict to `10-jury/` and append every `yes`/`borderline` finding to the slop memory before presenting candidates.

**MUST NOT:**

- Send artwork images to the generator.
- Send a prompt to the generator that differs from what is written in `08-generation-prompts/candidate-<n>.md`, or generate before the candidate file is written.
- Invent or rephrase the cover title — it comes from `idea-extractor`'s `article_title` and is typeset by `layout-planner`; the main agent never authors or splits it.
- Mix palette and layout sources — keep them separate until the generation prompt.
- Skip provenance, the blind jury, or the final visual checks.
- Commit the slop memory to git — it is local, gitignored learning.
- Let a juror see anything but its one image and the criterion.
- Put text on the WeChat 头条 banner.

## Limits

- You **MUST NOT** edit the article file unless the user explicitly asks.
- You **MUST NOT** overwrite an existing cover asset in `output-dir` unless the user explicitly asks.