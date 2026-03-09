# Fork Patches

This branch carries patches above
[upstream `mmistakes/minimal-mistakes`](https://github.com/mmistakes/minimal-mistakes).
Each section documents a patch, why it exists, and the upstream issue or PR it
relates to.

## 1. Sass deprecated built-in function migration

Replace deprecated `/` division operator with `math.div()` and deprecated
global color functions (`red()`, `green()`, `blue()`) with their `sass:color`
module equivalents. Dart Sass deprecated these in 1.33.0 and will remove them
in 2.0.

**Note:** The global `mix()` function (79 calls across `_variables.scss` and
skin files) is the same category of deprecation but is not addressed here due
to scope. It would be a separate patch.

**Files changed:**
- `_sass/minimal-mistakes/_forms.scss`
- `_sass/minimal-mistakes/_mixins.scss`
- `_sass/minimal-mistakes/vendor/breakpoint/_helpers.scss`
- `_sass/minimal-mistakes/vendor/breakpoint/parsers/resolution/_resolution.scss`
- `_sass/minimal-mistakes/vendor/magnific-popup/_magnific-popup.scss`
- `_sass/minimal-mistakes/vendor/magnific-popup/_settings.scss`
- `_sass/minimal-mistakes/vendor/susy/susy/_su-math.scss`
- `_sass/minimal-mistakes/vendor/susy/plugins/svg-grid/_svg-utilities.scss`

**Upstream:**
- [#4054](https://github.com/mmistakes/minimal-mistakes/issues/4054) — Deprecation Warning for sass (open)
- [#4303](https://github.com/mmistakes/minimal-mistakes/issues/4303) — Deprecation warnings for `/` outside calc (open)
- [#4858](https://github.com/mmistakes/minimal-mistakes/pull/4858) — PR closed accidentally
- Maintainer has not acted; concern is breaking remote-theme users on old `github-pages` gem.

## 2. Wrap declarations after nested rules with `& {}`

Dart Sass warns about "mixed declarations" when plain CSS properties appear
after `@include` blocks that expand to nested rules or `@media` queries. Wrap
the affected properties in `& {}` to create a new nesting context.

**Files changed:**
- `_sass/minimal-mistakes/_masthead.scss` — properties after `@include clearfix`
- `_sass/minimal-mistakes/_reset.scss` — properties after `@include breakpoint()`

**Upstream:** No tracking issue. This is a strict-mode Sass compatibility fix.

## 3. Preload header overlay images

Add a `<link rel="preload">` hint for `page.header.overlay_image` so the
browser starts fetching it immediately rather than waiting for CSS parsing.
Only emitted on pages that set an overlay image; no effect on other pages.

**Files changed:**
- `_includes/head.html`

**Upstream:**
- [#5241](https://github.com/mmistakes/minimal-mistakes/pull/5241) — Preload Header Overlay Images (open)

## 4. Add `fediverse:creator` meta tag support

Emit `<meta name="fediverse:creator">` for link preview attribution on
Mastodon, Pixelfed, Flipboard, and other fediverse platforms. Parallel to the
existing `twitter:creator` support. Uses `author.fediverse` from `_config.yml`
or per-page author data.

**Files changed:**
- `_includes/seo.html`

**Upstream:** No tracking issue or PR.

## 5. Update hardcoded icon classes for Font Awesome 6

The theme loads FA6 from CDN but several hardcoded icons in the author profile
and footer still use FA5 class names. FA6 ships backward-compatibility aliases
today, but these are not guaranteed to persist. Update to native FA6 names.

**Renames applied:**
- `fa-map-marker-alt` → `fa-location-dot`
- `fa-envelope-square` → `fa-square-envelope`
- `fa-twitter-square` → `fa-square-x-twitter`
- `fa-facebook-square` → `fa-square-facebook`
- `fa-xing-square` → `fa-square-xing`
- `fa-tumblr-square` → `fa-square-tumblr`
- `fa-lastfm-square` → `fa-square-lastfm`
- `fa-rss-square` → `fa-square-rss`

**Files changed:**
- `_includes/author-profile.html`
- `_includes/footer.html`

**Upstream:** No tracking issue or PR. The `_utilities.scss` color rules already
reference FA6 names alongside FA5 aliases.

## 6. Add missing brand-color rules for social platforms

Fix `$bluesky-color` having a variable but no icon-class mapping (bug), add
brand colors for Discord, Signal, Telegram, Threads, and WhatsApp, and add
FA6 `fa-square-*` selectors alongside existing FA5 `fa-*-square` selectors
so both naming conventions get the correct brand color.

**Files changed:**
- `_sass/minimal-mistakes/_variables.scss` — new color variables
- `_sass/minimal-mistakes/_utilities.scss` — updated `@each` loop

**Upstream:**
- [#4577](https://github.com/mmistakes/minimal-mistakes/issues/4577) — Missing brand colors for social icons

## 7. Add custom sidebar content hook

Add `{% include sidebar-custom.html %}` before the closing `</div>` in
`_includes/sidebar.html`, with an empty `_includes/sidebar-custom.html`
extension point. This follows the existing pattern of `head/custom.html` and
`author-profile-custom-links.html`, letting users inject content (blogroll,
widgets, etc.) without shadowing the entire sidebar file.

**Files changed:**
- `_includes/sidebar.html` — include hook added
- `_includes/sidebar-custom.html` — empty extension point (new)

**Upstream:** No tracking issue or PR.

## 8. Add config toggle to suppress taxonomy display on posts

Wrap the tag/category list rendering in `{% unless site.show_taxonomy == false %}`.
Default behavior is unchanged (taxonomy shown). Set `show_taxonomy: false` in
`_config.yml` to suppress tag and category lists on posts.

**Files changed:**
- `_includes/page__taxonomy.html`

**Upstream:** No tracking issue or PR.
