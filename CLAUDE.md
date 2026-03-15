# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Tools for collecting, verifying, and compile-testing the source files that were actually compiled into a Yocto image's installed binaries. A single Python 3.10+ script (`yocto/source_audit.py`) with a unified CLI that operates on a Yocto build directory.

## Running the Tools

All commands require a Yocto build directory (`$BUILDDIR`) and an image manifest. The Yocto build environment must be sourced (bitbake required on PATH).

```bash
# Full pipeline: collect → verify → add missing sources → compile test → report → archive
python3 yocto/source_audit.py all -b $BUILDDIR -m <your-image> --clean

# Collect sources into ./output/sources/
python3 yocto/source_audit.py collect -b $BUILDDIR -m <your-image> --clean

# Verify: DWARF cross-check + coverage check
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image>

# Verify with bitbake compile test
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image> --compile

# Verify specific packages only
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image> --compile -p busybox,dropbear

# Verify with comprehensive audit
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image> --audit

# Generate interactive HTML report
python3 yocto/source_audit.py report -b $BUILDDIR -m <your-image>

# Create per-recipe source distribution archives (copyleft only by default)
python3 yocto/source_audit.py archive -b $BUILDDIR -m <your-image>
python3 yocto/source_audit.py archive -b $BUILDDIR -m <your-image> --all-licenses
```

Output goes to `./output/` by convention:
```
./output/
    sources/          # collected source files per package
    sources/MANIFEST.txt  # source counts summary
    licenses/         # license text files per recipe (deduplicated)
    patches/          # applied patches per recipe (deduplicated)
    src_uri/          # Yocto additional sources per recipe (deduplicated)
    copyrights/       # per-recipe extracted copyright notices
    archives/         # per-recipe source tarballs (copyleft, for GPL written offer)
    report.html       # interactive HTML report (with license, copyright, patch, Yocto additional sources, metadata, dependency tabs)
    report.csv        # CSV export (one row per source file per package, with Deselected status)
    report.xlsx       # Excel workbook with 8 sheets (requires openpyxl)
```

There is no test suite, linter, or build system.

## Architecture

Single-file script: `yocto/source_audit.py` (~5400 lines)

The script is organized into logical sections:

1. **Constants & Data classes** — `KernelInfo`, `PackageInfo`, `CompileCmd` dataclasses; `SOURCE_EXTS`, `KERNEL_IMAGE_GLOBS`, regex constants; `_COPYLEFT_KEYWORDS`, `is_copyleft()`; license obligation categorization (`_SPDX_OBLIGATIONS`, `_LEGACY_LICENSE_MAP`, `classify_license()`)
2. **Discovery** — `discover_packages()`, pkgdata helpers, manifest parsing, DWARF extraction, ELF helpers, argparse setup
3. **License/patch/Yocto-additional-sources/metadata/copyright helpers** — `get_license_files()`, `get_patch_files()`, `parse_src_uri()`, `get_src_uri_files()`, `get_pkg_metadata()`, `extract_copyrights_from_file()`, `extract_copyrights_for_recipe()`, `build_shlibs_map()`, `get_needed_libs()`, `resolve_linking_deps()`
4. **Compile-log parsing** — `parse_compile_log()`, `check_coverage()`, `compile_test()`, `_make_shadow_cmd()`
5. **YoctoSession** — Shared state dataclass with `from_args()`, `discover()`, `sources_dir`, `pkgdata_dir`, `work_dir` properties; auto-detects TMPDIR
6. **Collector** — Collects DWARF-confirmed source files, license texts, patches, Yocto additional sources, and copyright notices per package/recipe
7. **Verifier** — Cross-checks collected sources against DWARF CU paths
8. **Reporter** — Generates self-contained HTML report with Chart.js, CSV export, Excel workbook (openpyxl)
9. **HTML template** — Inline HTML/CSS/JS template string with 8 detail tabs per package (Sources, Installed Files, License, Copyright, Patches, Yocto Additional Sources, Metadata, Dependencies)
10. **Source Distribution Archiver** — `Archiver` class creates per-recipe `.tar.gz` tarballs for GPL compliance (copyleft-only by default)
11. **CLI** — Unified entry point with subcommands: `all`, `collect`, `verify`, `report`, `archive`

