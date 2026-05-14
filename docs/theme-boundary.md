# Theme Boundary

This site uses BeautifulHugo as a pinned, read-only git submodule at `themes/beautifulhugo`.

## Ownership Rules

- `themes/beautifulhugo/` is upstream vendor code. Do not edit files there directly.
- `layouts/` contains site-owned template overrides. Changes here intentionally shadow theme templates.
- `static/assets/` contains personal site assets.
- `static/css/main.css` is site-owned CSS, even though parts may have originated from the theme.
- Other root `static/css/` files are pinned browser/vendor assets used by the local overrides.

## Working With The Theme

- Restore the theme with `git submodule update --init --recursive` after cloning.
- Treat theme upgrades as explicit maintenance work: update the submodule, compare root `layouts/` overrides against upstream templates, then run a full Hugo build.
- Keep `hugo.toml` using `theme = "beautifulhugo"` unless the theme dependency is intentionally removed and all required `data`, `i18n`, JS, and static assets are vendored into the root project.

## Verification

Run these checks after theme or layout changes:

```sh
git submodule status --recursive
hugo --printI18nWarnings --printPathWarnings
hugo --destination /tmp/roaming-public --cleanDestinationDir
test -f /tmp/roaming-public/js/main.js
test -f /tmp/roaming-public/fontawesome/css/all.css
test -f /tmp/roaming-public/css/main.css
```
