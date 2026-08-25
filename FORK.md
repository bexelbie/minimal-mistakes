# Fork Patches

This branch (`bex-master`) carries patches above
[upstream `mmistakes/minimal-mistakes`](https://github.com/mmistakes/minimal-mistakes)
that upstream cannot or will not merge.

Last rebased onto: **post-4.28.0** (`1d03dd1b`, 2026-04-20)

---

## 1. Sass deprecated built-in function migration

Replace the deprecated `/` division operator with `math.div()` and deprecated
global color functions (`red()`, `green()`, `blue()`) with their `sass:color`
module equivalents. Dart Sass deprecated these in 1.33.0 and will remove them
in 2.0.

**Note:** The global `mix()` function (79 calls across `_variables.scss` and
skin files) is the same category of deprecation but is not addressed here due
to scope.

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
- [#4303](https://github.com/mmistakes/minimal-mistakes/issues/4303) — Deprecation warnings for `/` outside calc (upstream attempted a fix in `9b19c5d2` then reverted it in `1d03dd1b`)

**Why not submitted:** The `github-pages` gem pins `jekyll-sass-converter`
1.5.2, which uses libsass/sassc and does not support `math.div()` or
`sass:color` modules. Submitting would break builds for anyone using
`remote_theme:` with the classic GitHub Pages pipeline.

## 2. Wrap declarations after nested rules with `& {}`

Dart Sass warns about "mixed declarations" when plain CSS properties appear
after `@include` blocks that expand to nested rules or `@media` queries. Wrap
the affected properties in `& {}` to create a new nesting context.

**Files changed:**
- `_sass/minimal-mistakes/_masthead.scss`
- `_sass/minimal-mistakes/_reset.scss`

**Upstream:** Same `github-pages` gem constraint as §1.

## 3. Preload header overlay images

Add a `<link rel="preload">` hint for `page.header.overlay_image` so the
browser starts fetching it before CSS parsing discovers the reference.

**Files changed:**
- `_includes/head.html`

**Upstream:**
- [#5241](https://github.com/mmistakes/minimal-mistakes/pull/5241) — Preload Header Overlay Images (open, but maintainer unlikely to merge)

## 4. Workflow action version updates

Update the GitHub Actions dependencies used by the upstream build workflow:
`actions/checkout` from `v4` to `v7` and `actions/cache` from `v4` to `v6`.

**Why not submitted:** The workflow's build job only runs in the upstream
`mmistakes/minimal-mistakes` repository, so these fork maintenance updates have
no effect on upstream theme consumers.

---

## Previously carried patches (now merged upstream in 4.28.0)

The following patches were developed in this fork and accepted upstream:

- **aria-label on nav elements** — [#5442](https://github.com/mmistakes/minimal-mistakes/pull/5442)
- **IndieWeb microformats + footer rel** — [#5443](https://github.com/mmistakes/minimal-mistakes/pull/5443)
- **og:image:alt support** — [#5444](https://github.com/mmistakes/minimal-mistakes/pull/5444)
- **fediverse:creator meta tag** — [#5445](https://github.com/mmistakes/minimal-mistakes/pull/5445)
- **Font Awesome 6 icon classes** — [#5446](https://github.com/mmistakes/minimal-mistakes/pull/5446)
- **Brand-color rules for newer platforms** — [#5447](https://github.com/mmistakes/minimal-mistakes/pull/5447)
- **Custom sidebar content hook** — [#5448](https://github.com/mmistakes/minimal-mistakes/pull/5448)
- **Taxonomy display toggle** — [#5449](https://github.com/mmistakes/minimal-mistakes/pull/5449)
