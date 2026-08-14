# PR 479 Fork Pre-release Design

## Goal

Publish a clearly unofficial Windows pre-release on `jshnaidman/BG3ModManager` containing the Script Extender detection fix from upstream PR 479, without modifying the PR branch or implying that the package is an official BG3 Mod Manager release.

## Release identity

- Repository: `jshnaidman/BG3ModManager`
- Source commit: `9d80ed303911b6544d276581c0f42e57cefa4b92`
- Tag and release name: `pr-479-wine-fix-1`
- GitHub release state: pre-release
- Asset name: `BG3ModManager_v1.0.12.9-pr479-wine-fix.zip`
- Product assembly version: unchanged at `1.0.12.9`

The release notes must call the package an unofficial fork build, link upstream PR 479, identify the exact source commit, report validation results, and include the asset's SHA-256 checksum.

## Build architecture

A temporary release branch will contain only release-support files added on top of the PR commit. A GitHub Actions workflow running on `windows-latest` will explicitly check out source commit `9d80ed303911b6544d276581c0f42e57cefa4b92`, including recursive submodules. Pinning the checkout separates the binary input from the workflow commit and ensures the package contains the approved PR code only.

The workflow will install .NET 8, restore dependencies, run the focused regression test project in Release configuration, and invoke the solution's existing `Publish|x64` configuration. The existing `BuildRelease.py` target will create the release ZIP from `bin/Publish`. The workflow will rename the ZIP to the distinct fork-release asset name and upload it as a workflow artifact.

## Verification and publication

Publication will occur only after all of the following evidence is available:

1. The workflow completed successfully on a GitHub-hosted Windows runner.
2. The regression test job reports zero failures.
3. The Publish build exits successfully and produces the expected ZIP.
4. The downloaded ZIP opens successfully and contains `BG3ModManager.exe` plus its runtime files.
5. A SHA-256 checksum is calculated from the exact ZIP that will be uploaded.

After verification, create tag `pr-479-wine-fix-1` targeting the exact PR source commit and create a GitHub pre-release with the verified ZIP attached. Fetch the published release afterward and compare its tag, pre-release flag, target commit, asset name, asset size, and checksum with the local verified values.

## Failure handling and cleanup

If tests, compilation, packaging, ZIP inspection, upload, or post-publication verification fails, do not advertise the release as usable. Preserve the workflow logs and report the exact failure. If publication partially succeeds with a missing or mismatched asset, repair the release before announcing completion.

The PR branch `agent/fix-wine-script-extender-detection` will remain unchanged. The release-support branch may remain in the fork to preserve build provenance; it will not be merged into the PR unless separately requested.