### Key classes

- **`YoctoSession`** — shared state (build_dir, tmpdir, manifest, machine, output_dir, cached package discovery, bitbake env caches); `from_args()` requires bitbake on PATH and queries `bitbake -e` for authoritative paths (TMPDIR, MACHINE, etc.); `query_recipe_env(recipe)` caches per-recipe `bitbake -e <recipe>` results for S, B, WORKDIR, STAGING_KERNEL_DIR, SRC_URI
- **`Collector`** — collects DWARF-confirmed source files, license texts (`output/licenses/`), patches (`output/patches/`), Yocto additional sources (`output/src_uri/`), and copyright notices (`output/copyrights/`) per recipe (deduplicated)
- **`Verifier`** — different verification strategies per package type (userspace DWARF, kernel `.o` cross-check, module `.mod` files)
- **`Reporter`** — generates interactive HTML report with 8 per-package detail tabs: Source Files, Installed Files, License (with obligation badges), Copyright, Patches, Yocto Additional Sources, Metadata, Dependencies; also generates `report.csv`, `report.xlsx` (Excel workbook with 8 sheets, requires openpyxl)
- **`Archiver`** — creates per-recipe `.tar.gz` tarballs including sources, licenses, patches, and Yocto additional sources; copyleft-only by default for GPL written offer compliance
- **License obligations** — `classify_license()` categorizes licenses into obligation flags: `source_distribution`, `object_linking`, `attribution`, `patent_grant`, `network_copyleft`, `permissive`; uses `_SPDX_OBLIGATIONS` lookup table with `_LEGACY_LICENSE_MAP` for Dunfell-era names and `_heuristic_obligations()` keyword fallback
- **Linking analysis** — `build_shlibs_map()` parses `shlibs2/*.list` to map sonames to provider packages; `resolve_linking_deps()` runs `readelf -d` on installed ELFs and resolves NEEDED libs to provider recipes/licenses, flagging copyleft dependencies

## Key Patterns

- Package types: `userspace`, `kernel_image`, `kernel_module`, `no_source` — all classes branch on `pkg.pkg_type`
- Source-root stripping: Yocto unpacks into versioned dirs (e.g. `busybox-1.35.0/`); `strip_src_root()` removes this prefix for stored paths
- Recipe prefix: `/usr/src/debug/{recipe}/{ver}/` — used to filter DWARF paths to same-recipe sources
- **`bitbake -e` path discovery**: `YoctoSession.from_args()` requires bitbake on PATH and runs `bitbake -e` once for global variables (TMPDIR, MACHINE, STAMPS_DIR, STAGING_BINDIR_NATIVE) and caches per-recipe queries (`bitbake -e <recipe>`) for S, B, WORKDIR, STAGING_KERNEL_DIR, SRC_URI. All results are disk-cached to `output/.bb_cache/`. Helpers: `_parse_bitbake_env()`, `_query_bitbake_global()`, `_query_bitbake_recipe()`, `_is_bitbake_available()`.
- **sstate detection**: `_warn_sstate()` checks if >50% of source packages have no work directory and warns the user to rebuild with `SSTATE_DIR="" SSTATE_MIRRORS="" bitbake --no-setscene <image>`. Called in `Collector.run()` and `cmd_verify()` after discovery.
- **`all` command pipeline**: `cmd_all()` runs 6 stages: (1) collect sources, (2) DWARF cross-check, (3) coverage check + copy missing sources from work tree into `output/sources/` via `check_coverage()`'s `not_collected_abs` mapping, (4) bitbake compile test with the now-complete source set, (5) generate HTML report, (6) create copyleft source archives.
- **`bitbake -e` disk cache**: Results from `bitbake -e` (global) and `bitbake -e <recipe>` are cached to `output/.bb_cache/*.json`. This avoids re-running expensive bitbake queries (~5-15s each) across commands. The cache is populated on first run and reused by subsequent `verify`, `report`, `archive` commands. Cleared by `--clean`.
- **TMPDIR auto-detection**: `_resolve_tmpdir()` auto-detects the Yocto TMPDIR instead of hardcoding `tmp`. Checks `bitbake -e` result first (if provided), then `conf/local.conf`, then scans for `tmp*/pkgdata` directories, falls back to `tmp`. All path construction uses the detected TMPDIR.
- External tools required: `readelf` (DWARF), `objcopy` (.o comparison), `bitbake` (required for all commands)
- `.o` mismatch in compile tests is expected (due to `-fdebug-prefix-map`) and reported separately from failures
- **Synthetic kernel-image**: The rootfs manifest lists `kernel-<version>` → `kernel-base` (a meta-package), not the actual `kernel-image-image`. `discover_packages()` injects a synthetic `kernel-image-image` entry from pkgdata when no `kernel_image` type is found.
- **Split-recipe packages**: Recipes like `glibc-locale` package pre-built binaries from `glibc` but have no `log.do_compile`. These are classified as `no_source` since compilation happened in the parent recipe.
- **Kernel module headers**: `Collector.collect_kernel_module()` collects `.h` headers by parsing Kbuild `.*.o.cmd` dependency files, same as `collect_kernel_image()` Step 2.

