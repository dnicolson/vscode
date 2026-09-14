# Bump gulp from 4 to 5 (vinyl-fs 4 compatibility)

## Summary

Upgrades `gulp` from `^4.0.0` to `^5.0.1` in `package.json` (lockfile regenerated)
and makes the build pipeline compatible with the vinyl-fs 4 behavior this brings in.

- `build/lib/gulp/facade.ts`
  - Wraps `gulp.src` (`gulpSrc`) to default the `encoding` src option to `false`,
    i.e. read files as raw Buffers — matching vinyl-fs 3 / gulp 4 semantics.
  - Removes the now-unused `gulp-bom` (`bom`) import and export.
- `build/lib/compilation.ts`
  - Replaces gulp-bom's unconditional `bom()` with an idempotent `ensureUtf8Bom`
    stream that adds a UTF-8 BOM only when it is not already present.
  - Drops the `bom` facade import.

## Why

vinyl-fs 4 (bundled with gulp 5) changed two things that affect the build:

1. **Binary corruption.** `encoding` now defaults to `'utf8'`, and because the utf8
   codec is BOM-aware with `removeBOM` defaulting to `true`, every file read is
   passed through a decode/re-encode pass (`read-buffer.js`). Bytes that are not
   valid UTF-8 (in `.ttf`, `.png`, `.ico`, `.svg`, `.woff`, ...) are replaced with
   U+FFFD (`EF BF BD`), silently corrupting the compiled output — e.g. the codicon
   `.ttf` in `out/` grew from 152736 to 164787 bytes with invalid content, producing
   missing icons. `encoding: false` disables that codec pass and preserves bytes.
2. **BOM handling.** With raw Buffers, existing BOMs are no longer stripped on read
   (the `removeBOM` logic only runs inside the `if (encoding)` branch). An
   unconditional `bom()` would therefore produce a double BOM in fixtures that
   already carry one (`some_utf8_bom.txt`). `ensureUtf8Bom` only adds the BOM when
   it's missing, so all utf8 test fixtures match their previous gulp-4 output.

## Related (not in this diff)

vinyl-fs 4.0.2 also reintroduces a symlink ENOENT crash when `gulp.src` globs the
relative symlink in `extensions/terminal-suggest/src/test/fixtures/symlink-test/`
(`findSymlinkHardpath`/`resolveSymlinks`). That is fixed with a node_modules patch
(via the project's pkg-patch/vendoring workflow), not in this commit. It is
orthogonal to the encoding changes here.

## Verification

- `npx tsc --project build/tsconfig.json` passes.
- Byte-for-byte copies through the facade are identical for binary and BOM'd
  fixtures: `codicon.ttf` (152736 B), `code.png`, `some_utf8_bom.txt` (single BOM
  preserved), `some_utf8.css`.
- `gulp compile` succeeds and icons render correctly in the running app.

## Follow-up

`gulp-bom` (`"gulp-bom": "^3.0.0"` in `package.json`) is no longer used by any
build task; consider dropping the dependency and its typing stub
(`build/lib/typings/gulp-bom.d.ts`) together with the lockfile entries.