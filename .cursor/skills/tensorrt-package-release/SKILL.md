---
name: tensorrt-package-release
description: Update TensorRT recipe versions, rebuild Windows and WSL/Linux package matrices, verify package outputs, and publish successful artifacts to the local private channel. Use when bumping TensorRT, refreshing source hashes, rerendering the build graph, checking repodata, or publishing repaired packages.
---

# TensorRT Package Release

## Use this skill when

- Updating the upstream TensorRT version in this repo
- Refreshing Windows zip and Linux tarball hashes
- Rebuilding packages on Windows and/or WSL
- Checking finished build results or repodata
- Publishing successful `.conda` artifacts to the local private channel

## Files to review first

- `recipe/recipe.yaml`
- `recipe/variants.yaml`
- `recipe/variants.win.yaml`
- `recipe/variants.linux.yaml`

## Smooth update workflow

1. Update recipe basics.
- Edit `context.version` in `recipe/recipe.yaml`.
- Reset `context.build` to `0` for a new upstream release unless the user asked for a rebuild-only change.
- Update every source URL and `sha256` for the Windows zip and Linux tarballs.
- Update `recipe/variants.yaml` if the supported CUDA matrix changed.

2. Start with the happy-path assumption.
- It is fine to render or build once assuming the upstream archive layout is unchanged.
- If the first failure shows missing files, wrong archive names, or copy/test path errors, inspect the relevant `src_cache` under `recipe/output` for Windows or `~/project/conda-packages` for Linux, then patch the recipe to match the extracted upstream layout.
- When checking extracted sources, verify archive naming, DLL and SO locations, builder-resource filenames, Python header paths, and license-file casing.

3. Sanity-check the rendered graph when the recipe changed materially.
- Windows render:

```powershell
mamba run -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.win.yaml -c conda-forge --render-only
```

- Linux render from the Windows host:

```powershell
wsl.exe bash -ic "cd /mnt/c/Users/TYTY/Libraries/conda-packages/tensorrt/recipe/ && rattler-build build -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge --output-dir ~/project/conda-packages --render-only"
```

- Use the rendered job list to confirm whether only the Python-bearing outputs vary by Python.

4. Keep the staging area clean.
- Treat output roots as one-version-at-a-time staging spaces.
- Windows builds use `recipe/output`; WSL/Linux builds use the native WSL output root `~/project/conda-packages` for build intermediates and results.
- Move finished versions aside before starting a different version update so repodata and `--skip-existing local` do not get confused.
- Always inspect output roots via the terminal, not file-search tools, because they may be ignored or outside the repo.
- Before a full rebuild, clean only the target platform output directory.
- Do not delete the other platform unless the user explicitly asks.
- Windows cleanup example:

```powershell
Get-ChildItem -LiteralPath "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64" -Filter *.conda
Get-ChildItem -LiteralPath "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64" -Filter *.conda | Remove-Item -Force
```

- Linux/WSL cleanup example from the Windows host:

```powershell
wsl.exe bash -ic "ls -1 ~/project/conda-packages/linux-64/*.conda"
wsl.exe bash -ic "rm -f ~/project/conda-packages/linux-64/*.conda"
```

- After Windows cleanup, run:

```powershell
mamba run -n base python -m conda_index "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output"
```

- After Linux/WSL cleanup, run from the Windows host:

```powershell
wsl.exe bash -ic "python -m conda_index ~/project/conda-packages"
```

5. Run the build in safe order.
- Before launching, check existing terminals to avoid duplicating a running build.
- Start Windows first. Once it is clearly healthy or has produced the first package, start the Linux build.
- Avoid overlapping first attempts for both platforms against the same output root.
- Windows:

```powershell
mamba run --live-stream -n base rattler-build build --package-format conda:max --no-include-recipe -r recipe.yaml -m variants.yaml -m variants.win.yaml -c conda-forge
```

- Linux from the Windows host through WSL:

```powershell
wsl.exe bash -ic "cd /mnt/c/Users/TYTY/Libraries/conda-packages/tensorrt/recipe/ && mamba run -n base rattler-build build --package-format conda:max --no-include-recipe -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge --output-dir ~/project/conda-packages"
```

6. Monitor correctly.
- After launch, do one quick terminal read for immediate failures.
- If startup looks healthy, stop polling and ask the user to report when the build ends or send the failure snippet.
- When the build ends, read the terminal footer and confirm `exit_code: 0`.
- Use `mamba run --live-stream` for better Windows-side output.
- Do not pass `--live-stream` to the WSL/Linux build command.

7. Evaluate the result.
- Treat these warnings as expected for this repo and do not fail the workflow on them alone:
  - `Exact MSVC version not found`: this repo repackages upstream binaries, so the important part is exporting the correct version requirement line.
  - overlinking warnings: this repo repackages upstream binaries instead of producing a fresh local link step.
- Investigate only if there is a real error, test failure, missing file, solver issue, or unexpected runtime requirement.
- After success, use the terminal to list the produced `.conda` files in the platform output directory and confirm the expected packages were written.
- Windows output is under `recipe/output/win-64`; Linux output is under `~/project/conda-packages/linux-64`.
- If the package split still matches the current recipe, expect 72 `win-64` packages and 74 `linux-64` packages per TensorRT version.
- Audit `repodata.json` before publish and confirm no package with build string like `cuda129_0` depends on the same TensorRT version with a different CUDA build string.
- When reading repodata, inspect both `packages` and `packages.conda`.
- Never publish anything from `recipe/output/broken`.

8. Publish successful artifacts to the local channel.
- Copy the successful Windows and Linux files into the matching subdir under `D:/PyEnv/channels/private`.
- Typical Windows copy commands:

```powershell
Copy-Item "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64/*.conda" "D:/PyEnv/channels/private/win-64/" -Force
```

- Typical Linux copy command from the Windows host through WSL:

```powershell
wsl.exe bash -ic "cp -f ~/project/conda-packages/linux-64/*.conda /mnt/d/PyEnv/channels/private/linux-64/"
```

- Then rebuild the private channel index:

```powershell
mamba run -n base python -m conda_index "D:/PyEnv/channels/private" --zst
```

9. Reindex the staging area after moving packages out.
- If you moved packages out of `recipe/output` or `~/project/conda-packages`, reindex the affected output root too so stale repodata does not interfere with later rebuilds.

## Recovery

- For partial rebuilds, repairing bad published packages, source-layout surprises, and failure triage, see [RECOVERY.md](RECOVERY.md).

## Notes

- `recipe/variants.yaml` carries shared CUDA and Python variants.
- `recipe/variants.win.yaml` and `recipe/variants.linux.yaml` carry platform compiler and sysroot pins.
- The goal here is repackaging NVIDIA binaries, so version requirements matter more than matching a full local toolchain installation.
