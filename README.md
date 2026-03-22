# Knip v6 Bug: `outDir` trailing slash breaks source path mapping

Minimal reproduction of a regression in knip v6.0.0 where a trailing slash in
`tsconfig.json`'s `outDir` causes package.json `exports` entries to fail
source-path resolution, producing false-positive "unused file" reports.

## The bug

When `tsconfig.json` has `"outDir": "lib/"` (trailing slash) and `package.json`
has subpath exports pointing to `lib/` paths:

```json
{
  "exports": {
    "./utils":   { "default": "./lib/utils/index.js" },
    "./helpers": { "default": "./lib/helpers.js" }
  }
}
```

Knip needs to map `lib/utils/index.js` back to `src/utils/index.ts` to mark the
source file as an entry point. In v6, this mapping breaks:

| Version | `outDir` resolved to | `rootDir` resolved to | `lib/utils/index.js` maps to |
|---------|---------------------|-----------------------|------------------------------|
| v5      | `.../lib`           | `.../src`             | `.../src/utils/index.ts`     |
| v6      | `.../lib/`          | `.../src`             | `.../srcutils/index.ts`      |

The `/` between `src` and `utils` is swallowed because `outDir` ends with `/`
but `srcDir` does not, and the mapping uses `String.replace()`.

## Root cause

Knip v6 replaced TypeScript's config parser with `get-tsconfig`'s
`parseTsconfig()`. Unlike TypeScript's `ts.parseJsonConfigFileContent()` which
normalizes paths (stripping trailing slashes), `get-tsconfig` preserves them.
Knip v6's `loadTSConfig` then calls `path.posix.join(dir, outDir)` which also
preserves the trailing slash, leading to an asymmetric replacement.

**Affected code:** `packages/knip/src/util/to-source-path.ts` —
`getToSourcePathsHandler` and `getModuleSourcePathHandler` both do
`filePath.replace(workspace.outDir, workspace.srcDir)`.

## Reproduce

```bash
# Install dependencies
cd v5 && pnpm install && cd ..
cd v6 && pnpm install && cd ..

# v5 passes — no issues found
cd v5 && pnpm test   # exit 0

# v6 fails — false positives for src/helpers.ts and src/utils/index.ts
cd v6 && pnpm test   # exit 1
```

Both directories contain **identical** source code and tsconfig. The only
difference is the knip version: v5 uses 5.88.1, v6 uses 6.0.0.

## Workaround

Remove the trailing slash from `outDir` in `tsconfig.json`:

```diff
-    "outDir": "lib/",
+    "outDir": "lib",
```

This is safe — TypeScript normalizes the path regardless of a trailing slash.

## Structure

```
v5/                     # knip 5.88.1 — passes
v6/                     # knip 6.0.0 — fails (false positives)
├── package.json        # exports with lib/ subpath entries
├── tsconfig.json       # outDir: "lib/" (trailing slash triggers bug)
└── src/
    ├── index.ts        # main entry (has "source" condition, works fine)
    ├── helpers.ts      # only reachable via exports "./helpers" → lib path
    └── utils/
        └── index.ts    # only reachable via exports "./utils" → lib path
```
