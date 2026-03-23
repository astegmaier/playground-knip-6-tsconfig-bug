# Knip v6 Bug: `outDir` trailing slash breaks source path mapping

Minimal reproduction of a regression in knip v6 where a trailing slash in
`tsconfig.json`'s `outDir` causes package.json `exports` entries to fail
source-path resolution, producing false-positive "unused file" reports.

## Repro Steps

```bash
pnpm install
pnpm test:v5   # exit 0 — no issues
pnpm test:v6   # exit 1 — false positive: src/greet.ts reported as unused
```

Both `v5/` and `v6/` contain **identical** source code, tsconfig, and
package.json exports. The only difference is the knip version (5.88.1 vs 6.0.3).

## The bug

When `tsconfig.json` has `"outDir": "lib/"` (with a trailing slash):

```json
{
  "compilerOptions": {
    "outDir": "lib/",
    "srcDir": "src/"
  }
}
```

...and `package.json` has an export pointing to a `lib/` path:

```json
{
  "exports": {
    "./greet": "./lib/greet.js"
  }
  // ...
}
```

Knip 6 is unable to map `lib/greet.js` back to `src/greet.ts`, causing it to report `src/greet.ts` as an unused file, even though it's correctly exported and used.

Knip 5 handles this case fine.

## Workaround

Remove the trailing slash from `outDir` in `tsconfig.json`:

```diff
-    "outDir": "lib/",
+    "outDir": "lib",
```

This is safe — TypeScript normalizes the path regardless of a trailing slash.

## Possible Root Cause

_Everything from here down is LLM-generated speculation that I haven't yet verified._

Knip needs to map `lib/greet.js` back to `src/greet.ts` to mark the source file
as an entry point. In v6, this mapping breaks:

| Version | `outDir` resolved to | `srcDir` resolved to | `lib/greet.js` maps to |
|---------|---------------------|-----------------------|------------------------|
| v5      | `.../lib`           | `.../src`             | `.../src/greet.ts`     |
| v6      | `.../lib/`          | `.../src`             | `.../srcgreet.ts`      |

The `/` between `src` and `greet` is swallowed because `outDir` ends with `/`
but `srcDir` does not, and the mapping uses `String.replace()`.

Knip v6 replaced TypeScript's config parser with `get-tsconfig`'s
`parseTsconfig()`. Unlike TypeScript's `ts.parseJsonConfigFileContent()` which
normalizes paths (stripping trailing slashes), `get-tsconfig` preserves them.
Knip v6's `loadTSConfig` then calls `path.posix.join(dir, outDir)` which also
preserves the trailing slash, leading to an asymmetric replacement.

**Affected code:** `packages/knip/src/util/to-source-path.ts` —
`getToSourcePathsHandler` and `getModuleSourcePathHandler` both do
`filePath.replace(workspace.outDir, workspace.srcDir)`.
