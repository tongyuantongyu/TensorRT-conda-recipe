# TensorRT Release Recovery

Use this file when a TensorRT build fails, only some outputs are bad, or already-published artifacts need repair.

## First triage

1. Confirm the failing platform and the exact failure from the terminal footer or last failing command.
2. Classify the problem before editing anything:
- Missing file or copy path failure: inspect extracted sources and build work dirs.
- Solver or dependency mismatch: audit `repodata.json`.
- Test failure after a `.conda` file was written: delete that package, reindex, and rebuild it.
- WSL command-line failure: simplify the Windows-host `wsl.exe bash -ic` command first.
3. Keep unrelated versions out of the target output root while repairing a specific version.

## Inspect the actual upstream layout

Read these locations when the failure suggests a path or naming mismatch.

Windows builds use `recipe/output`:

- `recipe/output/src_cache/*_extracted/`
- `recipe/output/src_cache/.metadata/*.json`
- `recipe/output/bld/<build>/work/`

WSL/Linux builds use `~/project/conda-packages`:

- `~/project/conda-packages/src_cache/*_extracted/`
- `~/project/conda-packages/src_cache/.metadata/*.json`
- `~/project/conda-packages/bld/<build>/work/`

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

1. Move unrelated finished versions out of the target output root or otherwise isolate them.
2. Delete only the broken or missing `.conda` artifacts for the failing platform.
3. Reindex the target output root.
4. Rerun the same platform build with `--skip-existing local`.
5. Verify the regenerated packages exist and audit repodata before publishing.

Windows example:

```powershell
$bad = @(
  "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64/libnvinfer-headers-python-plugin-dev-10.15.1.29-cuda129_0.conda",
  "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64/tensorrt-dev-10.15.1.29-cuda129_0.conda",
  "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output/win-64/tensorrt-10.15.1.29-cuda129_0.conda"
)
$bad | ForEach-Object { Remove-Item -LiteralPath $_ -Force }
mamba run -n base python -m conda_index "C:\Users\TYTY\Libraries\conda-packages\tensorrt/recipe/output"
mamba run --live-stream -n base rattler-build build -r recipe.yaml -m variants.yaml -m variants.win.yaml -c conda-forge --skip-existing local
```

Linux example from the Windows host through WSL:

```powershell
wsl.exe bash -lc 'rm -f "$HOME"/project/conda-packages/linux-64/libnvinfer-headers-python-plugin-dev-10.15.1.29-cuda129_0.conda "$HOME"/project/conda-packages/linux-64/tensorrt-dev-10.15.1.29-cuda129_0.conda "$HOME"/project/conda-packages/linux-64/tensorrt-10.15.1.29-cuda129_0.conda'
wsl.exe bash -ic "mamba run -n base python -m conda_index ~/project/conda-packages"
wsl.exe bash -ic "cd /mnt/c/Users/TYTY/Libraries/conda-packages/tensorrt/recipe/ && mamba run -n base rattler-build build --package-format conda:max --no-include-recipe -r recipe.yaml -m variants.yaml -m variants.linux.yaml -c conda-forge --output-dir ~/project/conda-packages --skip-existing local"
```

If both platforms need repair, rebuild Windows first and start Linux after Windows is clearly healthy.

## Repairing published packages

Use this when bad artifacts are already in `D:/PyEnv/channels/private`.

1. Move the target version back into the target output root or another repair staging area.
2. Delete only the bad packages for the affected platform.
3. Reindex the staging area.
4. Rebuild the missing outputs with `--skip-existing local`.
5. Audit repodata.
6. Move the repaired Windows and Linux packages back into the matching subdir under `D:/PyEnv/channels/private`.
7. Reindex both `D:/PyEnv/channels/private` and the affected staging/output root.

## Repodata audit checklist

Check all packages, not just the ones that failed visibly.

- If the package split still matches the current recipe, expect 72 `win-64` packages and 74 `linux-64` packages per TensorRT version.
- Parse both `packages` and `packages.conda` from `repodata.json`.
- For every package whose build string contains `cudaXYZ_0`, ensure dependencies on the same TensorRT version also use `cudaXYZ_0`.
- Metapackages and header or dev packages are often the first place a variant mismatch appears.

## Known failure patterns

- `copy` or `cp` cannot find files under `%SRC_DIR%` or `$TRT_LIB_DIR`: inspect the relevant `src_cache` under `recipe/output` for Windows or `~/project/conda-packages` for Linux, then update the recipe to match the real layout.
- `No license files were copied`: the upstream license filename or casing changed.
- `mamba run` under WSL errors on `--live-stream`: use `wsl.exe bash -ic` to run `mamba run -n base rattler-build` from the recipe dir without `--live-stream`.
- WSL package counts from PowerShell look wrong: use a simple single-quoted WSL command such as `wsl.exe bash -lc 'ls -1 "$HOME"/project/conda-packages/linux-64/*.conda 2>/dev/null | wc -l'`.
- Windows and Linux builds interfere in a shared `bld` directory: keep Windows in `recipe/output` and Linux in `~/project/conda-packages`, and do not overlap first-attempt builds against the same output root.
- A `.conda` file exists but tests failed afterward: delete that artifact, reindex, and rerun the target platform build.
