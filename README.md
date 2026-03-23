# Knip v6 Bug: `outDir` trailing slash breaks source path mapping

Minimal reproduction of a regression in knip v6.0.0 where a trailing slash in
`tsconfig.json`'s `outDir` causes false-positive "unused file" reports.

## Reproduce

```bash
pnpm install
pnpm test:v5   # exit 0 — no issues
pnpm test:v6   # exit 1 — false positive: src/greet.ts reported as unused
```

`v5/` and `v6/` are identical except for the knip version (5.88.1 vs 6.0.0).

## Root cause

Knip maps `lib/greet.js` (from `exports`) back to `src/greet.ts` using
`String.replace(outDir, srcDir)`. Knip v6 replaced TypeScript's config parser
with `get-tsconfig`, which preserves the trailing slash in `"outDir": "lib/"`.
TypeScript's parser stripped it. The mismatch means the replacement produces
`.../srcgreet.ts` instead of `.../src/greet.ts`.

## Workaround

```diff
-    "outDir": "lib/",
+    "outDir": "lib",
```
