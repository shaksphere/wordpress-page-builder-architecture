# no-build-wp-builder-architecture

A [Claude Code](https://claude.com/claude-code) **skill** capturing the
architecture for building a WordPress visual/page builder plugin **with no build
step** — plain PHP + vanilla JS, installable straight from a ZIP.

## What it covers

- A **single PHP renderer** that drives both the front end and a **same-origin
  iframe editor canvas** (true WYSIWYG, no drifting second renderer).
- A **JSON node tree in post meta** as the data model.
- **Sanitize-on-input / escape-on-output**, with one control schema shared by the
  editor UI and the sanitizer.
- **Compiled CSS written to cacheable files** with an inline fallback.
- Rendering theme chrome around an editable region, a typographic-baseline trick
  for editor/front-end parity, the widget registry pattern, REST/nonce setup, and
  the empty-map JSON round-trip gotcha.

It's a distilled, reusable version of the patterns behind
[Open Builder](https://github.com/shaksphere/open-builder).

## Install

```bash
git clone https://github.com/shaksphere/no-build-wp-builder-architecture.git \
  ~/.claude/skills/no-build-wp-builder-architecture
```

Claude Code loads skills from `~/.claude/skills/` (user scope) or a project's
`.claude/skills/`. It activates when a task matches the description — e.g.
designing or extending a custom WordPress builder, block renderer, or live
editor in PHP + vanilla JS.

## License

MIT. Extracted from building Open Builder with Claude Code.
