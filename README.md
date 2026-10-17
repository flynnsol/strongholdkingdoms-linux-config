# Stronghold Kingdoms on Linux (Steam + Proton) — working fix

**Symptoms this fixes**

- Launches to a black screen / black map, heavy lag and crackling audio
- Dialogs (e.g. "Your village was lost… Continue") can't be clicked, keyboard does nothing
- Popup: *"This application could not be started. You must enable the .NET Framework…"*
- Random crashes a few minutes in, especially when opening Mail / Factions panels

**Root causes (short version)**

| Problem | Cause | Fix |
|---|---|---|
| Lag, frozen UI, random crash on the finalizer thread | Game runs on Wine Mono instead of Microsoft .NET | Install `dotnet48` |
| `dotnet48` install fails (`status 67`, error 1603, `compressed file wasn't installed`) | Proton prefix DLLs are symlinks to read-only files, so the .NET installer can't overwrite them | Replace the symlinks with real files before installing |
| ".NET Framework must be enabled" popup | Launcher/game target the .NET 2.0 runtime; only 4.8 is installed | Tell both exes to use .NET 4 via their `.exe.config` |
| Can't click anything, map stays black | Wine's built-in D3DX9 text rendering is so slow it monopolises the UI thread | Install native `d3dx9_42` |
| Instant crash on Mail/Factions (`gdiplus/region.c` assertion) | Bug in Wine's built-in GDI+ | Install native `gdiplus` |
| Input lag from accessibility queries | GE-Proton's Xalia gamepad/accessibility helper hammers the WinForms UI | `PROTON_USE_XALIA=0` |
| Crackling audio (optional, only if you hear it) | Audio buffer too small | `PULSE_LATENCY_MSEC=60` |

Tested with **GE-Proton11-9** (Wine-staging 11). Other recent GE-Proton builds will probably work the same way.

---

## 0. Requirements

