# lucky-dog-release

Public release pipeline for the (private) [lucky-dog](https://github.com/) Electron app.
This repo exists for one reason: **GitHub Actions minutes are free and unlimited
on public repos**, but capped on free private repos. By splitting packaging out
into a public repo that pulls source from the private one via a PAT, we get
unlimited CI minutes for the cross-platform Electron builds without exposing
source code.

## Setup (one-time)

1. **Create this repo on GitHub as public**, e.g. `<owner>/lucky-dog-release`,
   and push this directory to it.

2. **Generate a fine-grained Personal Access Token** for reading the private
   source repo:
   - https://github.com/settings/personal-access-tokens/new
   - Resource owner: the account/org that owns `lucky-dog`
   - Repository access: Only select repositories → `lucky-dog`
   - Repository permissions: **Contents → Read-only**
   - Expiration: pick something reasonable (90d / 1y); set a calendar reminder
     to rotate
   - Copy the generated token

3. **In `lucky-dog-release` repo settings → Secrets and variables → Actions**:
   - Tab **Secrets** → New repository secret
     - Name: `SOURCE_REPO_TOKEN`
     - Value: the PAT from step 2
   - Tab **Variables** → New repository variable
     - Name: `SOURCE_REPO`
     - Value: `<owner>/lucky-dog` (the private repo's `owner/name`)

4. **Confirm in `lucky-dog-release` repo settings → Actions → General**:
   - Workflow permissions: Read and write permissions
   - Allow GitHub Actions to create and approve pull requests: not required

## Running a build

1. Go to **Actions → Release → Run workflow**
2. Fill in the inputs:
   - **ref**: source branch/tag/sha to build (default `main`)
   - **release_tag**: leave blank to only upload workflow artifacts; set to
     `v0.1.0` (or similar) to also publish a GitHub Release with the binaries
     attached to this repo
   - **draft**: keep `true` while testing — gives you a chance to review before
     making the release public
3. Wait for the 4 matrix builds (mac arm64, mac x64, win x64, linux x64) to
   finish. Roughly 8–12 minutes each, parallel.

## What the workflow does

```
┌─ checkout lucky-dog-release (this repo)
└─ checkout <owner>/lucky-dog @ inputs.ref  (via SOURCE_REPO_TOKEN)
       │
       └─ pnpm install --frozen-lockfile
       └─ pnpm run build           # turbo: main bundle + renderer static export
       └─ pnpm run make            # electron-builder
       └─ upload-artifact          # out/*.dmg etc., 30-day retention

if inputs.release_tag is set:
   └─ download-artifact (all 4 matrix slices)
   └─ flatten + softprops/action-gh-release → publishes to this repo's Releases
```

## Why not just run Actions on `lucky-dog` directly

The free private-repo tier gives 2,000 Actions minutes/month total across all
private repos in the account; cross-platform Electron packaging (esp. with
macOS notarization) burns minutes fast. Public repos have **no minute cap** at
all. Keeping the build pipeline public while source stays private is a common
pattern for indie / hobby Electron apps that want unlimited CI without paying
for Team/Enterprise.

## Caveats

- macOS code signing & notarization are not wired up here. Add `CSC_LINK`,
  `CSC_KEY_PASSWORD`, `APPLE_ID`, `APPLE_APP_SPECIFIC_PASSWORD`,
  `APPLE_TEAM_ID` secrets and pass them through `env:` of the Make step if you
  want signed/notarized macOS builds. Without signing, downloaded `.dmg` files
  will trigger Gatekeeper warnings.
- The PAT is the single trust boundary — if leaked, an attacker can read the
  private repo. Use fine-grained PATs (not classic), scope to the one repo,
  set short expirations, and rotate.
- `pnpm install --frozen-lockfile` requires the source repo to commit
  `pnpm-lock.yaml`. If the lockfile is missing or out of sync, the workflow
  will fail loudly — that's intentional, prevents accidental dependency drift
  between local dev and the released binary.
