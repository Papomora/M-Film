# Verification — Phase 0 (gate G0, builds)

- Date: 2026-11-03
- M-Film: branch `port-200d`, commit `1b65324` (upstream HNGSeries/M-Film + kit files)
- Magic Lantern: magiclantern_simplified `ed3e7c0` (the commit pinned in `ml-dev/setup_wsl.sh`)
- Environment: Windows 11, WSL**1** Ubuntu 24.04 (`Ubuntu-MFilm`, stored on D:). WSL2 is not available: CPU virtualization is disabled in the BIOS. arm-none-eabi-gcc 13.2.1 (20231009), Python 3.12.3.
- Extra packages the scripts need beyond `setup_wsl.sh`: `python3-docutils` (ML module strings), `python3-numpy`, `python3-pil` (simulator, genfilms), `python3-segno` (build_fork QR codes; the script falls back to `pip`, which Ubuntu 24.04 does not ship).

| Item | Result | Numbers |
|---|---|---|
| Stock ML `200D.101` (`make FATAL_WARNINGS=y`, worktree `/root/ml/ml-stock`) | PASS | `build/magiclantern.zip` 1,546,407 B. Boot size `_bss_end - RESTARTSTART` = 0x51c700 - 0x4d0000 = **313,088 B**. No compiler errors/warnings; only make "overriding recipe" and newlib "stdio bloat" linker notes (pre-existing upstream). |
| M-Film `EOSM.202` (`ml-dev/build_fork.sh EOSM.202`) | PASS | exit 0. **720,416 B at boot (limit 727,072)**. Card files in `ml-dev/out/fork/`. 63/63 Film-Luts fetched. |
| Simulator (`make -C ml-dev/filmos/sim`) | PASS | builds with `-Wall -Wextra`, no warnings shown. |
| Previews (`previews.sh`) | PASS (baseline set) | 22 files, deterministic (a second run is byte-identical). Baseline copied to `/root/ml/phase0/baseline_previews/`. 10 PNGs differ from the ones committed upstream (last regenerated in `d960057`, before later code changes): this is upstream staleness, not a local change. Future regression checks compare against the local baseline. |

Notes
- `build_fork.sh` rewrites `ml-dev/fork/film-mode.patch` from the ML tree; with only one camera applied the file loses the other cameras' hunks. It was reverted with `git checkout`; never commit it from a single-camera build.
- The first `build_fork.sh` run failed (missing docutils) after `apply_fork.py` had patched the tree; the rerun reported "already applied" everywhere (idempotent, as expected).