- **GE-Proton**: install via [ProtonUp-Qt](https://flathub.org/apps/net.davidotek.pupgui2), restart Steam, then *Stronghold Kingdoms → Properties → Compatibility → Force…* → pick GE-Proton.
- **protontricks** (distro package, pip, or Flatpak).
- Game installed and launched at least once.

## 1. Set your paths

Adjust `STEAM_LIB` if the game is in a different Steam library (Steam → game → *Properties → Installed Files → Browse* shows where). For Flatpak Steam, the default library is `~/.var/app/com.valvesoftware.Steam/data/Steam`.

```bash
STEAM_LIB="$HOME/.local/share/Steam"        # <- change if needed
GAME="$STEAM_LIB/steamapps/common/Stronghold Kingdoms"
CDATA="$STEAM_LIB/steamapps/compatdata/47410"
PFX="$CDATA/pfx"

# sanity check, both should list files
ls "$GAME" >/dev/null && ls "$PFX" >/dev/null && echo "paths OK"
```

Run all later steps **in the same terminal** so these variables stay set.

## 2. Start from a clean prefix

```bash
rm -rf "$CDATA"
rm -rf ~/.cache/winetricks/dotnet40 ~/.cache/winetricks/dotnet48   # clears possibly corrupt downloads
```

Now **launch the game once from Steam** and let the launcher finish downloading and updating. Wait until the game itself has started (or you get the .NET popup), then quit. Make sure nothing is still running:

```bash
protontricks -c 'wineserver -k' 47410
```

## 3. Replace Proton's DLL symlinks with real files

This lets the .NET installer overwrite system DLLs. It only touches file symlinks inside `drive_c/windows` and never touches `dosdevices` (which links to your whole filesystem).

```bash
cp -a "$PFX" "$CDATA/pfx-backup"          # backup (~few hundred MB)

cd "$PFX/drive_c/windows" || exit
find . -type l | while IFS= read -r f; do
  t=$(readlink -f "$f")
  if [ -f "$t" ]; then
    rm "$f" && cp "$t" "$f"               # symlink to a file -> real copy
  elif [ ! -e "$t" ]; then
    rm "$f"                               # broken symlink -> remove
  fi                                      # symlinks to directories are left alone
done
chmod -R u+w .
rm -f system32/mscoree.dll syswow64/mscoree.dll   # installer writes its own
echo "symlinks left: $(find . -type l | wc -l)"
cd - >/dev/null
```

The prefix grows by roughly 1 GB. Wine falls back to its built-in copy of any DLL that's missing, so the deletions are safe.

## 4. Install the native Windows components

```bash
protontricks 47410 -q -f dotnet48
protontricks 47410 -q -f d3dx9_42 corefonts gdiplus
```

`dotnet48` takes several minutes and installs .NET 4.0 first. Ignore the *"may not fully work on a 64-bit installation"* warning.

**Verify .NET installed** (expect `Release` ≥ `0x80ea8`):

```bash
protontricks -c 'wine reg query "HKLM\Software\Microsoft\NET Framework Setup\NDP\v4\Full" /v Release' 47410
```

If it fails with `compressed file wasn't installed (L"<something>.dll")`, find that DLL in `"$PFX/drive_c/windows/system32"` or `syswow64`, delete it, and re-run the `dotnet48` command.

## 5. Make the launcher and game use .NET 4

Both executables ask for the old .NET 2.0 runtime. A `<startup>` block in their `.exe.config` redirects them to 4.8. This helper inserts the block in the right place, makes a `.bak` of any file it changes, and skips files that already have one:

```bash
add_v4_startup() {
  f="$1"
  block='    <startup useLegacyV2RuntimeActivationPolicy="true">\n        <supportedRuntime version="v4.0"/>\n    </startup>'
  if [ ! -f "$f" ]; then
    printf '<?xml version="1.0" encoding="utf-8"?>\n<configuration>\n%b\n</configuration>\n' "$block" > "$f"
    echo "created: $f"
  elif grep -q '<startup' "$f"; then
    echo "already set: $f"
  else
    cp "$f" "$f.bak"
    if grep -q '</configSections>' "$f"; then
      sed -i "s#</configSections>#&\n$block#" "$f"        # must come after configSections
    else
      sed -i "0,/<configuration[^>]*>/s##&\n$block#" "$f"
    fi
    echo "patched: $f"
  fi
}

# launcher
add_v4_startup "$GAME/StrongholdKingdoms.exe.config"

# game client (lives in a versioned folder inside the prefix)
for d in "$PFX/drive_c/ProgramData/Firefly Studios/Stronghold Kingdoms"/*/; do
  [ -f "${d}StrongholdKingdoms.exe" ] && add_v4_startup "${d}StrongholdKingdoms.exe.config"
done
```

If the versioned folder doesn't exist yet, the launcher hasn't downloaded the client. Go back to the launch in step 2.

## 6. Launch options

Steam → game → *Properties → General → Launch Options*:

```
PROTON_USE_XALIA=0 %command%
```

*Optional:* if the audio crackles, add `PULSE_LATENCY_MSEC=60` at the front (try `90` if it still crackles). Don't leave `PROTON_LOG=1` on during normal play, because it slows the game down noticeably.

---

## Keeping it working

- **Don't switch Proton versions.** A version change makes Proton update the prefix, and that can overwrite the native DLLs. If you do switch, redo steps 2–5.
- **The .NET popup comes back after a game update:** the client moved to a new version folder. Re-run the `for d in …` loop from step 5.
- **The .NET popup comes back after "Verify integrity of game files":** Steam restored the launcher's config. Re-run `add_v4_startup "$GAME/StrongholdKingdoms.exe.config"`.
- **Back up the working prefix** so you can restore it quickly:
  ```bash
  cp -a "$CDATA" ~/stronghold-kingdoms-prefix-working
  ```

## Debugging a new crash

Run once with `PROTON_LOG=1 PROTON_USE_XALIA=0 %command%`, reproduce the crash, then:

```bash
tail -80 ~/steam-47410.log | grep -vE 'RtlGrowFunctionTable|NtGetContextThread'
grep -nE 'wine: Unhandled|page fault on|_wassert' ~/steam-47410.log | tail
```

An `err:msvcrt:_wassert` line names the Wine DLL that failed (e.g. `dlls/gdiplus/...`). Installing the native version of that component with protontricks is usually the fix.
