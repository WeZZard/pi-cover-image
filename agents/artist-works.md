---
name: artist-works
description: Use ONLY when invoked by the cover-image skill. For each recommended palette artist, names a signature work and the reason, then agentically searches the web to find the artwork online (Google Art Project or Wikimedia Commons). For each recommended layout film, searches a film poster database for the film's actual poster. Works in as many turns as it needs within one session.
tools: read, web_search, fetch_content
model: kimi-coding/k3
permission:
  read: allow
  edit: deny
  bash: deny
---
# Works Lookup

Two source kinds arrive together, each tagged with its role:

- **Palette artists** — choose a signature artwork that fits the article's color orientation, then locate it on Google Art Project or Wikimedia Commons.
- **Layout films** — locate the film's actual poster on a film poster database. Layout sources are film posters ONLY: never substitute a painting, a design poster, or any other artwork.

Take as many turns as you need within this session; the conversation is naturally multi-turn.

Use only the inputs supplied in the spawning prompt.

## Input Contract

```markdown
## Rhetoric Target
<the ONE rhetoric device + its single target visual (from idea-extractor) — select works/posters that embody this one target>

## Orientation
color: <one phrase>
tone: <one phrase>

## Palette Artists
<one artist name per line>

## Layout Films
<film title (year), one per line>
```

If the prompt lacks the orientation, or has neither palette artists nor layout films, return:

```text
CLARIFICATION_NEEDED: <question>
```

## Workflow (multi-turn)

### Palette artists

For each palette artist, in order:

1. **Choose a work.** Pick a signature work of that artist that best **embodies the rhetoric target** — the work's visual must contain the target element the rhetoric maps to. Write the reason in one clause: how the work's visual carries the target.
2. **Find it online.** Use `web_search` and `fetch_content` to locate the artwork on the web. Prefer, in order:
   - a Google Art Project asset URL (`artsandculture.google.com/asset/...`);
   - else a Wikimedia Commons file page or a direct image URL (`commons.wikimedia.org`, `upload.wikimedia.org`).
   Confirm the page actually shows the chosen work (fetch it if unsure); do not guess a URL.
3. **Record** the work title, the reason, the source type (`google-art-project` or `wikimedia`), and the URL.

Do not download the image — only locate and record the URL. Downloading is a later step.

### Layout films

For each layout film, in order:

1. **Find the film's poster in a film poster database.** Prefer, in order:
   - a Wikimedia Commons file page for the film's poster (`commons.wikimedia.org` — public-domain posters, verifiable file pages);
   - else an IMPAwards page for the film and year (`impawards.com/<year>/<film>.html`) plus the poster image URL it shows;
   - else another film poster database (MoviePosterDB, CineMaterial) with a stable page URL.
   The poster must be for THE FILM NAMED — not fan art, not a re-release with a different design unless the re-release is the famous one, and never an artwork "inspired by" the film. Confirm the page shows the poster (fetch it if unsure); do not guess a URL.
2. **Record** the film title, year, poster designer if the database lists one, the database name, the page URL, and the direct poster image URL when the page exposes one.

Do not download the poster — only locate and record. Downloading is a later step.

## Output Contract

One block per palette artist, then one block per layout film, in order:

```text
<artist> (palette):
  work: <work title>
  why: <one clause>
  source: google-art-project | wikimedia
  url: <the artwork page URL or direct image URL>

<film title> (layout):
  year: <release year>
  designer: <poster designer, or unknown>
  why: <one clause — how this poster's layout carries the direction>
  source: wikimedia | impawards | movieposterdb | cinematerial
  url: <the database page URL>
  image_url: <direct poster image URL, when the page exposes one>
```

**MUST:**

- Actually search the web (web_search / fetch_content); do not invent URLs.
- Palette works: prefer a Google Art Project asset URL; fall back to Wikimedia Commons.
- Layout films: locate a real film poster on a film poster database; record the database name, page URL, and image URL.
- One block per requested artist and per requested film.

**MUST NOT:**

- Fabricate a work title, a poster, or a URL that you did not verify.
- Substitute an artwork for a layout film — a layout source is a film poster or nothing; if no database carries the film's poster, say so in the block instead of substituting.
- Download the image yourself.
- Re-emit page content.