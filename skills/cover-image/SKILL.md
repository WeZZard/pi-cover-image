---
name: cover-image
description: Create a cover image for a target surface (X Article, 微信公众号头条/图片消息, 小红书, or a generic ratio). An idea-extractor reads the article and outputs idea + rhetoric + orientation; a title-link-reflector then gates track selection with the title-swap test (would this image fit a different title just as well?) before any art direction starts; palette-planner and layout-planner develop palette and layout directions in parallel — palette recommends artworks, layout recommends film posters from a film poster database (layout sources are film posters ONLY); works and posters are downloaded; features are extracted and combined into a pure-text generation prompt — no image is sent to the generator.
---

# Cover Image

Create a cover image for a target **surface**. The flow splits into:
- **Concept** (from the article): idea → rhetoric → orientation — all in one clean context.
- **Title-link reflection** (gate): title-link-reflector runs the title-swap test on every candidate and selects the two tracks — before any art direction spends effort. All-fail sends candidates back to the idea-extractor with notes.
- **Palette** (parallel): palette-planner develops a palette direction + recommends palette artists.
- **Layout** (parallel): layout-planner develops a layout direction + recommends films whose posters embody it. Layout references come ONLY from film posters located on a film poster database — never paintings, prints, or graphic-design posters.
- **Download**: artist-works finds palette artworks (Google Art Project / Wikimedia) and layout film posters (film poster database); both are downloaded.
- **Extract + Generate**: palette features from palette works + layout features from layout posters → pure-text generation prompt (no image sent).

References under `references/`:
- `surfaces.md` — the **surface registry**: one row per surface with `aspectRatio`, `safeArea`, `bleed`, `text`, `cropBehavior`, `filename`, and a `detail` path. Read this first to offer the surface choice and to load each surface's config.
- `surfaces/<id>.md` — per-surface detail: composition rules, crop behavior, element list (the `detail` each registry row points at).
- `poster-principles.md` — the tests a cover must pass (including title link and common visual memory); read by `idea-extractor` before it extracts anything.
- `visual-rhetoric.md` — catalog of visual rhetorical devices (read by idea-extractor).
- `titling.md` — how the article's single title is typeset on a banner, including when to split into kicker + main (read by layout-planner on text-carrying surfaces).
- `final-checks.md` — criteria for the visual subagents.

Subagents (spawned via the `subagent` tool from `pi-subagents`, wired in as a bundled dependency — see **Subagent runtime** below):
- `idea-extractor` — reads the article in a clean context; extracts idea (3 words) + creates rhetoric (1 device + target) + orientation (color/tone).
- `title-link-reflector` — judges each candidate on two reader-side axes: the title-swap test (would this image fit a different title just as well?) and the visual-memory check (recognized in one glance, or identified before it can be read?); selects the two tracks or sends all candidates back with notes. Kimi K3.
- `palette-planner` — given idea + rhetoric + orientation, develops a palette direction + recommends 2–3 palette artists.
- `layout-planner` — given idea + rhetoric + orientation + surface constraint, develops a layout direction + recommends 2–3 films whose posters embody it.
- `artist-works` — agentically searches the web: palette artists → artworks (Google Art Project / Wikimedia); layout films → actual film posters on a film poster database, with film + year + designer + database URL recorded.
- `artwork-feature-extractor` — views downloaded works and extracts visual key features as text (one per image).
- `layout-generator` — views downloaded works + surface constraint, extracts spatial tendencies, writes layout rules.
- `prompt-composer` — takes all intermediate products (rhetoric, palette direction + features, layout direction + features, surface constraints) and composes the final generation prompt as markdown text. Kimi K3.
- `slop-juror` — blind-judges one generated candidate against a single criterion (does it read as AI slop — machine tropes OR shoddy design: internal inconsistency, impossible geometry, style-clashing type) with no knowledge of the pipeline; one juror per candidate. Kimi K3.

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

