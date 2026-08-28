# CLAUDE.md

## What this is

Martin van der Schelling's personal résumé/CV site — a single-page Jekyll site published by
GitHub Pages at **mpvanderschelling.nl** (`CNAME`) / `mpvanderschelling.github.io`.

It is a fork of [jglovier/resume-template](https://github.com/jglovier/resume-template)
(still wired up as the `upstream` remote). `README.md` is the **upstream template's** README,
not documentation of this site — don't treat it as authoritative and don't bother updating it.

**The publish branch is `gh-pages`**, which is also the default branch. Committing there
deploys. There is no `main`/`master`.

## Where the content lives

Almost nothing lives in markup. Content is in two places:

| File | Holds |
| --- | --- |
| `_config.yml` | name, title, intro blurb, contact email, social links, **section on/off switches** |
| `_data/*.yml` | the repeating entries of each section |

- `index.html` is empty apart from front matter selecting `layout: resume`.
- `_layouts/resume.html` renders every section and is the only place with real markup.
- `_includes/` holds `head.html`, the social-icon lists, and inline SVG icons.
- `_sass/` + `css/main.scss` hold the styles. `css/main.scss` needs its empty `---\n---`
  front matter or Jekyll won't compile it.

To edit résumé content, edit YAML. Only touch `_layouts/resume.html` when the *structure*
of a section has to change.

### Live vs. dummy data

Sections are gated by `resume_section_*` flags in `_config.yml`. The disabled ones still
contain the original Simpsons-themed template content:

- **Live and real:** `experience.yml`, `education.yml`, `associations.yml`, `skills.yml`
- **Off and still dummy:** `projects.yml`, `recognitions.yml`, `links.yml`, `interests.yml`

Never flip one of those flags to `true` without first replacing the file's dummy content.

## Writing entries

Each data file is a YAML list. Field names differ per section — check the corresponding
`for` loop in `_layouts/resume.html` before inventing a key; unknown keys render as
nothing, silently.

**Raw HTML inside YAML strings is the norm here**, and Liquid outputs it unescaped. Existing
entries use `<b>`, `<br>`, `<a href>`, `<sub>`, `&mdash;` for date ranges, and
`<ul class="resume-item-list">` for bullets. Match that.

**YAML quoting traps.** The long bodies are unquoted *plain multi-line scalars*, so inside them:

- a `:` followed by a space, **or a `:` at end of line**, is read as a key and breaks the parse
  (`mapping values are not allowed here`). End such a line with `:<br>` instead of a bare `:`.
- `https://` is safe — a `:` followed by a non-space is not a key separator.
- a leading `"` or `'` on the value needs quoting or a block scalar.

There is no Ruby here, so validate edits with:
`uv run --with pyyaml python3 -c "import yaml;yaml.safe_load(open('_data/education.yml'))"`
and confirm the long field came back as **one** string.

Images go in `images/` and are referenced **relatively** (`images/foo.png`, no leading slash),
wrapped in the site's figure idiom:

```html
<figure>
 <img src="images/foo.png" alt="..." class="centering" itemprop="image">
 <figcaption class="figcaption">Caption.</figcaption>
</figure>
```

`class="centering"` is 70% width, `class="centeringfull"` is 100%.

### The collapsible pattern

Experience / education / associations entries can hide a long body behind a toggle button:

```yaml
- company: ...
  collapse: True            # renders the button; omit or False for a plain entry
  collapse_name: Text on the button
  collapsed_text: <the long HTML body>
```

An entry can carry **more than one** dropdown: add a `collapses:` list of
`collapse_name` / `collapsed_text` pairs, which the experience loop renders *after* the single
`collapse:` block. (The Delft PhD entry uses both: "Learning to Choose Optimizers" via `collapse:`,
f3dasm via `collapses:`.) Only the Experience section has this loop &mdash; education and
associations still take one dropdown each.

Long bodies with a `:` in them (a paper title, an enumeration) can't be plain scalars &mdash; use a
folded block scalar (`>-`) as the f3dasm entry does. Keep every line at the same indentation or the
extra-indented ones keep their newlines.

**Gotcha:** `.content` in `_sass/_resume.scss` sets *both* `display: none` and `max-height: 0`,
so `_layouts/resume.html` ends with **two near-identical `<script>` blocks** — one toggling
`display`, one toggling `maxHeight`. They look like a copy-paste accident but both are load-bearing.
Deleting either leaves the collapsibles permanently closed.

## Known quirks worth not "fixing" blindly

- The **Skills** section (`resume_section_skills`) renders under an `<h2>Interests</h2>` heading,
  hardcoded in the layout. That's deliberate — `skills.yml` mixes tooling with hobbies. The
  separate `resume_section_interests` section renders as "Outside Interests".
- `resume_contact_telephone: "555-7334"` and `resume_contact_address: "742 Evergreen Terrace,
  Springfield"` in `_config.yml` are leftover template placeholders. They're hidden from view
  (`display_header_contact_info: "false"`) but **are still emitted as schema.org `<meta itemprop>`
  tags** in the page source. Fix or blank them if the subject comes up.
- The top-level `collapsed_text:` key in `_config.yml` is dead — a duplicate of the Imperial
  College abstract that now lives in `education.yml`.
- The avatar path (`images/photo.png`) and the footer name/email are hardcoded in
  `_layouts/resume.html`, not driven by `_config.yml`.
- `.travis.yml` is vestigial; GitHub Pages builds the site.
- `_data/experience.yml` carries large commented-out blocks (the Imperial College job, band entry).
  Leave them unless asked — they're the author's parking lot.

## Building

**There is no Ruby, Bundler, or Jekyll in this container** — `bundle exec jekyll serve` cannot
be run here. Verify changes by reading the Liquid, not by building.

**This file is `exclude`d in `_config.yml`, and must stay that way.** Jekyll treats every
root-level `.md` as a page and runs **Liquid over it before Markdown** — so a `{`+`%` tag in
this file, *even inside backticks or a fenced code block*, is parsed as a real tag and fails
the GitHub Actions build with `Liquid syntax error ... in CLAUDE.md`. Two rules follow:

1. Don't remove `CLAUDE.md` from the `exclude` list (and note that setting `exclude` at all
   *replaces* Jekyll's defaults, which is why they're restated there).
2. Still avoid writing literal Liquid tags here — describe them ("the `for` loop") instead.
   Belt and braces: the exclude keeps it out of the build, this keeps it harmless if that lapses. If the user wants a local
preview they need to run it on their own machine:

```
bundle install
bundle exec jekyll serve   # → localhost:4000
```

`.bundle/config` pins `BUNDLE_PATH` to `_vendor/bundle` (gitignored, along with `_site`).
`Gemfile.lock` pins `github-pages 204` → Jekyll 3.8.5, but **CI ignores it**: the
`actions/jekyll-build-pages` container builds with its own `github-pages 232` / **Jekyll 3.10.0**
and logs two harmless warnings every run — a long "The following gems are missing" list and
"the github-pages gem can't satisfy your Gemfile's dependencies". Those are noise, not build
failures; look past them to the actual error. Either way it's Jekyll 3, so Jekyll 4-only Liquid
is unavailable. An `update-deps` branch exists on the remote if the lockfile is ever refreshed.
