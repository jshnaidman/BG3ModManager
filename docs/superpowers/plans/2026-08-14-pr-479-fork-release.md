# PR 479 Fork Pre-release Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build, verify, and publish an unofficial `pr-479-wine-fix-1` Windows pre-release on `jshnaidman/BG3ModManager` from the exact source commit in upstream PR 479.

**Architecture:** A release-support branch adds a push-triggered GitHub Actions workflow, while the workflow itself checks out the immutable PR commit before testing and publishing. The resulting ZIP is downloaded, inspected, hashed, and then attached to a pre-release whose tag targets that same PR commit.

**Tech Stack:** GitHub Actions, Windows Server runner, .NET 8 SDK, PowerShell, Python 3, GitHub CLI

## Global Constraints

- Do not modify `agent/fix-wine-script-extender-detection`.
- Build source commit `9d80ed303911b6544d276581c0f42e57cefa4b92` exactly.
- Publish only to `jshnaidman/BG3ModManager`.
- Use tag and release name `pr-479-wine-fix-1` and mark the release as a pre-release.
- Keep the product assembly version at `1.0.12.9`.
- Upload the asset as `BG3ModManager_v1.0.12.9-pr479-wine-fix.zip`.
- Do not publish unless tests, build, ZIP inspection, and checksum generation succeed.

---

### Task 1: Add and run the reproducible Windows build workflow

**Files:**
- Create: `.github/workflows/build-pr-479-fork-release.yml`

**Interfaces:**
- Consumes: PR source commit `9d80ed303911b6544d276581c0f42e57cefa4b92`, recursive public Git submodules, and the existing solution `Publish|x64` configuration.
- Produces: GitHub Actions artifact `pr-479-wine-fix-package` containing `BG3ModManager_v1.0.12.9-pr479-wine-fix.zip`.

- [ ] **Step 1: Create the release workflow**

```yaml
name: Build PR 479 fork release

on:
  push:
    branches:
      - release/pr-479-wine-fix-1

permissions:
  contents: read

jobs:
  build:
    runs-on: windows-latest
    steps:
      - name: Check out PR source commit
        uses: actions/checkout@v4
        with:
          ref: 9d80ed303911b6544d276581c0f42e57cefa4b92
          submodules: recursive

      - name: Set up .NET 8
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 8.0.x

      - name: Run regression tests
        run: dotnet test tests/BG3ModManager.Tests/BG3ModManager.Tests.csproj --configuration Release

      - name: Build release package
        run: dotnet build BG3ModManager.sln --configuration Publish -p:Platform=x64

      - name: Validate and rename package
        shell: pwsh
        run: |
          $source = "BG3ModManager_v1.0.12.9.zip"
          $destination = "BG3ModManager_v1.0.12.9-pr479-wine-fix.zip"
          if (-not (Test-Path $source)) { throw "Expected package not found: $source" }
          Move-Item $source $destination
          if ((Get-Item $destination).Length -le 0) { throw "Release package is empty" }

      - name: Upload package
        uses: actions/upload-artifact@v4
        with:
          name: pr-479-wine-fix-package
          path: BG3ModManager_v1.0.12.9-pr479-wine-fix.zip
          if-no-files-found: error
          retention-days: 30
```

- [ ] **Step 2: Validate the workflow file and branch diff**

Run:

```bash
git diff --check
git diff -- .github/workflows/build-pr-479-fork-release.yml
git diff 9d80ed303911b6544d276581c0f42e57cefa4b92 -- src tests
```

Expected: no whitespace errors; the workflow matches Step 1; the final command produces no output because there are no additional changes under `src` or `tests`.

- [ ] **Step 3: Commit the workflow**

```bash
git add .github/workflows/build-pr-479-fork-release.yml
git commit -m "ci: build PR 479 fork release"
```

- [ ] **Step 4: Push the isolated release-support branch**

```bash
git push --set-upstream fork release/pr-479-wine-fix-1
```

Expected: the push succeeds and starts `Build PR 479 fork release` in `jshnaidman/BG3ModManager`.

- [ ] **Step 5: Wait for the workflow and inspect its evidence**

```bash
run_id="$(gh run list --repo jshnaidman/BG3ModManager --branch release/pr-479-wine-fix-1 --workflow "Build PR 479 fork release" --limit 1 --json databaseId --jq '.[0].databaseId')"
gh run list --repo jshnaidman/BG3ModManager --branch release/pr-479-wine-fix-1 --workflow "Build PR 479 fork release" --limit 1
gh run watch "$run_id" --repo jshnaidman/BG3ModManager --exit-status
gh run view "$run_id" --repo jshnaidman/BG3ModManager --log
```

Expected: the workflow, regression test, Publish build, package validation, and artifact upload all succeed. If any step fails, inspect the complete log and fix only the release-support workflow; do not publish a release.

### Task 2: Verify the artifact and publish the pre-release

**Files:**
- Create locally: `/tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip`
- Create locally: `/tmp/bg3mm-pr479-release/SHA256SUMS.txt`

