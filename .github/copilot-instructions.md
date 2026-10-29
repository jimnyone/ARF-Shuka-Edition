# Copilot instructions for ARF-Shuka-Edition

## Project shape

This repository is not a standalone app or library. It is a source overlay for a custom Flipper Zero firmware build based on D4C1-Labs/Flipper-ARF. The authoritative build flow is: clone the upstream ARF tag from `ARF_VERSION`, apply the files in `modified_files/`, add protocol helpers from `new_files/`, patch the checkout via `scripts/*`, then run the Flipper build tool (`./fbt`).

Important directories:
- `new_files/`: custom and added protocol implementations/helpers (for example automotive sub-GHz decoders/encoders and crypto support).
- `modified_files/`: upstream file replacements that are applied on top of the ARF checkout during CI/release builds.
- `scripts/`: build-time patching, API consistency checks, and release helpers.
- `tests/`: standalone C vector tests for crypto helpers and rolling-code logic.
- `docs/`: release metadata and static website assets.
- `firmware/`: generated firmware artifacts; these are build outputs, not the source of truth.
- `extra_apps/`: extra app content that is not automatically discovered by `fbt` without a full upstream firmware checkout.

The build/release workflow is defined in `.github/workflows/build.yml`; the project README also documents the intended source build procedure.

## Build, test, and validation commands

There is no repo-local package manager or generic lint target here. This project is validated with C test vectors and the firmware build itself.

### Standalone crypto/rolling-code tests

Run these from the repo root:

```bash
cd tests

gcc -I.. -O2 -Wall -Werror -o test_aut64 test_aut64.c ../new_files/aut64.c
./test_aut64

gcc -I.. -O2 -Wall -Werror -o test_keeloq test_keeloq.c
./test_keeloq
```

These are the exact checks used by CI and are the fastest way to validate a focused change in the custom crypto helpers or rolling-code logic.

### Full firmware build

This repository expects the upstream ARF source to be cloned locally and patched in place. The CI build does this automatically:

```bash
git clone --depth 1 --branch "$(cat ARF_VERSION)" \
  https://github.com/D4C1-Labs/Flipper-ARF.git flipper-fw
cd flipper-fw
git submodule update --init --recursive --depth 1

# overlay the repo's customizations here
# then run the firmware build
./fbt COMPACT=1 DEBUG=0 DIST_SUFFIX="arf-shuka-edition" updater_package
```

This is the build target documented in the README and used by the GitHub workflow. If you are changing protocol implementations or added files, validate with the standalone tests first and then do a full `./fbt` build in the ARF checkout when practical.

### Size / packaging sanity checks

The repo includes a repo-specific size gate:

```bash
bash scripts/check_size.sh path/to/firmware.dfu
```

This is a quick check for the STM32WB flash boundary and is relevant when adding new protocols/apps because the project is tight on space.

## High-level architecture

The important architecture is the upstream ARF overlay model:

1. The base firmware comes from D4C1-Labs/Flipper-ARF.
2. `new_files/` adds custom protocol logic and helper files that are not part of upstream ARF.
3. `modified_files/` mirrors upstream file paths exactly and replaces the upstream sources when the CI workflow applies the overlay.
4. `scripts/` then patches the cloned firmware tree to add branding, menu entries, compatibility fixes, and release metadata.
5. The output is a Flipper firmware package (`.dfu` / `.tgz`) sized for a specific hardware target, with a strict flash boundary.

This means that source changes often need to be understood in the context of the ARF checkout rather than in isolation. A single file in this repo rarely tells the whole story; the real build behavior depends on how the upstream tree is patched and packaged.

## Key conventions

- Keep file paths in `modified_files/` aligned to the upstream ARF tree layout. These files are overlaid by path, not by symbol or project name.
- New protocol support belongs in `new_files/` and usually also needs registration in the upstream protocol registry (`protocol_items.c` / `protocol_items.h`) inside the ARF clone.
- Scripted patches are part of the build process; prefer updating the patch scripts or overlay files instead of hand-editing generated release artifacts.
- This project is highly sensitive to firmware-size constraints. Protocol additions and app additions can easily push the build over the STM32WB region boundary, so keep size impact in mind during changes.
- The project’s branding and compatibility patches are intentionally custom and should be kept consistent with the Shuka-ARF naming used in the README and CI build.
- `README.md`, `CHANGELOG.md`, and the build workflow are the reliable source for product intent. Follow the established naming, release layout, and custom firmware conventions rather than inventing a new repo structure.

## Practical guidance for future Copilot sessions

- Look first at `.github/workflows/build.yml` and `README.md` when validating a change or figuring out how this repo is assembled.
- Treat `ARF_VERSION` as the build anchor for the upstream dependency.
- When a fix relates to a protocol or firmware feature, inspect both the corresponding file under `new_files/` and the matching file under `modified_files/` if one exists.
- Prefer focused validation (crypto tests, then targeted firmware build) over broad repo-wide commands; there is no broad test suite for a normal app project here.
