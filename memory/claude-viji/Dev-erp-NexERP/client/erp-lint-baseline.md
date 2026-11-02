---
name: erp-lint-baseline
description: `npm run lint` is red before any change — eslint crashes on ecosystem.config.cjs and the repo is over its 344-warning budget; lint touched files instead
metadata:
  type: reference
---

As of 2026-10-02, `npx eslint .` crashes with "could not find plugin
react-hooks": the config applies `react-hooks/rules-of-hooks` to every file but
`eslint-config-next` only registers the plugin for js/ts globs, and the
committed `ecosystem.config.cjs` (PM2) falls outside them. With
`--ignore-pattern ecosystem.config.cjs` it runs: 8 errors, ~439 warnings,
above `--max-warnings=344` in package.json. Neither is caused by feature work —
judge a change by `npx eslint <touched files> -f json` against the pre-change
file (copy it from `git show HEAD:<path>` to a temp name inside the repo so
the config applies).
