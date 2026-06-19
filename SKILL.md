---
name: no-build-wp-builder-architecture
description: Architecture patterns for building a WordPress visual/page builder plugin with no build step — a single PHP renderer that drives both the front end and a same-origin iframe editor canvas, a JSON node tree stored in post meta, sanitize-on-input/escape-on-output, and compiled CSS written to cacheable files. Use when designing or extending a custom WordPress drag-and-drop builder, block/section renderer, or live editor in PHP + vanilla JS without a webpack/Vite toolchain.
---

# No-build WordPress visual builder architecture

A field-tested blueprint for a WordPress page/section builder that runs the
moment it's dropped into `wp-content/plugins/` — no `npm install`, no bundler.

## Core principles

1. **No build step.** The editor is plain ES5/ES6 JavaScript loaded directly.
   This keeps the plugin installable from a ZIP and trivial to debug in the wild.
2. **Single source-of-truth renderer.** One PHP `Renderer` turns the saved data
   into HTML. The front end *and* the editor canvas both call it, so the editor
   is true WYSIWYG by construction — there is no second, drifting JS renderer.
3. **Data as a sanitized JSON node tree in post meta.** Each node:
   `{ id, type, settings: { content, style, background, advanced, dynamic }, children }`.
   Store it as a JSON string in a meta key (e.g. `_openb_tree`).
4. **Sanitize on input, escape on output.** Sanitize the whole tree on save
   against each widget's declared control schema; escape every value at render.
5. **Backwards-compatible data.** Never break an existing saved tree; coerce/
   migrate. (JS gotcha below.)

## The editor canvas = a same-origin iframe

- Serve a minimal preview document at a query var (e.g. `?myeditor_preview=ID`),
  gated by `current_user_can('edit_post', $id)`. It contains one element —
  `<div id="canvas">…server-rendered tree…</div>` — plus the front-end CSS.
- The editor app (parent window) owns the node tree in memory, mutates it
  locally, and asks the server to re-render via a debounced REST call, swapping
  the returned HTML into `#canvas`. Same origin → the parent can reach into
  `iframe.contentDocument` to decorate nodes (select, hover, drop zones) without
  postMessage gymnastics.
- A `Render_Context` flag (`editor` vs `frontend`) lets dynamic widgets show
  placeholders in the canvas and live data on the front end.

## Rendering chrome "around" the editable region

To design in context (e.g. theme header/footer around page content), render the
chrome **once, server-side, outside** `#canvas`. The live re-renders only touch
`#canvas`, so the chrome is never disturbed. Mark those outside regions as
non-selectable and scope canvas decoration to `#canvas [data-node-id]` only.

## CSS strategy

- Compile each node's styles to a class scoped by node id (`.ob-<id>`), with
  desktop as the base and tablet/mobile inside `max-width` media queries.
- Write compiled CSS to **physical files** in `uploads/` (a site-wide globals
  file + a per-page file), enqueued with an `mtime` version, and **fall back to
  inline `<style>`** if the filesystem isn't writable. Cacheable, not re-parsed
  every request.
- Establish an explicit typographic baseline scoped to your root wrapper so the
  bare editor iframe and the themed front end render identically (don't inherit
  browser defaults in one place and theme fonts in the other).

## Widget registry

- One class per widget implementing a small interface: `type()`, `controls()`
  (the schema consumed by *both* the JS editor UI and the PHP sanitizer),
  `render($content, $innerHtml, $node)`. Register them in a static registry so
  the sanitizer can validate a tree without a full plugin instance.
- The control schema is the contract: the editor builds inputs from it, and the
  sanitizer validates against it. One definition, two consumers.

## REST

- Editor routes (`/render`, `/save`, …) require `edit_post` + the standard
  `X-WP-Nonce` cookie auth. Public routes (e.g. form submit) are guarded by a
  per-form nonce. Never auto-expose the raw tree meta in REST.

## JS gotcha: empty maps round-trip as arrays

PHP's `json_encode([])` is `[]`, and assigning a string key to a JS `Array`
(`style['desktop'] = …`) is dropped by `JSON.stringify`. On load, **coerce**
`settings.content/style/background/advanced/dynamic` (and per-breakpoint maps) to
plain objects, or newly-set values silently fail to save.

## Suggested module layout

```
plugin.php                 bootstrap + constants
includes/
  class-plugin.php         DI/boot
  class-post-types.php     meta keys + CPTs (templates, popups)
  class-widgets.php        registry
  widgets/                 one class per widget
  class-renderer.php       tree -> HTML (front end + canvas)
  class-css-generator.php  tree -> scoped CSS
  class-css-store.php      write/cache CSS files (+ inline fallback)
  class-security.php       recursive sanitizer, escapers, whitelists
  class-rest.php           nonce-guarded routes
  class-editor.php         serves the editor app + preview iframe
  class-frontend.php       the_content takeover + asset enqueue
editor/js/editor.js        the vanilla-JS app
assets/css/frontend.css    shared front-end + canvas styles
```

See [Open Builder](https://github.com/shaksphere/open-builder) for a complete
reference implementation.
