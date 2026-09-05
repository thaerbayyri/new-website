# Instructions: Releasing a new app version (for AI agents)

This directory holds **update manifests** that Android/Windows apps poll to check
for new versions. The actual binaries (APK/EXE) are **not** stored in this repo —
they live as assets on a GitHub Release, and each manifest's URL field points there.

- Repo: `thaerbayyri/new-website`
- Release tag: `app-updates-v1`
- Release asset base URL: `https://github.com/thaerbayyri/new-website/releases/download/app-updates-v1/`

Never point a manifest URL at `thaerbayyari.com`, `thaerbayyari.online`, or any other
domain path — those paths are not served (the binaries were removed from the git
working tree specifically to keep this Vercel-deployed repo small). The **only**
valid source for a download URL is a GitHub Release asset link.

## File map

| Manifest | App / platform | Schema | Release asset filename |
|---|---|---|---|
| `update.json` | Bayyari TV — phone build | Android schema | `bayyari-tv.apk` |
| `update-tv.json` | Bayyari TV — Android TV build | Android schema | `bayyari-tv.apk` ⚠️ see note below |
| `nashama-update.json` | NashamaTV — phone build | Android schema | `NashamaTV.app.apk` |
| `nashamatv/nashama-tv-update.json` | NashamaTV — Android TV build | Android schema | `NashamaTV.TV.apk` |
| `windows-app/nashama-update.json` | Nashama TV — Windows installer | Windows schema | `Nashama.TV.Setup.1.2.4.exe` (versioned name — see policy below) |
| `linux-app/nashama-update.json` | Nashama TV — Linux AppImage | Windows schema (no versionCode/apkSha256) | `NashamaTV-linux.AppImage` (stable name, `--clobber` on every release) |

⚠️ **Known inconsistency to resolve, don't silently perpetuate it**: `update.json`
(versionCode 27) and `update-tv.json` (versionCode 28) currently both point at the
identical `bayyari-tv.apk` asset. That can only be correct if phone and Android TV
share one build. If they are actually different binaries, the next release must
upload them as two distinct assets (e.g. `bayyari-tv.apk` and `bayyari-tv-tv.apk`)
and each manifest must reference its own file. If they really are the same build,
align both files to the same `versionCode`/`versionName` on every release. Do not
proceed with a release that leaves these two files claiming different version
numbers for the same asset — ask the human which case applies if unclear.

## JSON schemas

**Android schema** (`update.json`, `update-tv.json`, `nashama-update.json`,
`nashamatv/nashama-tv-update.json`):

```json
{
  "versionCode": 28,
  "versionName": "1.2.4",
  "apkUrl": "https://github.com/thaerbayyri/new-website/releases/download/app-updates-v1/<asset-filename>",
  "apkSha256": "<lowercase or uppercase hex sha256 of the exact uploaded apk>",
  "releaseNotes": "short human-readable summary of what changed"
}
```

- `versionCode` is an integer and **must strictly increase** from the previous value
  in that same file — never equal or decrease it.
- `versionName` is the human-facing semver-ish string (e.g. `"1.2.4"`), independent
  of versionCode but should also move forward.
- `apkSha256` must be the hash of the file that was actually uploaded to the
  release, not a placeholder — verify with the file, not by guessing.

**Windows schema** (`windows-app/nashama-update.json`):

```json
{
  "version": "1.2.4",
  "url": "https://github.com/thaerbayyri/new-website/releases/download/app-updates-v1/<asset-filename>",
  "notes": "short human-readable summary of what changed"
}
```

No `versionCode` or `apkSha256` field exists in this schema — don't add one.

## Asset naming policy

GitHub Release asset filenames get spaces converted to dots automatically on
upload (e.g. `"NashamaTV app.apk"` → `NashamaTV.app.apk`). Two supported patterns
exist in this project; use the one already established for that file:

1. **Stable filename, `--clobber` overwrite** (used for all APKs): keep uploading
   under the exact same asset name every release, so the manifest's URL field
   *never changes* — only the version fields and hash change. This is the
   preferred pattern going forward for every platform including Windows, to
   minimize edits.
2. **Versioned filename** (legacy pattern currently on the Windows `.exe` asset,
   e.g. `Nashama.TV.Setup.1.2.4.exe`): each release gets a new filename, so the
   manifest's `url` field must be edited every time in addition to the version
   fields. Only continue this pattern if explicitly asked to; otherwise prefer
   pattern 1 for new Windows releases too (upload as a stable
   `Nashama.TV.Setup.exe` and switch the manifest URL once).

## Step-by-step procedure for a new version

1. **Get the new build** (APK or EXE) from the user or build output. Confirm which
   manifest(s) it corresponds to using the file map above.
2. **Compute its SHA256** (Android builds only):
   ```powershell
   Get-FileHash "C:\path\to\newbuild.apk" -Algorithm SHA256
   ```
3. **Upload to the release**, using `--clobber` to overwrite a stable filename:
   ```powershell
   gh release upload app-updates-v1 "C:\path\to\newbuild.apk" --clobber --repo thaerbayyri/new-website
   ```
   If `gh` isn't authenticated in the current environment, run
   `gh auth login --hostname github.com --git-protocol https --web` and have the
   human complete the device-code approval — do not attempt to source a token
   from elsewhere.
4. **Edit the manifest**: bump `versionCode`/`versionName` (or `version` for
   Windows), set the correct `apkSha256` (Android only) from step 2, write real
   `releaseNotes`/`notes` describing the change, and only change the URL if the
   asset filename changed.
5. **Validate before committing**:
   - JSON parses (no trailing commas, matching braces/quotes).
   - `versionCode` increased strictly vs. the previous committed value in that file.
   - The URL points to `github.com/thaerbayyri/new-website/releases/download/...`
     — never a bare domain path.
   - If touching `update.json`/`update-tv.json` together, re-check the
     inconsistency note above still holds or has been resolved correctly.
6. **Commit and push**:
   ```powershell
   git add bayyari-update/<file>.json
   git commit -m "Bump <app name> to <versionName>"
   git push
   ```
7. Report back to the user which file(s) changed and the new version numbers —
   don't assume they tracked which manifest maps to which app.

## Do not

- Do not point any `apkUrl`/`url` at a domain path (`thaerbayyari.com`,
  `thaerbayyari.online`, or any other host) — those files are not deployed there.
- Do not fabricate an `apkSha256` — compute it from the actual uploaded file.
- Do not decrease or repeat a `versionCode`.
- Do not add fields that don't exist in the schema for that file (e.g. don't add
  `versionCode` to the Windows manifest).
- Do not silently resolve the `update.json`/`update-tv.json` versionCode mismatch
  without either confirming the intent with the human or uploading distinct assets
  for each variant.
