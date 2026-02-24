# oss-clearance

Collect, verify, and audit the source files compiled into a Yocto image — for OSS clearance and license compliance.

## Prerequisites

- Python 3.10+
- `readelf` and `objcopy` (from binutils)
- `openpyxl` (optional, for Excel report: `pip install openpyxl`)

**Important:**

- You **must** source your Yocto build environment before running the script (`bitbake` must be on PATH).
- **Do not** use `sstate` (shared state cache). The script needs full build artifacts. Build with:
  ```bash
  SSTATE_DIR="" SSTATE_MIRRORS="" bitbake --no-setscene <image>
  ```

## Quick start

```bash
# Source Yocto environment
source poky/oe-init-build-env build

# Run everything: collect → verify → compile test → report → archive
python3 yocto/source_audit.py all -b $BUILDDIR -m <your-image> --clean
```

## Commands

```bash
# Collect sources, licenses, patches, copyrights
python3 yocto/source_audit.py collect -b $BUILDDIR -m <your-image> --clean

# Verify collected sources (DWARF cross-check + coverage)
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image>

# Verify with bitbake compile test
python3 yocto/source_audit.py verify -b $BUILDDIR -m <your-image> --compile

# Generate HTML + CSV + Excel reports
python3 yocto/source_audit.py report -b $BUILDDIR -m <your-image>

# Create per-recipe source archives (copyleft only by default)
python3 yocto/source_audit.py archive -b $BUILDDIR -m <your-image>
```

### Flags

| Flag | Description |
|------|-------------|
| `-b / --build-dir` | Yocto build directory (`$BUILDDIR`) |
| `-m / --manifest` | Image name or path to `.manifest` file |
| `--machine` | Machine name (auto-detected) |
| `--clean` | Remove output and caches before collecting |
| `-v / --verbose` | Verbose output |
| `--compile` | Run bitbake compile test (verify command) |
| `--audit` | Run comprehensive audit (verify command) |
| `-p pkg1,pkg2` | Verify specific packages only |
| `--all-licenses` | Archive all recipes, not just copyleft |

## Output

```
./output/
    sources/           # DWARF-confirmed source files per package
    licenses/          # license texts per recipe
    patches/           # applied patches per recipe
    src_uri/           # Yocto additional sources per recipe (init scripts, configs)
    copyrights/        # extracted copyright notices per recipe
    archives/          # per-recipe .tar.gz tarballs (copyleft)
    report.html        # interactive HTML report
    report.csv         # one row per source file per package
    report.xlsx        # Excel workbook (8 sheets, requires openpyxl)
```

All collections are deduplicated per recipe.

## How it works

**Source collection** — For each installed package:
- *Userspace*: extracts DWARF compilation-unit paths from ELF binaries
- *Kernel image*: finds `.o` files not owned by any module, collects corresponding sources
- *Kernel modules*: uses `.mod` files to enumerate objects, collects sources

**Verification** — Re-reads DWARF paths from `.debug` binaries and checks that every source file exists in the collected directory.

**Compile test** — Parses `log.do_compile`, replays each `gcc -c` command with the collected source copy, verifies it compiles.

**License obligations** — Classifies each recipe's license into: `source_distribution`, `object_linking`, `attribution`, `patent_grant`, `network_copyleft`, `permissive`.

**Linking analysis** — Resolves shared library dependencies via `readelf -d` and Yocto's `shlibs2` database, flags copyleft linking chains.

**Reports** — HTML (interactive, Chart.js), CSV, Excel (8 sheets: Summary, Packages, Source Files, Licenses, Patches, Yocto Additional Sources, Dependencies, Copyrights).

## Notes

- `.o` mismatch in compile tests is expected (`-fdebug-prefix-map` embeds paths in DWARF).
- `INCOMPLETE` status means some compile commands target uninstalled binaries — correct omissions.
- `bitbake -e` results are disk-cached to `output/.bb_cache/` for performance. Cleared by `--clean`.
