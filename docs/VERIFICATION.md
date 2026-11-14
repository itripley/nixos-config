# Verification

What was checked before install, how, what it found, and what remains unproven.

## Why the check runs on the Unraid server

Nix doesn't run natively on Windows, so the config has to be evaluated somewhere Linux-y. Two local
options were tried and both failed because the Windows hypervisor isn't running on the current boot
(details in [INVENTORY.md](INVENTORY.md#windows-side-issue-found-along-the-way)):

| Tried | Result |
|---|---|
| Docker Desktop (local) | engine never starts; API returns HTTP 500, no `docker-desktop` WSL distro |
| NixOS-WSL 2605.7.2 (image downloaded, sha256 verified) | `wsl --install` fails with `HCS_E_HYPERV_NOT_INSTALLED` |
| **Docker on the Unraid server** (Engine 29.8.2, 192.168.1.65, over SSH) | ✅ works, and is the method used |

## How `scripts/verify.ps1` works

1. `git add -A`, because flakes only see tracked files.
2. Tars the tracked files (plus a small `run.sh`) on Windows.
3. Streams the tar over SSH (`~/.ssh/id_unraid`) into `docker run -i --rm nixos/nix`. Nothing is
   written to the server's disks except the Docker volume below.
4. Inside the container:
   `nix eval --raw path:.#nixosConfigurations.desktop-olivia.config.system.build.toplevel.drvPath`.
   This evaluates the **entire system** (every module, option, package and generated file) without
   building or downloading the packages themselves.
5. Prints warnings and `OK: /nix/store/…-nixos-system-desktop-olivia-26.05….drv`, or the error with
   its location in the repo.

`path:.` is used instead of `.` so Nix reads a plain directory: no `.git` is sent, which avoids git's
"dubious ownership" checks in the container.

**Server footprint:** the `nixos/nix` image (~500 MB) and the `nixos-eval-store` volume (~1.1 GB),
which caches downloaded sources so re-runs are fast. Remove both with
`docker volume rm nixos-eval-store && docker rmi nixos/nix` on the server.

## What was found and fixed

| # | Problem | Found by | Fix | Why this fix |
|---|---|---|---|---|
| 1 | `blender-hip` has been removed from nixpkgs (2026-01) | full eval | `pkgsRocm.blender` | The replacement the removal notice names; still the HIP/ROCm build for the 7800 XT. |
| 2 | `openssl-1.1.1w` is insecure, required by `sublime4` | eval, then a per-package probe that forces each package's derivation | permit exactly `openssl-1.1.1w` | Keeps Sublime Text working; scoped by exact name and documented so it's removed with Sublime. |
| 3 | `programs.ssh.matchBlocks` deprecated in HM 26.05 | eval warning | rewritten as `programs.ssh.settings` with OpenSSH directive names | Read the pinned HM module to get the new schema. The system derivation hash was **identical** before and after, proving the generated `~/.ssh/config` is unchanged. |

**Package probe:** all 116 package names (system, user, VS Code extensions, fonts, unstable CLIs) were
checked individually against the pinned nixpkgs for missing, removed, broken/insecure, or
throwing on evaluation. Only #1 and #2 failed.

## Firefox-specific checks

- Evaluated and printed the generated files: `configPath = .config/mozilla/firefox`, `profiles.ini`
  (profile `default-release` → `9ylnf32e.default-release`, default), `user.js` (the three prefs),
  exactly one Firefox (159.0.1) installed, Bitwarden native host passed to the wrapper.
- Pinned nixpkgs Firefox = **159.0.1**, the same as Windows, so the profile can move.
- Read the nixpkgs Firefox wrapper: it sets `MOZ_LEGACY_PROFILES=1` (uses `profiles.ini`, good) and
  `MOZ_ALLOW_DOWNGRADE=1` (no downgrade protection, hence the script's own version guard).
- The migration script was tested on a mock profile; results are in [FIREFOX.md](FIREFOX.md#testing-done-and-not-done).
- 2026-11-09, `userChrome.css`: 159.0.1's `omni.ja` has no `id="sidebar-main"`. The launcher's
  parent is `<box id="sidebar-container">`, and `#sidebar-box` still gets `checked="true"`. The rule
  now hides `#sidebar-container`. **Still to check by hand:** after `nh os switch`, quit and reopen
  Firefox. With Sidebery open, there should be no native launcher strip.

## Current result

```
OK: /nix/store/…-nixos-system-desktop-olivia-26.05.20261105.c56a87d.drv
```

Zero warnings, zero errors against `flake.lock`. Pinned revisions:

| Input | Branch | Rev |
|---|---|---|
| nixpkgs | nixos-26.05 | `c56a87d8` |
| nixpkgs-unstable | nixos-unstable | `f4a03c00` |
| home-manager | release-26.05 | `24f21905` |
| plasma-manager | | `a19a2a02` |
| disko | | `cedd626e` |
| nix-flatpak | | `02656785` |
| zen-browser (removed 2026-11-09) | | `5f45f55b` |
| serein (added 2026-11-09) | main | `c028eee6` |
| claude-desktop (added 2026-11-09) | main | `b16f4856` |
| artcraft (added 2026-11-09) | main | `6a3de1d2` |
| vicinae (added 2026-11-09) | main | `84a0086f` |
| nixpkgs-darwin (added 2026-11-10, Mac) | nixpkgs-26.05-darwin | `57f1c896` |
| nix-darwin (added 2026-11-10, Mac) | nix-darwin-26.05 | `d31953cf` |

## First real build (2026-11-08)

The install itself hit build failures, and to get a bootable system a set of apps in `apps.nix`
and `neovim`/`vscode` in `dev.nix` were commented out. Back on the installed system, the original
config was rebuilt with `nix build --keep-going` to find which packages really fail. Most of the
disabled apps built fine. Only these failed, all because of upstream bugs:

| # | Package | Error | Cause | Fix |
|---|---|---|---|---|
| 4 | `vscode` 1.144.0 | `rm: cannot remove 'resources/app/node_modules/@vscode/ripgrep/bin/rg'` | VS Code moved ripgrep to `@vscode/ripgrep-universal/bin/linux-x64/rg`; nixpkgs' `postPatch` still uses the old path | `overlays/build-fixes.nix`: link nixpkgs' ripgrep at the new paths |
| 5 | `openssl-1.1.1w` → `sublime4` | `30-test_afalg.t ... Failed: 1` | the test uses the running kernel's AF_ALG socket, which Linux 7.3 handles differently | drop that one test |
| 6 | `unstable.codex` 0.167.0 | `AWS_LC_SYS_USE_SYSTEM=1 requires a system-installed AWS-LC`, then `zstd-sys: Can't probe for zstd in pkg-config` | nixpkgs' new `cargoUseSystemLibs` hook forces system C libraries; codex's inputs weren't updated | `dontCargoUseSystemLibs = true` |
| 7 | `programs.neovim` | `Error installing file '.config/nvim/init.lua' outside $HOME` | HM 26.05 always writes `nvim/init.lua`, but `~/.config/nvim` is a live link to `dotfiles/nvim` | `sideloadInitLua = true` (HM passes its lines through the wrapper) |

#7 only showed up after #4–#6 were fixed, because `home-manager-files` is built after the packages.
`makemkv` logs `curl: (22) ... 404` but succeeds: its fetcher falls back to `makemkv.com/download/old/`.
codex is not in the binary cache (it fails on Hydra too), so it compiles locally (~10 min).

After the fixes, the full system (`config.system.build.toplevel`) with **every** app from the
original lists builds.

## Problems found after first boot (2026-11-08/09)

| # | Problem | Cause | Fix | Commit |
|---|---|---|---|---|
| 8 | WezTerm not installed | the original config linked `~/.config/wezterm` but never added the package | `wezterm` in `dev.nix` (plain package; `programs.wezterm` would write `wezterm.lua` into the live link) | `bb89df5` |
| 9 | `netbird up` logged in to NetBird's cloud (`LoginFailed`) | the server is self-hosted at `netbird.weblivia.dev` (found in Windows' `C:\ProgramData\Netbird\default.json`) | `services.netbird.clients.default.config.ManagementURL` | `8628c39` |
| 10 | After `nh os switch`, `netbird up` *still* went to the cloud | the module merges `config.d/50-nixos.json` only when the daemon starts, and a switch doesn't restart it when only that file changes | `systemd.services.netbird.restartTriggers` on that file. (The login was done once with `--management-url`, which also persists.) | `93865f2` |
| 11 | No internet while NetBird is connected | the server pushes **Exit Node (unraid)** (`0.0.0.0/0`, `::/0`); the client auto-selected it, the unraid peer never answered, so all traffic hit WireGuard with no peer (`write: required key not available`) | `netbird networks deselect "Exit Node (unraid)" "Exit Node (unraid)-v6"`, exactly what Windows' `state.json` had. Stored in `/var/lib/netbird/state.json`. | `93865f2` (docs) |
| 12 | No graphical VPN toggle for NetBird | WARP's `warp-taskbar` autostarts from its package; NetBird's tray app doesn't, and a second plain `netbird-ui` in `apps.nix` duplicated the menu entry | `makeAutostartItem` for the module's wrapped `netbird-ui`; dropped the duplicate | `7644365` |
| 13 | Fedora migration: `electron-41.10.7` refused as insecure | stable `ente-desktop` 1.7.22 is pinned to an EOL Electron | `unstable.ente-desktop` (1.7.27) | `0ef7162` |
| 14 | Fedora migration: `wineWowPackages` deprecation warning | renamed in 26.05 | `wineWow64Packages.stable` | `0ef7162` |
| 15 | Serein: `AWS_LC_SYS_USE_SYSTEM=1 requires a system-installed AWS-LC` | same as #6: its flake builds with nixpkgs-unstable, whose `cargoUseSystemLibs` hook forces system AWS-LC | `.overrideAttrs { dontCargoUseSystemLibs = true; }` where it's installed (`apps.nix`) | `5d2f60a` |
| 16 | No WARP on/off switch in the tray | `warp-taskbar` is Zero Trust only: with a free registration each click opens a team sign-in | `home/olivia/warp-gui.nix`: packaged warp-cloudflare-gui, `warp-taskbar` autostart hidden | `cc26af0` |
| 17 | Serein can't be resized smaller than ~836×572 px | `MIN_WINDOW_SIZE: [u32; 2] = [760, 520]` (logical px) in `apps/desktop/src/app_settings.rs`, applied with `with_min_inner_size`; no setting overrides it. Not the custom title bar: turning that off changed nothing. | `postPatch` with `substituteInPlace --replace-fail` → `[400, 300]` (`apps.nix`). A KWin minimum-size rule wouldn't work: winit clamps configures to its own minimum. ✅ patch applies and builds; the system closure has the new Serein path (nvd lists no change, same version 0.1.0). | not committed yet |

Other things checked along the way:
- Serein runs (Vulkan on the RX 7800 XT). The system `ldd` reports `GLIBC_2.43/2.44 not found` for it,
  which is misleading: it's linked against unstable's glibc 2.44 and its own loader resolves every
  library (`<its ld-linux> --list` shows none missing).
- The NetBird tray app (`netbird-ui`) and the CLI both default to the module's socket
  `unix:///var/run/netbird/sock`, so the plain package wasn't broken, just redundant.
- Fedora's Krohnkite binds `Meta+,` / `Meta+\` / `Meta+|` were Krohnkite's own defaults (decoded from
  `kglobalshortcutsrc` escaping), so they're left for Krohnkite to register.
- plasma-manager splits `configFile` group names on `/`, so `"Greeter/Wallpaper/<plugin>/General"`
  becomes `[Greeter][Wallpaper][<plugin>][General]` (checked in the generated `data.json`).
- The theme artwork links point at `themes/art/` through the store (`mkOutOfStoreSymlink`), and only
  `art/.gitkeep` is tracked by git.

## App changes (2026-11-09)

Built on the installed system with
`nix build .#nixosConfigurations.desktop-olivia.config.system.build.toplevel` after each change.
Switched to the same day: on 2026-11-09, `/run/current-system` was the exact output of that commit's
build.

| Change | Commit | Checked |
|---|---|---|
| PhotoCraft + FilmCraft (`artcraft` input, `programs.artcraft` in `desktop.nix`) | `3c298f4` | ✅ both build; `photocraft-cli --version` → 0.21.0, `filmcraft-cli --version` → 0.4.0; each ships a `.desktop` file. GUIs not opened. |
| 43 apps removed, plus their config | `1aba8c9` | ✅ evaluates and builds; `nix flake lock` dropped zen-browser and its inputs |
| Warehouse removed | `663b28e` | ✅ builds |
| linuxmeeter (`pkgs/linuxmeeter`) | never committed; uninstalled the same day | ✅ builds (pinned `3245d2d`); 67 unit tests pass; wrapper sets `LV2_PATH` to lsp-plugins 1.2.29, which has all four plugins it loads (`gate_stereo`, `compressor_stereo`, `para_equalizer_x8_stereo`, `limiter_stereo`); system PipeWire 1.6.6 ships the filter-graph LV2 plugin. `nvd diff`: +linuxmeeter, +tauri-asset-gst-plugin, +9.2 MiB. App not opened. |
| Claude Desktop (`claude-desktop` input) | `9be83f4` | ✅ builds (4.23118.0); its own loader (glibc 2.44) resolves all 89 libraries. The system `ldd` shows `GLIBC_2.43/2.44 not found`, the same false alarm as Serein. App not opened. |

Things found along the way:
- **FilmCraft is behind upstream.** artcraft-nix's `versions.json` stops at FilmCraft 0.4.0, while
  upstream (storytold/filmcraft) is on 0.18.0. PhotoCraft is current (0.21.0). The flake's daily job
  only adds a release after it builds and passes a library check, so newer FilmCraft releases are
  probably failing that check. `nix flake update artcraft` picks them up once it passes them. If it
  stays stuck, the alternative is packaging the upstream AppImage directly.
- artcraft-nix's module would install all seven ArtCraft apps by default; `apps = [ … ]` limits it.
  It also sets `fonts.fontDir.enable` and a tmpfiles link `/usr/share/fonts` →
  `/run/current-system/sw/share/X11/fonts`, because the apps' text tools don't use fontconfig.
- **linuxmeeter overlapped the existing audio setup** (moot since its uninstall later that day). Nothing clashes by name (its nodes are all
  `lm.*`), but: (1) its virtual inputs *System / Music / Chat* sit next to the `audio.nix` loopback
  sinks *Game / Music / Chat*, so the output list shows two "Music" and two "Chat"; (2) EasyEffects
  sits in the mic path ahead of linuxmeeter, so any EasyEffects input effect (e.g. RNNoise) runs before
  linuxmeeter's gate / compressor / EQ, and EasyEffects' output effects follow the default sink, which
  linuxmeeter's *default-output takeover* changes; (3) its tray, close-to-tray and autostart toggle
  are managed from the app, not Nix. Checked live after the switch (`wpctl status`, `pw-link -l`): no
  stream used the old sinks; the mic went Seiren → EasyEffects → `lm.strip.1.cap`, with no effects
  loaded in EasyEffects, so nothing was processed twice. Both kept on purpose.
- After the removals, `flatpak.nix` has an empty list on purpose: nix-flatpak uninstalls the
  Flatpaks it installed earlier on the next activation. `flatpak list --app` was already empty on
  2026-11-09 (JDownloader and Stremio were no longer installed), so in practice there's nothing left.

After the switch, check by hand:

| Area | How |
|---|---|
| PhotoCraft / FilmCraft | Both open from the app menu. The text tool lists the system fonts (if not: `ls -l /usr/share/fonts` should point at `/run/current-system/sw/share/X11/fonts`). |
| Claude Desktop | Opens on Wayland and signs in; Chat, Cowork and Code tabs load. |
| Removed apps gone | e.g. `which gimp wine codex` finds nothing; the Meta+F shortcut no longer opens Thunar. |
| Leftover data | ✅ Home-folder leftovers (Zen, Krita, Vesktop, Thunar; ~290 MB) and `/var/lib/libvirt` were deleted on 2026-11-09. See [OPERATIONS.md](OPERATIONS.md#leftover-data-from-removed-apps). |

## Audit (2026-11-09)

A read-only audit against current NixOS best practices, checked against the pinned nixpkgs, followed by
the fixes (commits `b96c7e4`…`ad773b9`, merged into `main`).

**Before the fixes:**

| Check | Result |
|---|---|
| `nix flake check`, full eval (`…toplevel.drvPath`) | ✅ no errors or warnings; eval ≈10 s; no import-from-derivation (`--option allow-import-from-derivation false` still evaluates) |
| Build vs running system | ✅ HEAD's build was identical to `/run/current-system`; `systemctl --failed` empty (system and user) |
| Clean clone | ✅ same `.drv` as the working copy; `nix store diff-closures` against the running system empty |
| `nix flake metadata` | ✅ every input locked, no duplicate nixpkgs, lock 0–3 months old |
| `nix-env -q`, `nix profile list` | ✅ empty (no imperative packages) |
| `nixos-rebuild build-vm` | ❌ failed: `services.btrfs.autoScrub` assertion (the VM has no btrfs). `vmWithDisko` built. |
| Channels | ❌ `nix.channel.enable` on, root had the ISO's `nixos` channel, and the installer's `/etc/nixos/configuration.nix` was still there: a plain `nixos-rebuild switch` without `--flake` would have built that stock config |
| `trusted-users` | ❌ `@wheel`: a passwordless route to root |
| Closure | 43.1 GiB, 3367 paths. Biggest: Resolve 6.5 GiB, ROCm, CEF (OBS), Proton-GE. Avoidable: orca/speech-dispatcher, Akonadi + MariaDB, Prism's JDK 8/17, a second .NET SDK. |
| statix / deadnix / nixfmt | style warnings only; 3 unused arguments; 3 files unformatted |
| Firewall | ✅ every open port traced to the module that opens it (LocalSend, plex-mpv-shim, Plex GDM, Steam, KDE Connect, Avahi, NetBird, WARP) |
| `vulnix --system` | ⚠️ not run: the NVD feed download timed out |

**After the fixes** (each commit built with `nix build --no-link`): `nix flake check` runs eval + lint
in ≈15 s and passes; the lint check was confirmed to fail on a file with an unused argument;
`build-vm` builds; `nix.nixPath` = `["nixpkgs=flake:nixpkgs"]`; `trusted-users = root`; kernel
command line has `amdgpu.ppfeaturemask=0xfffd7fff`; closure **40.6 GiB**. The CIFS/NTFS fstab entries
still carry `uid=1000,gid=100`. warp-cloudflare-gui's wrapper `PATH` is now cloudflare-warp's own
`bin`.

**One-time steps before applying them** (outside the repo, need root):

```sh
sudo rm /etc/nixos/configuration.nix /etc/nixos/hardware-configuration.nix   # keep smb-secrets
sudo nix-channel --remove nixos     # before the switch: nix-channel goes away with channels off
nh os boot && reboot                # overdrive needs a reboot
```

✅ Done on 2026-11-09: `/etc/nixos` holds only `smb-secrets`, root's channel profile is empty,
`NIX_PATH` is `nixpkgs=flake:nixpkgs`, `trusted-users = root`, and the booted kernel command line has
`amdgpu.ppfeaturemask=0xfffd7fff`.

**After the reboot, check by hand:**

| Area | How |
|---|---|
| LACT | The GPU page shows clock/voltage controls; `cat /proc/cmdline` has `amdgpu.ppfeaturemask`. |
| WARP tray | Connect/Disconnect still work (its `warp-cli` now comes from the package, not the system PATH). |
| Channels | `echo $NIX_PATH` → `nixpkgs=flake:nixpkgs`; `nix-shell -p hello` works. |
| Plasma | Logs in normally without Akonadi and Orca; the calendar widget still shows dates. |
| Prism Launcher | Minecraft 1.20.5+ instances start. Older ones need `jdk17` / `jdk8` back (OPERATIONS). |
| Godot C# / VS Code | Godot-mono builds C# projects with its own SDK. If VS Code's C# extension can't load a project, add `dotnet-sdk` back in `dev.nix`. |
| Root password | `sudo passwd -S root` shows `P` (set). Keep it: it's the emergency-mode login (DESIGN §9). |

**Left as they are, on purpose:** the `docker` group (root-equivalent, kept for sudo-less Docker;
DESIGN §9); blanket `allowUnfree` (DESIGN §4); `mutableUsers = true`; no `system.autoUpgrade`
(DESIGN §10); the `ntfs` entry in `supportedFilesystems`; `gamescope.capSysNice` (reported to break
gamescope started from inside Steam; test a game with a gamescope launch option, and set it to
`false` if it fails).

**Due in December 2026:** NixOS 26.05 is supported until about the end of December, a month after
26.11 comes out. Upgrade with the steps in [OPERATIONS.md](OPERATIONS.md) ("Upgrade to the next
NixOS release"), using `nh os boot`, and re-test `overlays/build-fixes.nix`.

## Checked on the installed system (2026-11-08)

| Area | Result |
|---|---|
| Disk id / disko | ✅ installed on the Samsung, Windows NVMe untouched |
| Display | ✅ DP-1 2560×1440 @ 165 Hz picked automatically; Adaptive Sync set to *Automatic* (`kscreen-doctor output.DP-1.vrrpolicy.automatic`). A second 1600×1200 @ 75 Hz monitor is on DP-2. |
| Services | ✅ `lactd`, `ollama`, `openrgb`, `docker`, `netbird`, `cloudflare-warp` running; no failed units |
| NetBird | ✅ logged in to `netbird.weblivia.dev`; Management, Signal and 2/2 relays connected, 2/5 peers; internet works with it up (exit node deselected). Tray app autostarts. |
| WARP | ✅ registered (free account); `warp-cli` works. ✅ confirmed `warp-taskbar` is Zero Trust only (no switch for the free account); replaced by `warp-gui.nix`, whose tray icon was checked to register with Plasma. Don't connect WARP and NetBird at the same time (both change routing and DNS). |
| PipeWire virtual sinks | ✅ `virtual_game`, `virtual_music`, `virtual_chat` exist. Newer `wpctl status` lists them under **Filters**, not Sinks. |
| SMB mounts | ✅ `/mnt/unraid/*` automount with the credentials file |
| `/mnt/windows` | ✅ mounted read-only (ntfs3) |
| Flatpaks | ✅ JDownloader and Stremio installed (both removed from the config 2026-11-09) |
| SSH | ✅ `ssh unraid` works. ❌ `github-artlivia`: GitHub rejects the deploy key (`Permission denied (publickey)`); re-add `id_ed25519_artlivia.pub` as a deploy key on `itripley/artlivia`. |
| Firefox migration | ✅ script ran: 126 profile add-ons (22 enabled). Opening it in Firefox is still to be checked. |

## Audio routing: not working (2026-11-09)

**Reported by Olivia:** the audio setup is a mess, and she can't get any of the routing to work.
The exact symptoms (which app, which device, silent or wrong output) aren't written down yet.
Pin them down first when this is picked up.

**What's stacked up.** Four tools that all change routing:

| Layer | Source | What it does |
|---|---|---|
| Game / Music / Chat | `audio.nix` loopback sinks | Always on. Each forwards to the default output. |
| EasyEffects | `services.easyeffects` (home/olivia/easyeffects.nix) | Always on. Takes over app output and the mic; **no effects loaded** (no RNNoise; the AirPods Max EQ preset added later on 2026-11-09 isn't loaded at startup). |
| linuxmeeter | `pkgs/linuxmeeter` (**uninstalled** later on 2026-11-09) | Only while it runs. Its own System / Music / Chat inputs, buses A1/A2 (hardware) and B1/B2 (virtual mics). |
| qpwgraph | `apps.nix` | Hand-made wires; patchbay with Auto pin on, Exclusive off. |

**Live graph read on 2026-11-09** (`wpctl`, `pw-dump`, `pw-link -l`; read only, nothing changed):
- Defaults: output = Ryzen HD Audio analog (`alsa_output.pci-0000_0e_00.6.analog-stereo`),
  input = Razer Seiren Mini.
- linuxmeeter **not running** (no `lm.*` nodes). EasyEffects and qpwgraph running.
- Output path: Plexamp, `virtual_game_out`, `virtual_music_out`, `virtual_chat_out` → `easyeffects_sink`
  → EasyEffects (spectrum + level meter only) → Ryzen analog. So the loopback sinks feed EasyEffects,
  not the hardware directly. That's two pass-through layers in front of the speakers.
- Mic: Seiren suspended, nothing recording from it. EasyEffects' input chain is spectrum + level meter only.
- linuxmeeter's saved profile (`~/.config/linuxmeeter/profiles/Default.toml`) has three settings that
  break its routing:
  1. The Mic strip captures `alsa_input.pci-0000_0e_00.6.analog-stereo` (motherboard line-in), not the Seiren.
  2. A1 *Speakers* and A2 *Headphones* both target `alsa_output.pci-0000_03_00.1.hdmi-stereo`
     (the monitor's HDMI audio), not the Ryzen jack that's the default.
  3. The Chat strip has B1 (*Stream Mic*) on, so a Discord call on Chat + Stream Mic echoes voices back.

**Idle static (also reported 2026-11-09):** the speakers play static whenever nothing has played
for a bit. Read: `snd_hda_intel` `power_save = 10`, `power_save_controller = Y`; both ALSA outputs
`suspended` with WirePlumber's default 5 s timeout. Fix in `audio.nix` (power saving off, analog
out never suspended); `nix flake check` passes, `nvd diff` adds only `51-analog-no-suspend.conf`.
Switched to on 2026-11-09: `wpctl inspect` shows `session.suspend-timeout-seconds = 0` on the analog
out, and `/etc/modprobe.d/nixos.conf` has the new options, but `power_save` stays 10 until a reboot
reloads `snd_hda_intel`.

**Likely direction (not decided):** use one routing layer instead of four. linuxmeeter was
uninstalled later on 2026-11-09, so its profile problems above no longer apply. What's left is the
loopback sinks + the Plasma volume applet, with EasyEffects either given real effects or dropped
(it currently passes everything through unchanged). An earlier attempt at the
first option was reverted the same day at Olivia's request. A walkthrough of the current setup
(diagrams, qpwgraph basics) was given as a separate page, not in the repo.

## Obsidian vault (2026-11-09)

Checked before switching:
- All 19 plugins and 6 themes resolved to the exact versions in the Windows vault; `nix build
  .#obsidian-plugins` built, and all 65 files are **byte-identical** to the Windows ones (after
  `main.js` gets the same source-map strip + `/* nosourcemap */` marker Obsidian's installer adds;
  without it, calendar and OA-file-hider differed by those 18 bytes, and dataview by its ~1 MiB
  inline source map).
- `nix flake check` passes; the system builds; `nvd diff` against the running system adds only the
  Obsidian files and packages (+17.4 MiB). The generated `appearance.json` and
  `community-plugins.json` match Windows (minus obsidian-git and the uninstalled `realclaudian`).
- The seed step, run against a scratch folder: copies missing files (private ones at 600, repo
  ones at 644), leaves existing `data.json` alone, skips a missing private folder.
- No secrets in `home/olivia/obsidian/` (Paperless and LiveSync files left out; checked for
  token/password/passphrase keys and personal paths). The repo is public.
- `update-time-on-edit` only reacts to `modify` events; the first pull into an
  empty vault creates files, so it shouldn't rewrite `updated:` on every note. Windows runs the
  same pair already.

After `nh os switch` (2026-11-09): LiveSync waited for *Fetch from remote* on first run (no
local database until then); after that Olivia confirmed the vault, plugins and settings work.

## VS Code extensions (2026-11-09)

- All 42 Windows extensions resolved on the marketplace at their Windows versions, plus nix-ide.
  Each downloaded VSIX's `package.json` id and version match, and the six platform-specific ones are
  `TargetPlatform="linux-x64"` builds. `nix build .#vscode-marketplace` builds all 43.
- `nix flake check` passes; the system builds. `nvd diff`: 42 extensions added, 5 updated to the
  Windows versions, +583 MiB. Removed with nixpkgs' Python/debugpy/Pylance packages: `gdb`,
  `jedi-language-server` and its Python deps; the marketplace builds bundle their own.
- Two can't work here: Custom CSS and JS and Custom UI Font write into VS Code's install directory,
  which is the read-only store.
- Switched to on 2026-11-09: `~/.vscode/extensions` holds home-manager's links to the declared set.

## Login screen (2026-11-09)

- Fedora's SDDM setup read from the Fedora drive: `/etc/sddm.conf` sets `Current=silent` (it wins
  over `sddm.conf.d/kde_settings.conf`'s `01-breeze-fedora`), the theme checkout is upstream
  `e334d92` with `metadata.desktop` pointing at `configs/rei.conf`, and `git diff` there shows 16
  changed settings plus the untracked `backgrounds/rei2.png`. Each one still exists in upstream's
  current `rei.conf`, and the font sections are the same set.
- Built (`nix build .#nixosConfigurations.desktop-olivia.config.programs.silentSDDM.package'`): the
  theme's `rei.conf` ends with all the overrides, `backgrounds/rei2.png` is there, `metadata.desktop`
  uses `configs/rei.conf`. Evaluated: `sddm.theme = "silent"`, Wayland with KWin, the
  `GreeterEnvironment`/`InputMethod` settings, and tmpfiles rules linking
  `/var/lib/AccountsService/icons/olivia` to the avatar.
- `nix flake check` passes; the system builds. `nvd diff`: adds the theme, `test-sddm-silent`,
  `avatar.png`, Red Hat Display, and Qt/GStreamer dev outputs; +92.6 MiB.
- Both images were added with `nix store add-file`; their hashes are in `desktop.nix`.

## Power and GTK theme (2026-11-09)

- Before: no `powerdevilrc` at all (Plasma defaults; no suspend in the journal for 4 days). GTK
  theme unset in all three places (`settings.ini` has no `gtk-theme-name`, dconf `gtk-theme` empty,
  xsettingsd `Net/ThemeName ""`), so GTK3 apps used Adwaita. `kconf_updaterc` shows `gtkconfig.upd`
  with only `dont_use_gtk_css_for_window_decorations` done: the `gtk_theme` migration that sets
  Breeze never ran. kde-gtk-config 6.6.6's `applyAllSettings()` doesn't set the theme, so values
  written by the config stay.
- plasma-manager's generated `data.json`: `powerdevilrc` `[AC][Display]` dim 900 s, off 1800 s,
  off when locked 300 s, `[AC][SuspendAndShutdown] AutoSuspendAction=0`; `gtk-3.0/settings.ini`
  `[Settings] gtk-theme-name=Breeze`. home-manager `dconf.settings` has
  `org/gnome/desktop/interface` `gtk-theme = "Breeze"`.
- `nix flake check` passes; the system builds. No package changes from these two.

## Windows settings sweep (2026-11-09)

Read-only pass over the Windows profile (`AppData`, `scoop/persist`, dotfiles, `Documents`) for apps
that run natively here, compared against `~` on NixOS. The app list itself was already complete
(MIGRATION.md); this looked for settings, plugins and extensions.

- **Declared:** mpv's `mpv.conf` + `input.conf`; plex-mpv-shim's `mpv.conf` (on NixOS it was an empty
  file); OBS plugins Background Removal and Scale to Sound (Image Reaction isn't in nixpkgs);
  Chromium extensions Bitwarden and Claude (from Helium); the OpenRGB **lowpink** colors as an
  `openrgb-colors` oneshot (MODULES `services.nix`, `mpv.nix`, `apps.nix`).
- **OpenRGB:** the Windows profile's controller locations are `I2C: PawnIO SMBus PIIX4 0, address
  0x18/0x1A` and `HID: \\?\HID#VID_…` paths; here `openrgb --nodetect --list-devices` shows
  `I2C: SMBus PIIX4 adapter port 0 at 0b00 (/dev/i2c-8)` and `/dev/hidraw0`/`hidraw3`. Same four
  devices (2× Corsair DDR5, B650M-K, Basilisk), same serials. Without `--nodetect` the CLI listed
  each device twice (its own detection plus the server's).
- **Hand steps** (logins or app-owned state, [OPERATIONS.md](OPERATIONS.md#settings-copied-from-windows-by-hand)).
  Done on 2026-11-09, with neither app running:
  - plex-mpv-shim `conf.json`: `fullscreen` false, `subtitle_size` 75, `transcode_kbps` 20000,
    `shader_pack_subtype` "hq". Previous file kept as `conf.json.bak-2026-11-09`.
  - Pear Desktop `config.json` copied, mode 600 (5 plugins enabled + Last.fm). It has no Windows
    paths in it.
  - Claude Code: `claude plugin marketplace add cloudflare/skills` +
    `claude plugin install cloudflare@cloudflare` (1.0.4, user scope, enabled); `enabledPlugins` and
    `extraKnownMarketplaces` in `~/.claude/settings.json` now match Windows.

  Still to do: Prism instances (~16 GiB), optionally Blender's `userpref.blend`.
- **Checked, nothing to carry:** Sublime Text (no packages, empty `Packages/User`), Legcord (stock
  Equicord; its plugin settings are in Local Storage and sync through Equicord's cloud), Godot
  (default editor settings, no export templates or themes), qBittorrent (no categories, 3 torrents),
  Ollama (no models), Claude Desktop (no MCP servers), PowerShell profile (empty), user fonts (none
  beyond TX-02), `.gitconfig` (only the email and Windows-only entries).
- **Still installed on Windows but not in MIGRATION.md:** Moosync, Houdoku, ImageGlass, Wabbajack;
  plus a `Calibre Library` folder (174 authors) with no Calibre install found. Not decided.
- `nix flake check` passes; the system builds. `nvd diff` against the running system (which also
  includes the not-yet-switched SilentSDDM work): adds `wrapped-obs-studio` 32.1.2,
  `obs-backgroundremoval` 1.3.7, `obs-scale-to-sound` 1.2.5, the mpv and Chromium config files, the
  `openrgb-colors` unit, and Qt/GStreamer/ODBC dev outputs; +178.7 MiB in total.

## Any-machine deploy (2026-11-10)

The restructure for new machines (DESIGN §12) and `scripts/bootstrap.sh` / `bootstrap.ps1`. Checked
on the desktop, without activating anything and without sudo:

| Check | Result |
|---|---|
| Desktop `drvPath` before/after | Changed, by order only. `nix-diff` finds three leaf derivations: `home-manager-path` and `man-paths` (`chosenOutputs`, same entries in a new order) and `olivia-fish-completions` (the same 68 paths in a new order). Everything above them differs only through those. Cause: the apps now come in through `home/linux`, so `plasma.nix`, `flatpak.nix` and `warp-gui.nix` contribute their packages later. The Windows mount, the clock, OpenRGB, ROCm Ollama and the login art evaluate as before. |
| Desktop build + `nvd diff /run/current-system` | Built. "No version or selection state changes", closure 3518 → 3518 paths (12 swapped: the reordered lists), +0 B. |
| Mac `drvPath` before/after (evaluated on Linux) | Changed. `nix-diff`: `system-path` and `system-applications` (`chosenOutputs`, same entries, new order: `common.nix` is now an import), and `darwin-version.json`, whose `configurationRevision` is now `f77a47a…-dirty`, as with any uncommitted tree. Nothing else. |
| `nix flake lock` | Added only `nixos-wsl` (`e45b24b`, 2026-10-26) and its `flake-compat`; no other input moved. |
| `nix flake check` | Passes in 26 s (15 s before). `checks.aarch64-darwin.toplevel-eval` also evaluates from Linux. |
| New targets | Evaluate: `nixosConfigurations.wsl` and all six `homeConfigurations`. `olivia@x86_64-linux` **built**: 19 new derivations (home-manager's own files and `non-nixos-gpu`); every app is a path the desktop already had. Its `home-path/bin` has 141 commands, including `code`, `obsidian`, `blender`, `davinci-resolve`, `wezterm`, `nvim`, `fish`, `home-manager` and `non-nixos-gpu-setup`. |
| `boot.nix` autoScrub | true on the desktop, false in its `vmVariant` and on an adopted ext4 install. |
| bootstrap.sh generators, in a scratch copy of the repo | `pick_disk` lists the three disks (Philips USB, Samsung, Fanxiang) with bus, model and filesystems and returns the by-id path; `gpu_modules` → `amd`; `host_disks desktop-olivia` → the Samsung by-id; `nixpkgs_release` → `26.05`. It wrote four hosts: a new AMD PC, a new NVIDIA PC, an adopted ext4 install with its ESP at `/boot/efi`, and a new Mac. All four evaluate. New PCs mount by partition label (`disk-main-root`, `disk-main-ESP`) with the swapfile. The AMD one gets `ollama-rocm`, the NVIDIA one kernel 6.18 + driver 595.71 and CPU Ollama, the adopted one `efiSysMountPoint = /boot/efi` and no scrub, and none of them gets the Windows mount or the local-time clock. |
| Generated files vs nixfmt | `nixfmt --check` passes on all four, and on disk ids of eight lengths around the 100-column limit (the long ones are wrapped as nixfmt does). |
| shellcheck (bootstrap.sh) | Clean. It found one real bug, fixed before the first run: `curl … \| sh … </dev/tty` would have fed the terminal to `sh` instead of the Nix installer. |
| PowerShell parser (bootstrap.ps1, pwsh 7.6) | No errors. |

**Not tested:** none of the target machines exist yet, so these have only been evaluated. They're
in the table below: a real USB install of a new PC, adopting an installed NixOS, an NVIDIA GPU,
NixOS-WSL on Windows (bootstrap.ps1 has only been parsed), a new Mac through the script, and
standalone home-manager on another distro (activation and `non-nixos-gpu-setup`).

## tidal-hifi blank window (2026-11-12)

Symptom: tidal-hifi 7.0.1 opened a blank window and KDE showed a crash dialog. `coredumpctl list tidal-hifi`
had SIGTRAP dumps of `tidal-hifi --type=zygote` at every launch since the first one on 2026-11-09. The dumps
had no usable backtrace (stripped binary, child in its own PID namespace).

- A scratch `--user-data-dir` crashed the same way, so it isn't the profile or its GPU caches.
- `--disable-gpu` crashed the same way too, so it isn't the GPU or Mesa. No other Electron app had dumps.
- Every run logged `Unable to access(W_OK|X_OK) /dev/shm: No such process (3)`. `/dev/shm` itself is
  fine (tmpfs, 1777). This matches upstream #958 and the explanation in PR #1007 and #1018: the
  `disableSandbox` setting appends `no-sandbox` after the sandboxed zygote has started.
- `tidal-hifi --no-sandbox` with a scratch profile: no crashes, no `/dev/shm` error.
- Later the same day it was stuck on the logo at every start. Over the remote-debugging port, the page's
  `GET /v1/users/<id>?countryCode=…` failed with `net::ERR_BLOCKED_BY_CLIENT`. tidal-hifi's **Ad block**
  setting (`adBlock` in `~/.config/tidal-hifi/config.json`) cancels every URL matching
  `/\/users\/.*\d\?country/` (`ts-dist/main.js` in 7.0.1), and the player won't start without that
  response. Keep Ad block off.
- Exit crashes (KDE's crash dialog): with `--no-sandbox` there were still SIGTRAP dumps on quit
  (the main process and the `--no-zygote-sandbox` zygote), but only while Ad block was on, or when
  the stuck app was killed while debugging. With Ad block off, a normal quit of the real profile
  (14:00) and a SIGTERM quit of a scratch profile left no dumps. The cores are truncated, so there's no
  backtrace. If the dialog comes back, look at `coredumpctl list tidal-hifi` and the
  `journalctl --user` lines just before the dump.
- Testing from a Claude Code shell needs `LD_LIBRARY_PATH` reset first: that shell inherits an unstable
  `alsa-lib` (glibc 2.43), and tidal-hifi then fails with `GLIBC_2.43 not found`. The real launch doesn't have it.

## Plex separate task entry (2026-11-12)

Symptom: starting Plex from its pinned task manager icon opened the window as a second task entry with
the generic Wayland icon.

- A KWin script listing every window (`workspace.windowList()`) showed the Plex window (pid of
  `bin/Plex`) with `resourceClass` and `desktopFileName` both `tv.plex.Plex`. So 1.112.0 runs as a
  native Wayland window, even though its `Plex.sh` exports `DISABLE_WAYLAND=1`.
- nixpkgs installs the launcher as `plex-desktop.desktop` with `StartupWMClass=Plex`, which fits an X11
  window. There was no `tv.plex.Plex.desktop`, so the task manager had no launcher for the window.
- Fix: the wrapper in `apps.nix` installs the same file as `tv.plex.Plex.desktop`. The built system has
  only that Plex launcher (contents unchanged, `StartupWMClass=Plex` kept for X11). `nvd diff`
  against the running system: only `plex-desktop` changes (+1 path, +2.2 KiB).
- Not changed in nixpkgs yet. An upstream fix would rename the desktop item to `tv.plex.Plex`.
- Hand check passed after `nh os switch` (2026-11-12): re-pinned from the app menu, the window opens
  under the pinned icon with no second Wayland-icon entry.

## Taskbar pins (2026-11-12)

- Read from `plasma-org.kde.plasma.desktop-appletsrc`: two panels, one per screen. Screen 0 (DP-1,
  containment 3, icontasks applet 6) had 15 pins; screen 1 (containment 29) had 4.
- Three pins pointed at store paths (`file:///nix/store/…-system-path/…/ai.storyteller.photocraft.desktop`,
  and `termius-app.desktop` and `code.desktop` in a `…-user-environment`). They're declared as
  `applications:<id>.desktop`. Every declared ID has a launcher in the current profile.
- Atomic Agent's pin pointed into `~/.cache/appimage-run/<hash>/`. Left out of the list.
- Read-only dry run of the script's selection over `qdbus … evaluateScript` (`print` instead of
  `writeConfig`): `panel 3 screen=0 icontasks=1`, `panel 29 screen=1 icontasks=1`. So it writes only to
  the main screen's widget.
- Build: the home files have `scripts/4_script_taskbar-launchers.sh` (runs every time, no hash check)
  and `data/taskbar-launchers.js` with the 14 IDs. `nvd diff`: those two files only (+1.8 KiB).
  The wallpaper's desktop script is unchanged, so it still runs only when its own content changes.
- Not run against the live panel yet: see the hand check below.

## Still to check by hand

The Mac's checks are in [MACOS.md](MACOS.md#checked); all of them are done.

| Area | How |
|---|---|
| Desktop after the rename to `nix-config` | ✅ Done 2026-11-14: `NH_FLAKE` is the new path (Plasma's environment too), `~/nixos-config` removed. `nh os switch` (until then `NH_FLAKE` is the old `~/nixos-config`, a symlink to the new folder, so nh still finds the repo). Then log out and back in: session variables come from `/etc/set-environment` once per login, so terminals opened before that keep the old value. Then `echo $NH_FLAKE` → `/home/olivia/nix-config`, and `readlink -f ~/.config/nvim` ends in `nix-config/home/olivia/dotfiles/nvim`. Then `rm ~/nixos-config` (only the symlink) and check that nvim, WezTerm and fastfetch still load their configs. nix-diff before the switch: only the checkout path changes (nh's variable, the dotfile, font, theme and wallpaper links, the login-art message). |
| Desktop after the any-machine restructure | ✅ Checked 2026-11-14: `openrgb-colors` succeeded, `/mnt/windows` opens, RTC in local TZ, Rei login screen and a model running on the GPU (Olivia); Ollama's log finds the RX 7800 XT through ROCm (gfx1101). `nh os switch`. The diff lists no packages added or removed (order only, above). Then: `systemctl status openrgb-colors` ran, `/mnt/windows` opens, `timedatectl` shows `RTC in local TZ: yes`, `ollama ps` still uses the GPU, the login screen still has the Rei background. |
| tidal-hifi `--no-sandbox` wrapper | ✅ Checked by Olivia 2026-11-14; `--no-sandbox` on the process, no coredumps since the switch. `nh os switch`, quit tidal-hifi (tray too: `pkill -f share/tidal-hifi`) and start it again. The TIDAL home page loads, a track plays, and `coredumpctl list tidal-hifi --since -5min` shows no new dumps. `pgrep -af tidal-hifi` shows `--no-sandbox` on the main process. |
| Taskbar pins | ✅ Checked by Olivia 2026-11-14 (14 pins, Atomic Agent gone). `nh os switch`, then log out and back in (or run `~/.local/share/plasma-manager/scripts/4_script_taskbar-launchers.sh` to apply them now). The main screen shows the 14 pins in order with their icons, and Atomic Agent's pin is gone. The second screen keeps its 4. Then pin or unpin something on the main screen and log in again: it's back to the 14. |
| krabby greeting (2026-11-13) | ✅ Checked by Olivia 2026-11-14. `nh os switch` (Mac: `darwin-rebuild switch`). Open a new terminal: the greeting shows a Pokémon from generations 1–4 with its name, never a mega, gigantamax or regional form. `which pokemon-colorscripts` finds nothing. Shinies show up about 1 in 128 times. |
| Mac after the restructure | ✅ Switched by Olivia 2026-11-14, together with the rename below. `git pull`, `sudo darwin-rebuild switch --flake ~/nix-config#macbook-olivia`. Nothing should change except the revision; `darwin-version --configuration-revision` shows the new commit. |
| Mac after the rename to `nix-config` | ✅ Done by Olivia 2026-11-14: moved, remote updated, switched, symlink removed. Before pulling: `mv ~/nixos-config ~/nix-config && ln -s nix-config ~/nixos-config`, then `cd ~/nix-config && git remote set-url origin https://github.com/itripley/nix-config.git && git pull`, then `sudo darwin-rebuild switch --flake ~/nix-config#macbook-olivia`. Check that `readlink ~/.config/nvim` points into `nix-config`, then `rm ~/nixos-config`. Without the move, the switch would point the dotfile links at a folder that doesn't exist. |
| A new PC from the USB installer | Run `bootstrap.sh` on the installer. The USB is marked `(this installer)`, the typed confirmation names the right disk, it installs and boots, and `hosts/<name>/` is committed and pushed afterwards. Best first done in a VM (the installer ISO in QEMU/virt-manager with UEFI firmware and a spare virtual disk). |
| Adopting an installed NixOS | On a throwaway install (VM): the script writes `hosts/<name>/` with the existing filesystems, `nixos-rebuild boot` works, it boots into systemd-boot, and `sudo passwd olivia` from the old account lets olivia log in. |
| NVIDIA | On an NVIDIA PC: the driver builds for the default kernel, Plasma starts on Wayland, `nvidia-smi` works. Older-than-Turing cards need `hardware.nvidia.open = false`. |
| NixOS-WSL | On Windows: `bootstrap.ps1` installs the distro, the first build and the restarts make olivia the default user, `~/nix-config` is there, a GUI app (e.g. `wezterm`) opens through WSLg, and later `nh os switch` works. |
| A new Mac | `bootstrap.sh` in Terminal: Command Line Tools, Nix and Homebrew are installed, `hosts/<name>/darwin-configuration.nix` is written for the logged-in user, and the switch installs the casks and App Store apps. |
| Another distro | On Fedora or Ubuntu (x86_64): the standalone switch works with `-b hm-backup`, `non-nixos-gpu-setup` runs, and a GUI app (WezTerm, Obsidian) starts with GPU rendering. |
| git author name (after `nh os switch`) | ✅ Checked by Olivia after the 2026-11-10 switch. The running system is exactly the `8aeee79` build. `git config user.name` prints `olivia` (from `home/shared/git.nix`; before this, git used the account name `Olivia`). `nix-diff` before the switch: only the home-manager git config changes. |
| Desktop after the shared-module move (2026-11-10) | ✅ Switched and rebooted 2026-11-10; Olivia checked the fish greeting, git config, `ssh -G unraid` and LazyVim. `nh os switch`. The diff should list only a few rebuilt derivations, with no package added or removed. Then check that the fish greeting shows a Pokémon, that `git config --list --show-origin` has the email and `credential.helper=libsecret`, that `ssh -G unraid` uses `id_unraid`, and that nvim opens LazyVim. |
| Idle static gone | ✅ After the reboot `power_save` is `0`; `wpctl status` lists the Ryzen analog output as the default sink. Still to do: leave audio paused for a minute and listen. |
| Power (after the switch) | ✅ Checked by Olivia after the 2026-11-10 switch. System Settings → Power Management (AC): dim 15 min, screen off 30 min (5 min when locked), *When inactive: Do nothing*. |
| GTK theme (after the switch, restart GTK apps) | ✅ Checked by Olivia after the 2026-11-10 switch (`Breeze`). `dconf read /org/gnome/desktop/interface/gtk-theme` → `'Breeze'`; a GTK3 app (e.g. the file chooser in a GTK app, or `gtk3-demo` via `nix shell nixpkgs#gtk3`) looks like Breeze in Catppuccin colors, not Adwaita. |
| Login screen (after `nh os switch` + reboot) | ✅ Checked by Olivia after the 2026-11-10 reboot. Looks like Fedora's: Rei background, JetBrainsMono clock, round avatar on the left. Size: if it's bigger or smaller than on Fedora, set `settings.General.scale` (Fedora had 1.25, before the theme applied display scaling). Typing the password and Alt+Shift layout switching work; the session menu offers Plasma. `test-sddm-silent` previews it before the reboot. |
| Plasma shortcuts | System Settings → Shortcuts; adjust `plasma.nix` if a KWin action name differs. |
| ROCm in Resolve / Blender / Ollama | Render with HIP in Blender, run a model and check `ollama ps` shows GPU. |
| Seiren Mini + EasyEffects | Record a test clip. |
| Firefox | Open it: add-ons, Sidebery panels, logins and tabs should all be there. |
| Fedora theme (after `nh os switch` + re-login) | Catppuccin colors, Klassy borders, Lain splash, Bocchi cursor, per-screen wallpapers, animated lock screen (Meta+L). |
| Krohnkite | New windows tile; Meta+H/J/K focus, Meta+Shift+H/J/K/L move. If not, System Settings → Window Management → KWin Scripts → Krohnkite. |
| Vicinae | ✅ After the reboot `vicinae.service` is active, 0 restarts. Still to do: Meta+Space opens it with focus (Catppuccin colors, blurred background); Meta+V opens clipboard history, and Enter on an entry pastes it into the focused window (input server; `ls /run/wrappers/bin/vicinae-input-server`); Alt+Shift switches pt/us. |
| Serein minimum size | Shrink the window below ~836×572 px; check the channel list, chat and composer still lay out without overlapping at narrow widths. |
| Obsidian, Windows side | On Windows, `git status` in the vault shouldn't show a flood of notes changed by the Linux side (update-time-on-edit). |
| VS Code extensions | *Extensions* view lists all 43, none flagged. Open a `.py` file (Pylance + interpreter), a C# project (C# Dev Kit; may need `dotnet-sdk` in `dev.nix`), and the Claude Code panel. If debugpy's *attach to process* needs `gdb`, add `gdb` to `dev.nix`. |
| TX-02 font | ✅ `fc-list` lists 30 TX-02 files. WezTerm (restart it) shows TX-02 with Nerd icons intact in the tab bar and prompt; Obsidian's interface and notes use it. |
| RGB colors (after the switch) | `systemctl status openrgb-colors` succeeded; RAM and board LEDs off, the Basilisk dusty pink. If it ran before the server had all devices, `journalctl -u openrgb-colors` shows which call failed. |
| mpv | Play a file: Japanese audio and English subs are picked, the mouse wheel changes volume, `s` saves a webp to `~/Pictures/mpv-screenshots`, `b` toggles deband. |
| plex-mpv-shim | After the `conf.json` step: play from Plex, check subs are the Windows size and HDR content tone-maps (`target-peak=200`). Pans in a 23.976 fps film are smooth on both monitors; with the shim's player focused, `Shift+I` then `2` shows `Interpolation` on and few or no mistimed frames. (Tested before the fix with the shim's `mpv.conf` on a 24p clip: 0 drops, `vsync-ratio` 3.13 on DP-2, i.e. a 3,3,…,4 repeat pattern.) |
| OBS plugins | Add a source: *Background Removal* and *Scale to Sound* appear in its filters. |
| Chromium extensions | Start Chromium: Bitwarden and Claude install from the Web Store within a minute. |
| `github-artlivia` deploy key | Re-add the public key on GitHub, then `git ls-remote git@github-artlivia:itripley/artlivia.git`. |

After any edit, build before switching: `nh os build` (or `nh os switch`, which stops on errors
before activating). `scripts/verify.ps1` is still useful when editing from Windows.