**Interfaces:**
- Consumes: successful workflow run ID and artifact `pr-479-wine-fix-package` from Task 1.
- Produces: verified GitHub pre-release `pr-479-wine-fix-1` with one Windows ZIP asset and a documented SHA-256 checksum.

- [ ] **Step 1: Download the exact successful-run artifact into a fresh temporary directory**

```bash
mkdir -p /tmp/bg3mm-pr479-release
run_id="$(gh run list --repo jshnaidman/BG3ModManager --branch release/pr-479-wine-fix-1 --workflow "Build PR 479 fork release" --limit 1 --json databaseId --jq '.[0].databaseId')"
gh run download "$run_id" --repo jshnaidman/BG3ModManager --name pr-479-wine-fix-package --dir /tmp/bg3mm-pr479-release
```

Expected: exactly one ZIP exists at `/tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip`.

- [ ] **Step 2: Inspect the ZIP and calculate its checksum**

```bash
unzip -t /tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip
unzip -l /tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip
sha256sum /tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip | tee /tmp/bg3mm-pr479-release/SHA256SUMS.txt
```

Expected: archive test exits zero; the listing contains `BG3ModManager.exe`, `BG3ModManager.dll`, and `_Lib/`; the checksum file contains exactly one entry for the release ZIP.

- [ ] **Step 3: Verify the tag and release do not already exist**

```bash
git ls-remote --tags fork refs/tags/pr-479-wine-fix-1
gh release view pr-479-wine-fix-1 --repo jshnaidman/BG3ModManager
```

Expected: neither a remote tag nor a release exists. If either exists, stop rather than overwrite it.

- [ ] **Step 4: Create the tag at the exact PR source commit**

```bash
git tag -a pr-479-wine-fix-1 9d80ed303911b6544d276581c0f42e57cefa4b92 -m "Unofficial PR 479 Wine fix build"
git push fork refs/tags/pr-479-wine-fix-1
```

Expected: the remote annotated tag resolves through its peeled target to `9d80ed303911b6544d276581c0f42e57cefa4b92`.

- [ ] **Step 5: Assemble concrete release notes from verified results**

Read the workflow URL and checksum into shell variables, then assemble the final notes in memory:

```bash
workflow_url="$(gh run view "$run_id" --repo jshnaidman/BG3ModManager --json url --jq .url)"
checksum="$(cut -d ' ' -f1 /tmp/bg3mm-pr479-release/SHA256SUMS.txt)"
release_notes=$'## Unofficial fork build\n\nThis is an unofficial Windows build of BG3 Mod Manager containing the Script Extender detection fix from [upstream PR #479](https://github.com/LaughingLeader/BG3ModManager/pull/479). It is not an official LaughingLeader release.\n\n- Source commit: `9d80ed303911b6544d276581c0f42e57cefa4b92`\n- Product version: `1.0.12.9`\n- Validation: 3 regression tests passed, 0 failed; Publish build succeeded\n'
release_notes+="- Build workflow: ${workflow_url}"$'\n'
release_notes+="- SHA-256: \`${checksum}\`"$'\n\nThe fix prevents a recurring `NullReferenceException` under Wine when `DWrite.dll` has no readable PE `ProductName`, while retaining conservative Script Extender updater detection.'
```

Expected: `workflow_url` identifies the successful run, `checksum` is a 64-character lowercase hexadecimal value, and `release_notes` contains no unresolved tokens.

- [ ] **Step 6: Create the pre-release and attach the verified ZIP**

```bash
gh release create pr-479-wine-fix-1 \
  /tmp/bg3mm-pr479-release/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip \
  --repo jshnaidman/BG3ModManager \
  --title "pr-479-wine-fix-1" \
  --notes "$release_notes" \
  --prerelease \
  --verify-tag
```

Expected: GitHub returns the new release URL and lists one ZIP asset.

- [ ] **Step 7: Verify the published release from GitHub**

```bash
gh release view pr-479-wine-fix-1 --repo jshnaidman/BG3ModManager --json url,isPrerelease,tagName,targetCommitish,assets
git ls-remote --tags fork refs/tags/pr-479-wine-fix-1 refs/tags/pr-479-wine-fix-1^{}
```

Download the published asset into a second fresh directory and compare it with the verified artifact:

```bash
mkdir -p /tmp/bg3mm-pr479-published
gh release download pr-479-wine-fix-1 --repo jshnaidman/BG3ModManager --pattern 'BG3ModManager_v1.0.12.9-pr479-wine-fix.zip' --dir /tmp/bg3mm-pr479-published
sha256sum /tmp/bg3mm-pr479-published/BG3ModManager_v1.0.12.9-pr479-wine-fix.zip
```

Expected: `isPrerelease` is true; the tag peels to the exact PR source commit; the asset name and size match; and the downloaded published asset has the same SHA-256 as the workflow artifact.

- [ ] **Step 8: Report the release and retained provenance**

Report the GitHub release URL, workflow URL, source commit, test count, asset name, byte size, SHA-256, and that `agent/fix-wine-script-extender-detection` remained unchanged. Retain `release/pr-479-wine-fix-1` as build provenance unless the user separately asks to remove it.