## Compile Test Internals (`python3 yocto/source_audit.py verify --compile`)

The compile test replays original gcc commands with the collected source copy via `_make_shadow_cmd`. Key subtleties:

- **Ninja/meson progress prefix**: Meson/ninja builds prefix compile lines with `[N/M]` (e.g. `[1/88] gcc ...`). `parse_compile_log` strips this prefix before parsing so it doesn't leak into the replayed command.

- **Cross-libtool prefix**: Libtool emits the actual gcc command as `<cross-prefix>libtool: compile:  gcc ...`. The cross-prefixed variant (e.g. `aarch64-poky-linux-libtool: compile:`) is matched by `\S*libtool:\s+compile:\s+` and stripped using `re.split` + take the last element. This handles both single and double libtool prefixes (from line truncation/re-emit in long lines like libgnutls30). The wrapper invocation (`--mode=compile`) is skipped entirely.

- **Shell operators**: Trailing `|| ( ... )` error-handling (bash), `&& true`/`&& :` chaining (nettle), and `>/dev/null 2>&1` redirections (sudo) are all stripped before tokenization.

- **Bare-quoted -D values**: The compile log may show `-DNAME="value"` where the double quotes are C string delimiters that need preserving. `_make_shadow_cmd` wraps such values in single quotes: `-DNAME='"value"'`. However, values containing shell metacharacters (`<>|&$` etc.) are left untouched — the double quotes are shell protection, not C quoting (e.g. `-DCURSESINC="<ncurses.h>"` uses quotes to protect angle brackets from shell redirection). Already-single-quoted values (e.g. `-DPROGRAM='"bash"'`) are left untouched.

- **Quoted source arguments**: Meson/ninja builds may quote source file arguments (e.g. `-c 'xkbcommon@sha/parser.c'`). The source file regex in `_make_shadow_cmd` uses backreference matching to handle both quoted and unquoted source tokens.

- **Generated-then-deleted sources**: Some build systems generate temporary `.c` files, compile them, then delete them (e.g. bash `mkbuiltins` generates `trap.c` from `trap.def`). If the original source no longer exists on disk, the compile test skips it to avoid compiling the wrong file (a different file may share the same stripped name).

- **Backtick VPATH expansion**: Some build systems (e.g. util-linux) use `` `test -f 'src.c' || echo '../pkg-ver/'`src.c `` in commands. `_make_shadow_cmd` strips backtick spans via regex and appends the collected source explicitly.

- **Include symlinks**: Collected source/header files may `#include "file.h"`, `#include "file.def"`, or `#include "file.tbl"` — files that aren't collected. Before running compile tests, the code symlinks missing `.h`/`.def`/`.tbl` files from the original tree into collected directories (and cleans them up after). Both the source tree and build tree are scanned for original files, since out-of-tree builds may reference headers from either location.

- **Ancestor `-iquote` paths**: `_make_shadow_cmd` adds `-iquote` for the original source file's parent directory and its ancestors up to `work_ver`, so `#include "..."` from the collected copy resolves headers in the original tree.

- **PATH fallback for Python do_compile**: Some recipes (e.g. psplash) use a Python `do_compile` function instead of a shell script. `_get_build_env` falls back to `run.oe_runmake.*` files to find the cross-compiler PATH when `run.do_compile` doesn't contain shell exports.
