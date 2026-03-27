---
name: tensorrt-package-release
description: Updates the TensorRT recipe version, rebuilds Windows and WSL/Linux package matrices, verifies build results, and publishes successful artifacts to the local private channel. Use when the user asks to bump TensorRT, refresh source hashes, rerun package builds, inspect output packages, or copy finished artifacts into D:/PyEnv/channels/private.
---

# TensorRT Package Release

## Use this skill when

- Updating the upstream TensorRT version in this repo
- Rebuilding packages on Windows and/or WSL
- Checking finished build results
- Copying successful `.conda` artifacts to the local private channel

## Files to review first

- `recipe/recipe.yaml`
- `recipe/variants.yaml`
- `recipe/variants.win.yaml`
- `recipe/variants.linux.yaml`
- `recipe/conda-package-file-plan.md`

## Workflow

1. Update the recipe.
- Edit `context.version` in `recipe/recipe.yaml`.
- Reset `context.build` to `0` for a new upstream release unless the user asked for a rebuild-only change.
- Update every source URL and `sha256` for the Windows zip and Linux tarballs.
- If the upstream archive layout or package split changed, inspect the extracted sources or upstream metadata and update file lists, package dependencies, and tests before building.

2. Sanity-check the rendered graph when the recipe changed materially.
- Windows render:

```powershell
conda run -n base rattler-build build -r recipe/recipe.yaml -m recipe/variants.yaml -m recipe/variants.win.yaml -c conda-forge --render-only
```

- Linux render from the Windows host:

```powershell
wsl.exe bash -ic "cd /mnt/d/Python/conda-packages/tensorrt/recipe && conda run -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge --render-only"
```

- Use the rendered job list to confirm whether only the Python-bearing outputs vary by Python.

3. Clean only the target platform before each build or retry.
- Always inspect `recipe/output` via the terminal, not file-search tools, because it is ignored.
- Windows build: list and remove `recipe/output/win-64/*.conda`.
- Linux/WSL build: list and remove `recipe/output/linux-64/*.conda`.
- Do not delete the other platform unless the user explicitly asks.
- Windows cleanup example:

```powershell
Get-ChildItem -LiteralPath "D:/Python/conda-packages/tensorrt/recipe/output/win-64" -Filter *.conda
Get-ChildItem -LiteralPath "D:/Python/conda-packages/tensorrt/recipe/output/win-64" -Filter *.conda | Remove-Item -Force
```

- Linux/WSL cleanup example:

```powershell
Get-ChildItem -LiteralPath "D:/Python/conda-packages/tensorrt/recipe/output/linux-64" -Filter *.conda
Get-ChildItem -LiteralPath "D:/Python/conda-packages/tensorrt/recipe/output/linux-64" -Filter *.conda | Remove-Item -Force
```

- After cleanup, run:

```powershell
conda index output
```

4. Run the build.
- Windows:

```powershell
conda run --live-stream -n base rattler-build build -r recipe/recipe.yaml -m recipe/variants.yaml -m recipe/variants.win.yaml -c conda-forge
```

- Linux from the Windows host through WSL:

```powershell
wsl.exe bash -ic "cd /mnt/d/Python/conda-packages/tensorrt/recipe && conda run --live-stream -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge"
```

5. Monitor correctly.
- Before launching, check existing terminals to avoid duplicating a running build.
- After launch, do one quick terminal read for immediate failures.
- If startup looks healthy, stop polling and ask the user to report when the build ends or send the failure snippet.
- When the build ends, read the terminal footer and confirm `exit_code: 0`.

6. Evaluate the result.
- Treat these warnings as expected for this repo and do not fail the workflow on them alone:
  - `Exact MSVC version not found`: this repo repackages upstream binaries, so the important part is exporting the correct version requirement line.
  - overlinking warnings: this repo repackages upstream binaries instead of producing a fresh local link step.
- Investigate only if there is a real error, test failure, missing file, solver issue, or unexpected runtime requirement.
- After success, use the terminal to list the produced `.conda` files in the platform output directory and confirm the expected packages were written.
- Never publish anything from `recipe/output/broken`.

7. Publish successful artifacts to the local channel.
- Copy the successful files into the matching subdir under `D:/PyEnv/channels/private`.
- Typical Windows copy commands:

```powershell
Copy-Item "D:/Python/conda-packages/tensorrt/recipe/output/win-64/*.conda" "D:/PyEnv/channels/private/win-64/" -Force
Copy-Item "D:/Python/conda-packages/tensorrt/recipe/output/linux-64/*.conda" "D:/PyEnv/channels/private/linux-64/" -Force
```

- Then rebuild the private channel index:

```powershell
conda index D:/PyEnv/channels/private --zst
```

## Notes

- `recipe/variants.yaml` carries shared CUDA and Python variants.
- `recipe/variants.win.yaml` and `recipe/variants.linux.yaml` carry platform compiler and sysroot pins.
- The goal here is repackaging NVIDIA binaries, so version requirements matter more than matching a full local toolchain installation.
- If a build fails after writing `.conda` artifacts, clean the target platform output again and rerun `conda index output` before retrying.
