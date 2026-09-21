# FineWine Patcher.app

A minimal macOS app that turns a copy of **CrossOver 26.2** into the patched build that runs
**Arknights: Endfield** on Apple Silicon. It is the GUI equivalent of
[`scripts/swap-into-crossover.sh`](../scripts/swap-into-crossover.sh)'s core Wine-module swap:

1. Copies your `CrossOver.app` (the original is never touched).
2. Swaps in the three pre-built patched Wine modules bundled inside the app
   (`ntdll.so`, `kernel32.dll`, `ntoskrnl.exe`), keeping `.cxorig` backups.
3. Ad-hoc signs the swapped files, strips the bundle seal, removes quarantine.
4. Verifies the swap (sizes, `ntdll.so` signature, and the `lib64` rpath D3DMetal needs).

Out of scope by design: the optional GPTK4 / MoltenVK graphics upgrades (Apple's GPTK may not
be redistributed) — see the [main README](../README.md#graphics--performance-gptk4) for those.

**End users need no developer tools** — the `lib64` rpath is baked into the payload at app-build
time, so at patch time the app only uses `codesign` and `xattr`, which ship with macOS.

## Mod chain (EFMI) — optional second phase

Below the patch flow there is an independent **bottle-side** phase that makes the **XXMI Launcher /
EFMI** mod stack work: it writes 3DMigoto's own chain-load option,

```ini
[System]
proxy_d3d11 = <the backend d3d11.dll inside the patched app>
```

into EFMI's `d3dx.ini`, so the mod's `d3d11.dll` hands off to this app's **D3D11→Metal backend**
instead of Wine's wined3d. (Why this is needed at all — 3DMigoto resolves its "original" as
`C:\Windows\system32\d3d11.dll`, which under CrossOver is wined3d on *every* backend — is
documented in [docs/mod-injection/](../docs/mod-injection/07-load-ordering-and-chaining.md).)

What it does, in order:

1. Reads the bottle's graphics backend from `cxbottle.conf`
   (`CX_ACTIVE_GRAPHICS_BACKEND`, falling back to `CX_GRAPHICS_BACKEND`). A bottle on
   **wined3d** (no backend) needs no chain and the phase reports that.
2. Finds the EFMI folder (XXMI Launcher's `XXMI Launcher Config.json` → its default
   `%APPDATA%\XXMI Launcher\EFMI` → or you pick it).
3. Backs up `d3dx.ini` → `d3dx.ini.cxorig` (once), then sets `proxy_d3d11` inside `[System]`
   **only** — everything else in the file stays byte-identical, and the shipped
   `;proxy_d3d11=…` example line is kept.
4. Verifies the edit and writes a `d3dx.ini.finewine-chain` state file (app path, backend, mode,
   target, SHA-256, date) so re-runs and reverts know what they did.

Guarantees:

- **Idempotent** — re-running with the same app + backend changes nothing.
- **Byte-exact revert** — "Revert" removes only the two added lines and restores any line the
  apply replaced (recorded in the state file as `replaced_line`); if XXMI has since rewritten
  `d3dx.ini`, the pre-chain backup is used instead.
- **Backend-aware** — if you switch the bottle's backend (`d3dmetal` ↔ `dxmt` ↔ `dxvk`), re-running
  rewrites the target.
- **No CrossOver files are bundled or shipped** — the backend `d3d11.dll` is referenced at its
  location inside *your* patched app (or, when the bottle has no `Z:` mapping, copied from it into
  the EFMI folder as `d3d11_cx.dll`). It is never included in the app payload or a release.
- It never changes the bottle's `CX_GRAPHICS_BACKEND`; it adapts to it.

The same feature is available from the shell, either with flags or environment variables
(a flag always wins over its env var):

```bash
scripts/swap-into-crossover.sh --mod-chain --skip-app-patch --bottle "Arknights Endfield"   # apply
scripts/swap-into-crossover.sh --mod-chain-revert --skip-app-patch                          # undo
```

Other flags: `--app PATH` (patched app to chain against), `--importer PATH` (explicit EFMI
folder), `--chain-mode path|copy`, `--bottles-root PATH`, plus the app-patch flags
(`--src-app`, `--dest-app`, `--gptk-dir`, `--skip-gptk`, `--skip-mvk`). `--help` has the full
list; the env equivalents (`MOD_CHAIN`, `MOD_BOTTLE`, …) are documented in the script header.

