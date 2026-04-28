# muddpatcher — Notes for Claude

This repo is a fork of <https://github.com/xackery/eqemupatcher> customized
for the **Mudd Dawgz / Ascendant** EQEmu server. Don't touch `EQEmu Patcher/`
source code unless absolutely necessary — the upstream patcher logic is
load-bearing and brittle. Most maintenance work is just dropping new client
files into `rof/` and pushing.

---

## Layout

| Path | Purpose |
|---|---|
| `rof/` | Files distributed to player EQ folders. Anything in here ends up next to `eqgame.exe` after they patch. Tracked in git. |
| `rof/eqhost.txt` | Loginserver address. Currently `Host=20.51.117.201:5999`. |
| `rof/eqemupatcher.png` | Splash image. **Filename is hardcoded** in `MainForm.cs` — must stay literally `eqemupatcher.png` even though the .exe is now `muddpatcher.exe`. 400×450 PNG. |
| `rof/spells_us.txt`, `dbstr_us.txt`, `BaseData.txt`, `SkillCaps.txt`, `spells_us_norequirements.txt` | Server-truth client files exported from the EQEmu DB. |
| `EQEmu Patcher/` | C# / WinForms source for the patcher GUI (Visual Studio solution). Built only by GitHub Actions, never locally on this Linux box. |
| `.github/workflows/build.yml` | CI: runs filelistbuilder against `rof/`, builds the patcher (Windows runner), publishes a release. |
| `filelistbuilder.yml` | Upstream template; not actually used at build time (CI invokes `filelistbuilder-win-x64.exe rof <STORAGE_URL> <patcher.exe>` directly). |

---

## Build pipeline (CI)

Trigger: `push` to `master`. Run at <https://github.com/numbersix6/muddpatcher/actions>.

Steps:

1. Checkout
2. MSBuild restore + build of `EQEmu Patcher.sln` → produces `eqemupatcher.exe` (the C# `<AssemblyName>` is locked to `eqemupatcher`)
3. `filelistbuilder` runs over `rof/`:
   - md5-hashes every file in `rof/`, writes `filelist_rof.yml` listing them
   - hashes the just-built `eqemupatcher.exe`, writes `eqemupatcher-hash.txt`
   - `STORAGE_URL` is baked in as `https://raw.githubusercontent.com/numbersix6/muddpatcher/master/rof/`
4. Renames the exe and hash to use `${FILE_NAME}` (default: `muddpatcher`):
   - `eqemupatcher.exe` → `muddpatcher.exe`
   - `eqemupatcher-hash.txt` → `muddpatcher-hash.txt`
5. `marvinpinto/action-automatic-releases` publishes a tag `1.0.6.<run_number>` with three assets attached:
   - `muddpatcher.exe`
   - `filelist_rof.yml`
   - `muddpatcher-hash.txt`

`FILE_NAME` is a single env var in `build.yml` controlling that rename. The
`<AssemblyMetadata("FILE_NAME", "muddpatcher")>` baked into the assembly is
what tells the running patcher to fetch `muddpatcher-hash.txt` for self-update
(see `MainForm.cs:225` `string url = $"{patcherUrl}{fileName}-hash.txt";`).

---

## Runtime flow on the player's PC

1. Player puts `muddpatcher.exe` in their EQ folder (alongside `eqgame.exe`).
2. Patcher md5-hashes `eqgame.exe` and looks it up in a hardcoded allowlist
   (`MainForm.cs:291-331`). If unknown, it pops the "couldn't detect" dialog.
3. Patcher downloads `filelist_rof.yml` from the **release URL**
   (`https://github.com/numbersix6/muddpatcher/releases/latest/download/`)
   and patches files from `https://raw.githubusercontent.com/numbersix6/muddpatcher/master/rof/`.
4. Patcher checks `muddpatcher-hash.txt` against its own md5. If they differ,
   it self-updates by downloading the latest `.exe` from the release.
5. After patching, on next startup it loads `eqemupatcher.png` from its own
   directory as the splash. Filename is hardcoded; don't rename it.

---

## Common maintenance tasks

### Ship updated client files

Don't do this manually. The post-merge hook on `/home/EqEmu/Ascendant-Server`
runs `/home/EqEmu/bin/update-patcher.sh` after every successful `git pull`,
which exports client files, copies them into `rof/`, and pushes if anything
changed. CI auto-cuts a release.

If you need to do it by hand: `/home/EqEmu/bin/update-patcher.sh`.

### Add a new eqgame.exe hash to the allowlist

Player reports "couldn't detect EQ in this directory." Get their md5
(`certutil -hashfile eqgame.exe MD5` on Windows). Edit `MainForm.cs` and add
their hash (UPPERCASE — the C# switch is case-sensitive) to the appropriate
`case` group based on what client they're running. For RoF2 variants it's the
group ending around line 320 (`currentVersion = VersionTypes.Rain_Of_Fear_2`).
Commit + push; CI rebuilds.

### Change the loginserver IP

Edit `rof/eqhost.txt`. Commit + push. The next time players patch, their
`eqhost.txt` gets overwritten with the new address.

### Replace the splash image

Drop a 400×450 PNG into `rof/eqemupatcher.png` (must keep that filename).
Commit + push. Players' next patch picks it up.

### Modify the patcher source (`MainForm.cs` etc.)

Pushing changes to anything under `.github/workflows/` OR the patcher source
requires a PAT with **Workflows: write** AND **Contents: write**. The PAT
cached at `/home/EqEmu/.git-credentials` should have both; if a push gets
rejected with "refusing to allow a Personal Access Token to create or update
workflow", the PAT's Workflows scope was dropped — re-grant it at
<https://github.com/settings/personal-access-tokens>.

---

## Gotchas

1. **`eqemupatcher.png` filename is hardcoded.** The `.exe` is now
   `muddpatcher.exe`, but the splash file the patcher looks for is still
   `eqemupatcher.png`. If you rename it the splash silently won't load.
   (`MainForm.cs:271` `Application.ExecutablePath + "\\eqemupatcher.png"`.)

2. **First-run splash is the default RoF image, not the custom one.** The
   patcher loads the custom splash only if `eqemupatcher.png` is already
   on disk in the player's EQ folder. On first launch the file isn't there
   yet — they have to click Patch (downloads the png), then close + reopen.
   The custom splash shows on the second launch onward.

3. **Don't use `PictureBox.Load(path)` for the splash** — it holds a file
   handle open for the lifetime of the form, which prevents subsequent
   patching from overwriting `eqemupatcher.png`. Always read into a
   `MemoryStream` and create a fresh `Bitmap` copy. See the pattern in
   `MainForm.cs` where the custom splash is loaded.

4. **GitHub Actions GITHUB_TOKEN needs `contents: write`** for the
   `marvinpinto/action-automatic-releases` step to publish releases. Either
   the repo setting (Settings → Actions → General → Workflow permissions →
   Read and write) is enabled, or `permissions: contents: write` is in the
   workflow file. Currently it's the repo setting; consider baking it into
   the workflow eventually.

5. **`marvinpinto/action-automatic-releases` is archived upstream.** Works
   today, but if it ever breaks, swap to `softprops/action-gh-release`. Same
   inputs (`tag_name`, `files`, `prerelease`).

6. **CI runs only on push to master.** Branches and PRs build but don't
   release. Force-pushing to master is fine for hotfixes; release tag
   collisions are avoided because the tag uses `github.run_number` (which
   only increments).

7. **`filelistbuilder` invokes the .exe at the un-renamed path** in CI:
   `../EQEmu Patcher/EQEmu Patcher/bin/Release/eqemupatcher.exe`. The rename
   happens AFTER filelistbuilder runs. Don't try to "fix" this — order
   matters because filelistbuilder hashes the patcher itself for the
   self-update flow.

---

## Auth on this box

- Remote: `https://github.com/numbersix6/muddpatcher.git` (HTTPS).
- Credential helper: `store`.
- Credential file: `/home/EqEmu/.git-credentials` (mode 600), holds a
  fine-grained PAT with `Contents: write` + `Workflows: write` on this
  single repo.
- SSH is NOT configured for this repo. The local `~/.ssh/id_rsa` is a deploy
  key for `Twochords/Secret-Server` only; it can't push here.

If the cached PAT expires/revokes, future pushes will start prompting again.
Get a new fine-grained PAT (same scopes), and either:
- Overwrite `/home/EqEmu/.git-credentials` with `https://numbersix6:<PAT>@github.com`
- Or `git push` once and enter user/PAT — the helper saves it.

---

## Quick reference URLs

- Repo: <https://github.com/numbersix6/muddpatcher>
- Actions: <https://github.com/numbersix6/muddpatcher/actions>
- Releases: <https://github.com/numbersix6/muddpatcher/releases>
- Workflow permissions: <https://github.com/numbersix6/muddpatcher/settings/actions>
- Repo PAT scopes (when revoking/regenerating): <https://github.com/settings/personal-access-tokens>