When the caller explicitly asks to continue an earlier task **from stage N**, this is a **checkpoint resume**. Even here, the source run dir is immutable: **two generations never share a directory.** Create a fresh timestamped run dir, copy the verified artifacts from stages `1..N-1` out of the source dir into it, record `"mode": "checkpoint-resume"`, `"resume-from-stage": N`, `"source-run": "<prior run dir>"`, and a UTC timestamp in the NEW dir's `meta.json`, then re-execute stage N and every following stage in the new dir. The source run stays frozen as provenance — nothing is written into it, so no snapshot mechanism is needed.

Restart far enough back to cover the stage whose output is suspect: suspect rhetoric → stage 2; suspect title-link selection → stage 3; suspect palette/layout → stages 4/5; suspect prompt → stage 10; suspect generation only → re-run `image-gen` on the existing `09-generation-prompts/`. When a checkpoint resume follows a slop verdict, append the rejected tropes to the spawning prompt of the resumed stage as exclusion constraints (they are already in the slop memory).

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
  02-title-link-reflection.json # per-candidate title-swap verdicts + the two selected tracks (or SEND_BACK notes)
  03-palette-planner-<rhetoric-id>.json # one palette direction + artists per selected rhetoric
  04-layout-planner-<rhetoric-id>.json  # one layout direction + title block + films per selected rhetoric
  05-artist-works.json          # all works (palette) + film posters (layout), grouped by rhetoric and role
  06-downloads/                 # downloaded artworks and posters (all)
  07-palette-features-<rhetoric-id>.json # palette features for one rhetoric's palette artists
  08-layout-features-<rhetoric-id>.json  # layout features for one rhetoric's layout posters
  09-generation-prompts/        # one .md per candidate, written before generation
  10-candidates/                # generated candidate PNGs (2-3)
  11-jury/                      # one blind-jury verdict .md per candidate
  12-provenance.json            # per candidate: palette source + layout source + rhetoric + features + jury verdict
  13-final.png                  # the user's pick
