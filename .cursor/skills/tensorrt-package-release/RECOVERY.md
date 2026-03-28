# TensorRT Release Recovery

Use this file when a TensorRT build fails, only some outputs are bad, or already-published artifacts need repair.

## First triage

1. Confirm the failing platform and the exact failure from the terminal footer or last failing command.
2. Classify the problem before editing anything:
- Missing file or copy path failure: inspect extracted sources and build work dirs.
- Solver or dependency mismatch: audit `repodata.json`.
- Test failure after a `.conda` file was written: delete that package, reindex, and rebuild it.
- WSL command-line failure: simplify the WSL command first.
3. Keep unrelated versions out of `recipe/output` while repairing a specific version.

## Inspect the actual upstream layout

Read these locations when the failure suggests a path or naming mismatch:

- `recipe/output/src_cache/*_extracted/`
- `recipe/output/src_cache/.metadata/*.json`
- `recipe/output/bld/<build>/work/`

Use them to compare the real archive contents against:

- `source` URLs and filenames
- build-script copy commands
- `files` lists
- tests
- license file paths

Common layout changes seen in practice:

- Windows archive names or download hostnames changed
- DLLs moved between `bin/` and `lib/`
- builder-resource files were renamed or collapsed
- Python plugin headers moved
- `README` filename casing changed

## Partial rebuild workflow

Use this when only a few packages are bad and most of the target version is already correct.

1. Move unrelated finished versions out of `recipe/output` or otherwise isolate them.
2. Delete only the broken or missing `.conda` artifacts for the failing platform.
3. Reindex `recipe/output`.
4. Rerun the same platform build with `--skip-existing local`.
5. Verify the regenerated packages exist and audit repodata before publishing.

Windows example:

```powershell
$bad = @(
  "D:/Python/conda-packages/tensorrt/recipe/output/win-64/libnvinfer-headers-python-plugin-dev-10.15.1.29-cuda129_0.conda",
  "D:/Python/conda-packages/tensorrt/recipe/output/win-64/tensorrt-dev-10.15.1.29-cuda129_0.conda",
  "D:/Python/conda-packages/tensorrt/recipe/output/win-64/tensorrt-10.15.1.29-cuda129_0.conda"
)
$bad | ForEach-Object { Remove-Item -LiteralPath $_ -Force }
mamba run -n base python -m conda_index "D:/Python/conda-packages/tensorrt/recipe/output"
mamba run --live-stream -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.win.yaml -c conda-forge --skip-existing local
```

Linux example from the Windows host:

```powershell
$bad = @(
  "D:/Python/conda-packages/tensorrt/recipe/output/linux-64/libnvinfer-headers-python-plugin-dev-10.15.1.29-cuda129_0.conda",
  "D:/Python/conda-packages/tensorrt/recipe/output/linux-64/tensorrt-dev-10.15.1.29-cuda129_0.conda",
  "D:/Python/conda-packages/tensorrt/recipe/output/linux-64/tensorrt-10.15.1.29-cuda129_0.conda"
)
$bad | ForEach-Object { Remove-Item -LiteralPath $_ -Force }
mamba run -n base python -m conda_index "D:/Python/conda-packages/tensorrt/recipe/output"
wsl.exe bash -ic "cd /mnt/d/Python/conda-packages/tensorrt/recipe && mamba run -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge --skip-existing local"
```

If both platforms need repair, rebuild Windows first and start Linux after Windows is clearly healthy.

## Repairing published packages

Use this when bad artifacts are already in `D:/PyEnv/channels/private`.

1. Move the target version back into `recipe/output` or another repair staging area.
2. Delete only the bad packages for the affected platform.
3. Reindex the staging area.
4. Rebuild the missing outputs with `--skip-existing local`.
5. Audit repodata.
6. Move the repaired version back into `D:/PyEnv/channels/private`.
7. Reindex both `D:/PyEnv/channels/private` and `recipe/output`.

## Repodata audit checklist

Check all packages, not just the ones that failed visibly.

- If the package split still matches the current recipe, expect 72 `win-64` packages and 74 `linux-64` packages per TensorRT version.
- Parse both `packages` and `packages.conda` from `repodata.json`.
- For every package whose build string contains `cudaXYZ_0`, ensure dependencies on the same TensorRT version also use `cudaXYZ_0`.
- Metapackages and header or dev packages are often the first place a variant mismatch appears.

## Known failure patterns

- `copy` or `cp` cannot find files under `%SRC_DIR%` or `$TRT_LIB_DIR`: inspect `recipe/output/src_cache` and update the recipe to match the real layout.
- `No license files were copied`: the upstream license filename or casing changed.
- `mamba run` under WSL errors on `--live-stream`: remove that flag from the WSL command.
- Windows and Linux builds interfere in `recipe/output/bld`: do not overlap first-attempt builds for both platforms.
- A `.conda` file exists but tests failed afterward: delete that artifact, reindex, and rerun the target platform build.