After applying, verify with experiment **L3** in
[docs/mod-injection/06-experiment-plan.md](../docs/mod-injection/06-experiment-plan.md): launch
Endfield via XXMI and check `d3d11_log.txt` for *"Proxy loading active, Forcing
load_library_redirect=0"*. Note that XXMI rewrites `d3dx.ini` when it updates EFMI
(`overwrite_ini=True`), so re-run the phase (or disable that option in XXMI) after an update.

## Building the app

Requires the Xcode Command Line Tools only (no Xcode):

```bash
# 1. Build the patched Wine first (once) — produces build/wine-build64
./scripts/build-wine.sh all

# 2. Build the app around it
./patcher-app/scripts/build-app.sh
open "patcher-app/build/FineWine Patcher.app"
```

`PAYLOAD_DIR` overrides where the modules come from (the `build/wine-build64` tree layout or a
flat directory with the three files). `CODESIGN_ID` sets a real signing identity (default:
ad-hoc). `ALLOW_MISSING_PAYLOAD=1` produces a payload-less smoke-test build that refuses to patch.

### App icon

`build-app.sh` bakes in `Resources/AppIcon.icns` (a committed file). To regenerate it after
changing the source art, run `scripts/make-appicon.sh` — it renders the full macOS size set from
`Resources/appicon/appicon-source.png` with Apple tools only (`swift` + `iconutil`, no
third-party image libraries), centering the artwork on a transparent square with a 6% margin
(`MARGIN=…` to change). If `AppIcon.icns` is absent, the app simply builds without a custom icon.

## Licensing (important if you distribute the built app)

- **The app itself** (Swift sources, UI, this directory): [MIT](../LICENSE).
- **The bundled payload** (`ntdll.so`, `kernel32.dll`, `ntoskrnl.exe`): **LGPL-2.1-or-later** —
  they are Wine, built from CodeWeavers' freely published
  [CrossOver 26.2 Wine source](https://www.codeweavers.com/crossover/source) with this repo's
  [patches](../patches/) applied (which include the
  [dw-proton](https://dawn.wine/) anti-cheat patches — see [patches/README.md](../patches/README.md)
  for authorship).
- The app's **Licenses…** window shows all of this, with the full license texts, offline.

If you publish a built `FineWine Patcher.app` (e.g. a GitHub Release), LGPL-2.1 requires you to
make the **complete corresponding source** of the payload available: this repository's patches +
the exact `crossover-sources-26.2.0` archive from
[media.codeweavers.com/pub/crossover/source](https://media.codeweavers.com/pub/crossover/source/).
Best practice: attach (or mirror in a release) the exact source tarball you built from, so your
source offer doesn't depend on a third-party URL staying alive.

The app never contains or redistributes CrossOver itself, Apple's Game Porting Toolkit,
MoltenVK, or the game. It requires the user's own licensed CrossOver install as input, and links
to [codeweavers.com/store](https://www.codeweavers.com/store) for buying one.

## Layout

```
Package.swift                     SwiftPM manifest (macOS 13+)
Sources/FineWinePatcher/
  FineWinePatcherApp.swift        app entry
  ContentView.swift               the single-window UI (incl. the Mod chain group box)
  PatcherEngine.swift             the patch steps (mirrors swap-into-crossover.sh) + the mod-chain phase
  ModChain.swift                  the mod (EFMI) chain support: backend/bottle/EFMI discovery,
                                  the d3dx.ini [System] proxy_d3d11 edit, verify, revert, state file
  LicensesView.swift              the Licenses window + component metadata
Tests/FineWinePatcherTests/
  ModChainTests.swift             the mod-chain ini-editing rules (swift test)
Resources/licenses/               license texts bundled into the app
Resources/AppIcon.icns            the app icon (committed; regenerate with make-appicon.sh)
Resources/appicon/                the icon source art
scripts/build-app.sh              compile + assemble + payload staging + signing
scripts/make-appicon.sh           regenerate AppIcon.icns from the source art
scripts/make-appicon.swift        the icon renderer (ImageIO/Core Graphics)
build/                            (gitignored) the assembled .app
```