```

The final image (`12-final.png`) and `11-provenance.json` are copied to `output-dir` with the surface's `filename` at the end. The temp run dir may be kept for provenance or discarded.

## Input

- `surface` — a surface `id` from `references/surfaces.md` (e.g. `x-article`, `wechat-headline`, `wechat-image-message`, `xiaohongshu`, `square`, `4x3`, `9x16`, `16x9`).
- `article-path` — absolute path to the article file (the text to make a cover for).
- `output-dir` — absolute path of the directory where the final cover PNG is written.
- `title-override` — optional. When supplied, this string replaces the article's frontmatter title EVERYWHERE: the idea-extractor carries it as `article_title`, the title-link reflector judges against it, the layout-planner typesets it, and it is what renders on the image. Record it in `meta.json`.

If the surface is missing, ask once which surface, listing the `label` column from the registry; cache the answer in the session (it stays in the transcript). If `article-path` or `output-dir` is missing, ask once for each. **Default to New task only when no execution mode is stated; never infer rerun or checkpoint resume from existing files.**

## Workflow

1. Resolve `surface`, `article-path`, `output-dir`, **and the execution mode** (see **Run modes**). Read `references/surfaces.md`, look up the surface row, and read its `detail` file (`references/surfaces/<id>.md`). Build the **surface constraint** from the row: `aspectRatio` + `safeArea` + `bleed` + `text` + `cropBehavior` + `filename`.

   - **New task / rerun:** create the fresh timestamped run dir with subdirs `06-downloads/`, `09-generation-prompts/`, `10-candidates/`, `11-jury/`; write `meta.json` with the mode. A rerun also records its source run.
   - **Checkpoint resume:** create a fresh timestamped run dir, copy stages `1..N-1` from the named source dir into it, record the source run in the new dir's `meta.json`, and leave the source dir untouched.

   Continue with the selected run dir.
2. **Idea + rhetoric + orientation + article title.** Spawn `idea-extractor` with the article path AND `references/poster-principles.md` AND `references/visual-rhetoric.md` AND the slop memory file when it exists (see **Slop memory**) AND the `title-override` when one is supplied. It reviews the poster principles, surveys the article, develops THREE idea+rhetoric candidates, scores them on the poster tests in the reference, and outputs the highest-scoring one (with per-test breakdown and the two runners-up). It also carries the article's frontmatter `title` **verbatim** as `article_title` — it does NOT split or rephrase it; splitting is a layout decision. **Write** the full output to `01-idea-extractor.json`.
3. **Title-link reflection (gate).** Spawn `title-link-reflector` with the article title (the override when supplied), the idea-extractor's article survey, and all three candidates. It runs BOTH reader-side checks on each — the title-swap test and the visual-memory check — **selects the two rhetoric tracks for this run** (`chosen` and `runner-up-1`), and records any gap a track carries on either axis. **Write** the full output to `02-title-link-reflection.json`. On `SEND_BACK`, re-run step 2 with the reflection's notes appended to the spawning prompt as guidance (one send-back per run is normal; a second SEND_BACK means the title itself is too generic — surface that to the user instead of looping). Each selected track owns its own palette, layout, seed artworks, features, and generation prompt.
4. **[parallel] Palette directions.** For EACH selected rhetoric track (`chosen`, `runner-up-1`), spawn `palette-planner` with that track's idea, rhetoric, and orientation. **Write** one result per track to `03-palette-planner-<rhetoric-id>.json`.
5. **[parallel] Layout directions.** For EACH selected rhetoric track, spawn `layout-planner` with that track's idea, rhetoric, orientation, surface constraint, article title, and `references/titling.md`. **Write** one result per track to `04-layout-planner-<rhetoric-id>.json`. All four planners are independent — run them concurrently.
6. **Find works + posters.** Spawn `artist-works` with every palette artist and every layout film from both tracks, each tagged with its rhetoric track and its role (palette / layout), plus that track's rhetoric target. Palette artists are located on Google Art Project / Wikimedia; layout films are located as actual posters on a film poster database (Wikimedia Commons first, then IMPAwards, then MoviePosterDB / CineMaterial), each with film + year + designer (or unknown) + database name + page URL + image URL. **Write** to `05-artist-works.json`.
7. **Download works + posters.** For each palette work, download with `fetch.py` into `06-downloads/`. For each layout poster, download the recorded `image_url` directly (plain HTTP) into `06-downloads/`; if the database page exposes no direct image URL, fetch the page, extract the poster image, and download that. Retain every file's rhetoric-track and role tag.
8. **[parallel] Extract palette features.** Spawn one `artwork-feature-extractor` per downloaded palette work, across BOTH tracks, all concurrently. **Write** one grouped result per track to `07-palette-features-<rhetoric-id>.json`.
9. **[parallel] Extract layout features.** Spawn one `artwork-feature-extractor` per downloaded layout poster, across BOTH tracks, all concurrently. **Write** one grouped result per track to `08-layout-features-<rhetoric-id>.json`. Steps 8 and 9 may overlap; the runtime's queue handles overflow.
10. **Generate.** Each rhetoric track is independently art-directed. Spawn `prompt-composer` once per track, passing ONLY that track's rhetoric, palette direction/features, layout direction/features, **that track's layout provenance** (film + year + designer + database + page URL per layout source — the composer refuses to compose without it), and the shared surface constraint; write its candidate prompt(s) before generation. Generate two candidates from the `chosen` track and one from `runner-up-1`, each from the matching prompt, to `10-candidates/`. This prevents one weak rhetoric from poisoning the whole set and prevents a runner-up's content from being forced into the chosen rhetoric's layout.
11. **[parallel] Blind jury.** Spawn one `slop-juror` per candidate — all concurrently, each in a fresh context, each given ONLY the candidate's image path and the criterion (no article, no rhetoric, no pipeline context: jurors stay blind). The criterion covers both machine tropes and shoddy design — a beautifully rendered contradiction is still slop. **Write** each verdict verbatim to `11-jury/candidate-<n>.md`, then append every `yes`/`borderline` finding to the slop memory (see **Slop memory**). The jury is **advisory**: verdicts are presented with the candidates; the user rules.
12. **Record provenance.** Write `12-provenance.json` mapping each candidate to its rhetoric (chosen / runner-up), palette source (artist + work + URL) + layout source (film + year + designer + database + page URL) + features, the backend that generated it (with any ratio deviation), and its jury verdict.
13. **Ship + check.** Present the candidates WITH their jury verdicts; copy the user's pick to `13-final.png`, then to `output-dir` with the surface's `filename` (and copy `12-provenance.json` alongside if useful). Run `final-checks-verifier` (passing `article_title` as ground truth for the on-image text) for a light visual self-check. If the user rejects every candidate, choose the explicit next mode: a new **rerun** for a fresh attempt, or a **checkpoint resume** from stage 2 for slop-driven rhetoric repair / stage 10 for a taste-driven generation repair. The rigorous SAM-based crop-safety check is the host project's job (e.g. its preflight) — this skill does not ship a SAM stage. The host project handles any post-placement (e.g. copying into a post directory) and its own preflight — this skill stops at writing the cover to `output-dir`.

## Stacking rules

**MUST:**

- Let `idea-extractor` read the article in a clean context; it creates idea + rhetoric + orientation — do not hardcode any of them.
- Let `title-link-reflector` gate track selection: no planner runs before the reflection has picked `chosen` and `runner-up-1`; on SEND_BACK, re-run idea-extraction with the notes instead of forcing a track through.
- Select the chosen rhetoric and first runner-up as separate tracks; run the palette/layout planners, source lookup, and feature extraction once **per track**. Do not use the chosen track's art direction to render the runner-up.
- Keep the source split hard: layout sources come ONLY from film posters located on a film poster database; palette sources come ONLY from artworks (Google Art Project / Wikimedia). A painting, print, or design poster is never a layout source, whatever its composition.
- Let `artist-works` find works and posters for every artist and film from every track; keep each source tagged with its track and palette/layout role.
- Do NOT send any artwork image to the generator — the generator works from text only.
- Let `prompt-composer` compose each track's prompt from only that track's intermediate products — do not compose it yourself.
- Render the title **verbatim** from each track's `layout-planner` title block (`article_title` / `kicker` / `main`). Do not invent, rephrase, or re-split cover text.
- **Write each candidate's full prompt to `09-generation-prompts/candidate-<n>.md` before generating, and pass that file's content verbatim to the `image-gen` skill.**
- Generate 2–3 candidates concurrently — two from the chosen track, one from runner-up-1 — with each candidate using its matching track prompt.
- Keep jurors blind: a `slop-juror` spawn carries only the image path and the criterion — never the article, rhetoric, prompt, or other verdicts.
- Write every jury verdict to `10-jury/` and append every `yes`/`borderline` finding to the slop memory before presenting candidates.

**MUST NOT:**

- Send artwork images to the generator.
- Send a prompt to the generator that differs from what is written in `09-generation-prompts/candidate-<n>.md`, or generate before the candidate file is written.
- Invent or rephrase the cover title — it comes from `idea-extractor`'s `article_title` and is typeset by `layout-planner`; the main agent never authors or splits it.
- Start art direction before the title-link reflection has selected the tracks, or repair a failed bridge yourself — that judgment belongs to the reflector and the idea-extractor.
- Mix palette and layout sources — keep them separate until the generation prompt.
- Accept an artwork as a layout source, or compose a prompt whose layout provenance is missing or non-film — the composer returns REFUSED and the lookup step must be re-run.
- Skip provenance, the blind jury, or the final visual checks.
- Commit the slop memory to git — it is local, gitignored learning.
- Let a juror see anything but its one image and the criterion.
- Put text on the WeChat 头条 banner.

## Limits

- You **MUST NOT** edit the article file unless the user explicitly asks.
- You **MUST NOT** overwrite an existing cover asset in `output-dir` unless the user explicitly asks.