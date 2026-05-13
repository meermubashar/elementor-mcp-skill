---
name: elementor-mcp
description: Use this skill whenever you are working with an Elementor or Elementor Pro WordPress site via the Elementor MCP — recognizable by `mcp__*__elementor-mcp-*` tools being available in the session, or by the user asking to create, edit, refactor, or style a page on a WordPress site that uses Elementor (often with the Hello Elementor theme). Teaches the fluent patterns for building pages with native Elementor widgets (image, heading, text-editor, icon-box, container, etc.) instead of falling back to HTML widgets; the two-step `add-*` then `update-element` pattern for getting styling into Elementor's native data model so it remains editable from the panel UI; the global kit / `__globals__` strategy for brand consistency; common traps like `content_width: "boxed"` collapsing flex layouts; the safe-svg upload-roles gate; and the agent-browser visual-review loop. Trigger even when the user doesn't say "Elementor MCP" explicitly — if Elementor MCP tools are available in the session, this skill applies to any page-building, page-editing, or page-refactoring task on that site. Also trigger when the user mentions building a landing page, sales page, one-sheet, or pitch-deck-style page on a WordPress site that uses Elementor.
---

# Elementor MCP — fluent page building

Most pain when building Elementor pages via MCP comes from one mistake: dropping into HTML widgets the moment a styling argument is rejected by an `add-*` tool. The widgets accept much more than their strict `add-*` validators expose. The patterns below let you build pages that are visually faithful AND remain fully editable from the Elementor panel UI — which is the whole point of using Elementor.

## Workflow

