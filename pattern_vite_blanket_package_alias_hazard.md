---
name: pattern-vite-blanket-package-alias-hazard
description: "Why aliasing a scoped npm package (e.g. @mui/*) straight to a node_modules directory in Vite/bundler config can silently break dual-format packages, and how to scope the alias instead"
aliases:
  - Vite Blanket Alias Dual-Package Hazard
tags:
  - pattern
  - vite
  - bundling
created: 2026-08-05
---

# Vite blanket `@scope/*` alias breaks dual-format (CJS+ESM) packages

A common workaround in a monorepo: some source directory has no `node_modules` ancestor of
its own (e.g. shared packages living outside any app's install tree), so its bare imports of a
scoped package like `@mui/material` can't resolve. The quick fix is a regex alias:

```js
{ find: /^@mui\//, replacement: '/path/to/some/app/node_modules/@mui/' }
```

**This is a trap.** Bundler alias resolution (Vite, webpack, etc.) is global — it doesn't know
or care *which file* triggered the import. That same alias also intercepts every *other*
import of `@mui/*` in the whole build, including a package's own **internal** cross-package
imports (e.g. `@mui/material`'s source importing `@mui/system`). Aliasing straight to a
directory path bypasses `package.json`'s `exports` map entirely — module resolution falls back
to the legacy `main` field instead of the `exports`-declared entry point.

For a package that ships **both** a CJS build and a real ESM build (common for anything still
supporting old bundlers) — where the CJS build re-exports its default via a Babel-interop
convention (`exports.__esModule = true; exports.default = fn`) — this means:
- Code that reaches the package through the alias gets the CJS build (`main` field).
- Code that reaches it through a real `import` statement elsewhere (unaffected by the alias,
  or reached before the alias applies) gets the real ESM build.
- These are two different module instances with two different default-export shapes. Whichever
  side does Babel-style interop unwrapping (`mod.default`) on the *wrong* shape gets back an
  object instead of a function — surfacing as `TypeError: ... is not a function` at first call,
  with **no build-time error at all**, since both forms typecheck/bundle fine individually.

**Fix**: never blanket-alias a scoped package. Scope the alias to only the specific importer
paths that actually need it, using a `customResolver` (Vite's `resolve.alias` entries support
this) or an equivalent importer-aware resolve hook — return `null`/`undefined` for every other
importer so normal, `exports`-map-respecting resolution proceeds untouched:

```js
{
  find: /^@mui\//,
  replacement: '@mui/', // unused when customResolver is set, but required by the type
  customResolver(source, importer) {
    const needsAlias = importer?.includes('/packages/') && !importer.includes('/node_modules/');
    if (!needsAlias) return null;
    return source.replace(/^@mui\//, '/path/to/app/node_modules/@mui/');
  }
}
```

If you hit `is not a function` on a package you know works fine standalone (test it directly
with `node --input-type=module -e "import x from 'pkg'; console.log(typeof x)"` — if that
prints `function`, the package itself is fine), suspect a blanket alias like this before
anything else.