1. **Read the source artifact first.** PDF, Figma, sketch, brief — whatever's authoritative for content and design. Don't guess.
2. **Get the site's global kit.** Call `elementor-mcp-get-global-settings` before placing the first container. It returns Primary/Secondary/Accent/Text colors and H1/H2/body typography — the source of truth for brand. Use these via `__globals__` references rather than redefining colors per element, so kit edits cascade.
3. **Narrate the layout before any build call.** Decompose into sections → containers → widgets → settings in prose. Catches design mistakes before they're 20 `add-*` calls deep, and turns the eventual implementation into a transcription rather than a design exercise.
4. **Create the page as draft first.** `elementor-mcp-create-page` with `status: "draft"`. Pick the right page template (see [Page template choice](#page-template-choice)).
5. **Build sections incrementally.** Container → its children → next container. Don't try to one-shot the whole page through `build-page` — small steps make iteration cheap.
6. **Publish, then visual review.** Use `agent-browser` (see [Visual review](#visual-review-with-agent-browser)). Compare against the source for both rendering glitches AND content fidelity. Iterate per-section using `update-element`, `update-container`, or `add-custom-css`.

## The two-step widget pattern

This is the single most important pattern in the skill. Internalize it.

**`add-<widget>` for content. `update-element` for styling.**

The `add-*` tool validators are strict — they accept only a widget's core content keys and reject most styling args. For example, `add-heading` rejects `title_color`; `add-image` rejects `width`. The tool descriptions list those keys because the widget really does accept them, but they live in conditional sub-control groups that the `add-*` schema validator doesn't expose.

`update-element` does a partial merge into the element's settings without the strict validator and accepts the full long-tail control surface.

**Example — accent-blue centered H2 heading:**

```
# Step 1: add the widget with content only
add-heading(post_id=1537, parent_id="abc123", title="The Next Evolution", header_size="h2")
→ returns element_id "64bfc7a"

# Step 2: apply the styling via update-element
update-element(post_id=1537, element_id="64bfc7a", settings={
  "align": "center",
  "title_color": "#366FFE",
  "typography_typography": "custom",
  "typography_font_family": "Helvetica",
  "typography_font_weight": "700",
  "typography_font_size": {"unit": "px", "size": 56, "sizes": []},
  "typography_font_size_mobile": {"unit": "px", "size": 34, "sizes": []},
  "typography_line_height": {"unit": "em", "size": 1.15, "sizes": []},
  "_margin": {"top": "24", "right": "0", "bottom": "16", "left": "0", "unit": "px", "isLinked": false}
})
```

Don't waste calls trying styling args inside `add-*` and recovering from rejections. Go straight to `update-element` after the add. This applies to every widget type.

**Why this matters:** styling stored in element settings shows up in the Elementor Style tab and is editable from the panel UI. Styling stored in Custom CSS blocks is not. The user shouldn't have to dig into code to change a color.

## Choosing widgets — favor native over HTML

HTML widgets defeat Elementor's value proposition. Once content is in `add-html`, the editor user can't drag, recolor, or reorder pieces from the panel — they're stuck with raw markup.

Use the table below to pick the right widget. Reach for `add-html` only when the design truly needs markup with no native equivalent (embeds, specific semantic HTML, etc.).

| Design pattern | Native expression |
|---|---|
| Heading + paragraph | `heading` + `text-editor` widgets in a container |
| Icon + title + description card | column container with image (or icon) + heading + text-editor |
| Feature list with bullets/icons | `icon-list` widget |
| Call-to-action button | `button` widget |
| Horizontal divider | `divider` widget |
| Gradient or solid panel with content | container with `_background_background: "gradient"` or `"classic"` |
| Single circular icon badge | image widget styled directly (see [Image-as-badge](#image-as-badge)) |
| Counter/stats display | `counter` widget |
| Testimonial block | `testimonial` widget |

**Decomposition principle.** When in doubt, prefer more widgets over fewer. Three native widgets (icon, title, description) in a column container beat one icon-box widget when the design needs custom styling, and beat one HTML widget almost always. Each widget is its own draggable, content-editable unit in the panel.

## Image-as-badge

A circular gradient icon badge can be the image widget itself — no wrapper container required. Set the wrapper width, padding, gradient background, and 50% border-radius on the image widget directly:

```
update-element(post_id=<id>, element_id="<image-widget-id>", settings={
  "width": {"unit": "px", "size": 52, "sizes": []},
  "_element_width": "initial",
  "_element_custom_width": {"unit": "px", "size": 96, "sizes": []},
  "_padding": {"top": "22", "right": "22", "bottom": "22", "left": "22", "unit": "px", "isLinked": true},
  "_background_background": "gradient",
  "_background_color": "#762ED3",
  "_background_color_b": "#366FFE",
  "_background_gradient_angle": {"unit": "deg", "size": 135, "sizes": []},
  "_border_radius": {"top": "50", "right": "50", "bottom": "50", "left": "50", "unit": "%", "isLinked": true}
})
```

96px wrapper + 22px padding all sides = 52px image area inside a perfect 50%-rounded circle. The image (typically a 1200×1200 SVG) renders centered. One widget, fully editable, no nested containers.

## Container width on flex children — the boxed trap

The single most common layout bug: a 3-column flex row that should sit horizontal collapses to a vertical stack.

**Cause:** child containers default to `content_width: "boxed"` (inherited from common usage), which adds the `e-con-boxed` class. Elementor's CSS forces `e-con-boxed` to 100% width regardless of the container's own `width` setting.

**Fix:** flex children that have explicit widths should be `content_width: "full"`. Their `width` setting (e.g. 340px) is then respected and they lay out as expected.

```
add-container(parent_id="<flex-row-parent>", settings={
  "content_width": "full",            # ← critical for flex children
  "width": {"unit": "px", "size": 340, "sizes": []},
  "flex_direction": "column",
  "flex_align_items": "center",
  ...
})
```

The parent flex row itself can stay `content_width: "boxed"` — that's fine for the row's outer wrapper. Only the flex CHILDREN need `"full"`.

## Custom CSS — fallback only

`add-custom-css` is the escape hatch, not the workhorse. Use it only when the styling has no native control:

- `<strong>`, `<em>`, `<a>` styling inside heading/text content
- `white-space: nowrap`
- Pseudo-elements (`::before`, `::after`)
- Complex selectors (`& + &`, `:nth-child`, etc.)
- Mix-blend modes on non-image elements
- CSS Grid declarations (Elementor's grid container support is limited)

If a styling decision can plausibly be expressed via `title_color`, `text_color`, `typography_*`, `_background_*`, `_border_*`, `_padding`, `_margin`, `_element_width`, `_element_custom_width`, `_z_index`, etc., it goes in `update-element` instead.

**`replace: true`** on `add-custom-css` overwrites the element's existing CSS block (defaults to append). Use it when shrinking a CSS block after moving styles to native controls.

## SVG icons via Media Library (safe-svg gate)

WordPress core blocks SVG uploads. The [safe-svg](https://wordpress.org/plugins/safe-svg/) plugin handles this safely, but **version 2.4.0+ has an opt-in role gate**: the plugin is active but its `upload_mimes` filter only registers when at least one role is added to the `safe_svg_upload_roles` option (default: empty array → no uploads allowed).

**If `wp media import` on an SVG fails with "Sorry, you are not allowed to upload SVG files":**

```bash
wp option update safe_svg_upload_roles '["administrator"]' --format=json
```

Then `wp media import path/to/icon.svg --user=<admin-username>` works and returns an attachment ID.

**Always prefer Media Library attachments over inline `<img src="...">` URLs** in image widgets — Media Library items show up in Elementor's image picker for easy swapping later, and they get proper srcset/lazy-loading.

If safe-svg isn't installed at all, recommend installing it (`wp plugin install safe-svg --activate`) before reaching for inline-SVG workarounds.

## Visual review with agent-browser

```bash
agent-browser set viewport 1280 800              # ALWAYS do this first
agent-browser open https://<site>/<slug>/
agent-browser screenshot --full /tmp/page.png
```

Then `Read` the screenshot.

**The default agent-browser viewport is narrow.** Without an explicit `set viewport`, screenshots come out at mobile-width and make every flex row look broken. Always size to desktop first unless you're specifically testing the mobile breakpoint.

For tighter UI loops, `agent-browser snapshot -i` gives a text outline of interactive elements with `@eN` refs that you can target with `agent-browser click @e3` etc.

After each visual-review pass, decide:
- **Rendering glitch?** → fix the offending element via `update-element` or `update-container`.
- **Content gap vs source?** → add the missing piece.
- **Both look right?** → mark this section done, move on.

## Page template choice

| Use case | Setting |
|---|---|
| Standalone landing page (no theme header/footer) | `template: "elementor_canvas"` |
| Full-width within site chrome | `template: "elementor_header_footer"` |
| Content page integrated with site nav | omit or `template: "default"` |

Set via `elementor-mcp-update-page-settings` after page creation. Use canvas for one-sheets, pitch decks, sales pages; default for content/blog pages.

## Element ID tracking

Every `add-*` call returns an element_id (7-character random string). You'll need these for `update-element`, `add-custom-css`, `remove-element`, `find-element`. They're easy to lose track of — keep a running map. A todo list or a scratch table works:

```
Hero container         2e6d951
  Logomark image       78dfdf2
  Wordmark heading     29cd8b9
Tagline (h6)           24de3db
Accent H1 (h2)         64bfc7a
Subhead text-editor    f3eda9f
```

If you lose an ID, recover it with `elementor-mcp-get-page-structure` or `elementor-mcp-find-element`.

## Refactoring HTML widgets to native widgets

If a section is built with HTML widgets and the user wants it refactored to native:

1. Read the structure: `elementor-mcp-get-page-structure` for the layout, `elementor-mcp-get-element-settings` on the HTML widget to capture the content.
2. Decompose the HTML widget's content into native widgets you'll place in the SAME parent container.
3. `elementor-mcp-remove-element` the HTML widget.
4. `add-*` each new native widget into the parent, capturing IDs.
5. `update-element` each new widget for native styling.
6. Visual-review the section; iterate.

This preserves the parent container's styling (background, padding, layout) and only swaps the content.

## Anti-patterns to avoid

| Anti-pattern | What to do instead |
|---|---|
| Falling back to `add-html` after one `add-*` styling rejection | Go straight to `update-element` with the styling keys |
| Skipping `get-global-settings` | Read the kit first; reference it via `__globals__` |
| Defining brand colors per-element | Use `__globals__` references: `"title_color__globals__": "globals/colors?id=accent"` |
| Re-uploading images already in Media Library | Check `wp media list` first |
| Wrapping every icon in its own badge container | Style the image widget itself — see Image-as-badge |
| Building a whole section in one `build-page` call | Build incrementally; iterate per-section |
| Forgetting `agent-browser set viewport` | Always set to 1280×800 (or wider) before screenshotting desktop layouts |
| Putting layout-affecting CSS in custom-css blocks | If there's a native control (padding, margin, flex), use it via update-element |
| Treating `add-custom-css` as primary, `update-element` as fallback | Inverse: native first, CSS only when no native control exists |

## Key tool reference

| Purpose | Tool |
|---|---|
| Read brand kit | `elementor-mcp-get-global-settings` |
| Create page | `elementor-mcp-create-page` |
| Set page template / page-level options | `elementor-mcp-update-page-settings` |
| Add container | `elementor-mcp-add-container` |
| Update container | `elementor-mcp-update-container` |
| Get widget's accepted schema | `elementor-mcp-get-widget-schema` |
| Get element's current settings | `elementor-mcp-get-element-settings` |
| See page tree | `elementor-mcp-get-page-structure` |
| Find elements by criteria | `elementor-mcp-find-element` |
| Add widget (heading/image/text-editor/button/icon-box/icon-list/divider/html/etc.) | `elementor-mcp-add-<widget>` |
| Apply styling to any element | `elementor-mcp-update-element` ⭐ |
| Remove element | `elementor-mcp-remove-element` |
| Per-element or page-level custom CSS | `elementor-mcp-add-custom-css` (use `replace: true` to overwrite) |
| Upload SVG icon for icon widgets | `elementor-mcp-upload-svg-icon` |
| Build complete page from declarative spec | `elementor-mcp-build-page` (powerful but hard to iterate) |

⭐ = the workhorse tool. If you're not calling `update-element` more than `add-custom-css`, you're probably still in HTML-widget muscle memory.
