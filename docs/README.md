# Documentation

This repo turns DESKTOP-OLIVIA (Windows 11) into a NixOS 26.05 system that dual-boots alongside the
existing Windows install. Since 2026-11-10 it also manages the MacBook (`macbook-olivia`, nix-darwin
+ home-manager), sharing the cross-platform home-manager modules, and it can set up any new
machine: another PC, a Mac, Windows (NixOS-WSL) or any other Linux (DESIGN §12). These docs record **what** was set up, **how** it works, and **why** each
choice was made, so the config can be maintained without re-deriving any of it.

| Doc | Read it when you want to know… |
|---|---|
| [INVENTORY.md](INVENTORY.md) | what was found on the Windows machine (hardware, disks, network, apps, dotfiles) and how it was gathered. This is the evidence behind every decision. |
| [DESIGN.md](DESIGN.md) | why the overall shape is what it is: dual-boot target, disk layout, boot loader, flake inputs, stable vs unstable, home-manager, repo structure |
| [MODULES.md](MODULES.md) | what each file and setting does, and why it's there |
| [MIGRATION.md](MIGRATION.md) | where every Windows app went: same app, Linux replacement, or stays on Windows |
| [FEDORA.md](FEDORA.md) | what was taken from the old Fedora 43 install (theme, tiling, apps), the conflicts with the Windows-based setup and how each was decided |
| [FIREFOX.md](FIREFOX.md) | how the Firefox profile (extensions + their data, Sidebery, themes, userChrome) is carried over, and why it's done this way |
| [VERIFICATION.md](VERIFICATION.md) | how the config was checked before install, what broke during and after the install and how it was fixed, what was checked on the real machine, and what is still to check by hand |
| [MACOS.md](MACOS.md) | the MacBook (`macbook-olivia`, nix-darwin): what was on it, where each app and setting went, progress, what stays manual |
| [OPERATIONS.md](OPERATIONS.md) | day-to-day use (rebuild, update, add apps, roll back), keyboard shortcuts, VPN toggles (NetBird/WARP), theme upkeep, troubleshooting, and leftover data from removed apps |

Installation steps are in the top-level [README](../README.md); a new machine of any kind is set up
with `scripts/bootstrap.sh` ([OPERATIONS.md](OPERATIONS.md#setting-up-a-new-machine)).

## Status (2026-11-10)

- **Renamed to `nix-config` (2026-11-10):** the GitHub repo is `itripley/nix-config` (GitHub
  redirects the old name), and the desktop's checkout is `~/nix-config`, with `~/nixos-config` left
  as a symlink to it until the desktop has switched. The Mac still has to move its checkout:
  [VERIFICATION.md](VERIFICATION.md#still-to-check-by-hand).
- **Any-machine deploy committed and pushed (`eb1d172`, 2026-11-10), not switched yet.** Hosts are folders
  in `hosts/`; new: `hosts/wsl` (NixOS-WSL), standalone home-manager for other distros and locked
  Macs, and `scripts/bootstrap.sh` / `bootstrap.ps1`. Desktop and Mac evaluate to the same
  packages (nix-diff: order only, plus the Mac's revision string). Every target evaluates and the
  standalone Linux home builds. None of the new paths has run on a real machine yet:
  [VERIFICATION.md](VERIFICATION.md#any-machine-deploy-2026-11-10).
- **Review fixes switched on both machines (2026-11-10, `d2fbf68`), and the hand checks passed.**
  The Mac now runs weekly garbage collection and store optimisation, and mise is gone from zsh.
  Both machines use git `user.name` `olivia`. Afterwards, the desktop was running exactly that
  commit's build. Details: [MACOS.md](MACOS.md#checked) step 13.
- **MacBook migrated (2026-11-10):** `darwinConfigurations.macbook-olivia` is switched to, now at
  the review fixes (`d2fbf68`). It covers the hostname, Nix management, home-manager, CLI tools, shell/git/ssh/neovim/WezTerm,
  Homebrew (declared, then cleaned up), macOS defaults, Touch ID sudo and fonts. All the steps are
  done, and CI checks the Mac on `macos-latest`. What stays manual:
  [MACOS.md](MACOS.md#stays-manual).
- **Desktop switched to `main` and rebooted (2026-11-10).** That includes the shared modules
  (`home/shared/`), the Fedora login screen (SilentSDDM), Fedora's power settings, Breeze for GTK apps
  and the Windows settings sweep. Olivia checked the fish greeting, git, ssh, Neovim, the login
  screen, the power settings and the GTK theme. The sweep's own hand checks (mpv, plex-mpv-shim, OBS,
  Chromium, RGB) are still open in [VERIFICATION.md](VERIFICATION.md#still-to-check-by-hand).
- **Installed** on the Samsung 850 EVO and in daily use; `hardware-configuration.nix` is the generated one.
- Every app from the original lists builds. Upstream build failures are patched in
  `overlays/build-fixes.nix` ([VERIFICATION.md](VERIFICATION.md#first-real-build-2026-11-08)).
- Done after first boot: SSH keys, WezTerm (package + wallpapers), Firefox profile migration,
  plex-mpv-shim `allow_http`, Adaptive Sync, NetBird on the self-hosted server (exit node off),
  WARP registered, tray toggles for both, and the old **Fedora** setup (theme, Krohnkite, Ulauncher (now Vicinae),
  apps; [FEDORA.md](FEDORA.md)).
- The 2026-11-09 app changes (PhotoCraft + FilmCraft, Claude Desktop, 44 apps removed) are
  **switched to**; the running system was byte-identical to that commit's build. Their hand checks
  are in [VERIFICATION.md](VERIFICATION.md#app-changes-2026-11-09).
- **Audit fixes (2026-11-09) are committed, booted and in use:** the one-time root steps were done,
  and the kernel command line has `amdgpu.ppfeaturemask`. The hand checks are in
  [VERIFICATION.md](VERIFICATION.md#audit-2026-11-09).
- **Everything after the audit is committed, booted and in use** (`45b44c5`…`cb317e2`: Vicinae,
  Obsidian, VS Code extensions, TX-02, WezTerm tabs, the Serein patch, the idle-static fix). After the
  2026-11-09 reboot the running system was exactly that commit's build, and `power_save` is `0`.
- **The desktop worked in parallel with the Mac on 2026-11-10.** Its uncommitted work went on a
  `desktop-wip` branch, was merged on the Mac (`07f7a8b`) and pulled as a fast-forward. The branch
  is deleted. How to avoid it next time: [OPERATIONS.md](OPERATIONS.md#working-from-two-machines).
- The Windows sweep's hand steps for plex-mpv-shim, Pear Desktop and Claude Code are done; Prism is
  left ([OPERATIONS.md](OPERATIONS.md#settings-copied-from-windows-by-hand)).
- **Open: audio routing doesn't work.** As of 2026-11-09 Olivia can't get the routing she wants out of
  the Game/Music/Chat sinks, EasyEffects, linuxmeeter (since uninstalled) and qpwgraph together. Observed state and next
  steps: [VERIFICATION.md](VERIFICATION.md#audio-routing-not-working-2026-11-09).
- **Obsidian (2026-11-09) works:** LiveSync pulled the vault, plugins and settings in place.
- `vicinae.service` no longer restart-loops: after the reboot it's active with 0 restarts and
  `vicinae ping` answers.
- Open: the `github-artlivia` deploy key is rejected by GitHub; a few hand checks
  ([VERIFICATION.md](VERIFICATION.md#still-to-check-by-hand)); remaining app logins (Steam, Bitwarden, …).

## History

| Date | What happened | Details |
|---|---|---|
| 2026-11-08 | Config written on Windows from an inventory of the machine; evaluated through Docker on Unraid | [INVENTORY.md](INVENTORY.md), [VERIFICATION.md](VERIFICATION.md) |
| 2026-11-08 | Installed from the Philips USB. Build failures forced commenting out apps during the install. | [VERIFICATION.md](VERIFICATION.md#first-real-build-2026-11-08) |
| 2026-11-08 | Failures diagnosed on the installed system: vscode, openssl 1.1 (Sublime), codex, neovim module. Fixed in an overlay; all apps back. Post-install steps done. | `909310b` |
| 2026-11-08 | WezTerm package added (only its config had been linked) | `bb89df5` |
| 2026-11-08 | NetBird pointed at the self-hosted server, restarted on config changes, exit node deselected | `8628c39`, `93865f2` |
| 2026-11-09 | NetBird tray app autostarts (WARP's already did) | `7644365` |
| 2026-11-09 | Old Fedora setup brought over: theme, Krohnkite, Ulauncher (layout switch → Alt+Shift), apps | `0ef7162`, [FEDORA.md](FEDORA.md) |
| 2026-11-09 | Serein (native Discord client) added as a flake input | `5d2f60a`, DESIGN §4 |
| 2026-11-09 | PhotoCraft and FilmCraft (ArtCraft) added through the artcraft-nix flake's module | `3c298f4`, DESIGN §4 |
| 2026-11-09 | 43 apps removed (Zen, the JDownloader/Stremio Flatpaks, most chat, image, audio and Wine tools, codex, opencode, …), plus what only they used: Zen input, Thunar shortcut, Vesktop autostart, libvirtd, codex build fix | `1aba8c9`, DESIGN §8 |
| 2026-11-09 | Warehouse removed | `663b28e` |
| 2026-11-09 | Claude Desktop (official Linux beta) added from the claude-desktop-nix flake | `9be83f4`, DESIGN §4 |
| 2026-11-09 | WARP tray: Cloudflare's Zero Trust-only `warp-taskbar` replaced by warp-cloudflare-gui (consumer on/off) | `cc26af0` |
| 2026-11-09 | Best-practice audit. Fixes: root-only `trusted-users`, no legacy channels, `build-vm` works, LACT overdrive, `nix fmt`/`nix flake check`/`nix develop` + CI, warp-gui moved to `pkgs/`, shared nixpkgs config, no hardcoded uid/path, closure 43.1 → 40.6 GiB (orca/speechd, KDE PIM, extra JDKs, second .NET SDK) | `b96c7e4`…`18fdfd7`, [VERIFICATION.md](VERIFICATION.md#audit-2026-11-09) |
| 2026-11-09 | linuxmeeter (Voicemeeter-style PipeWire mixer) packaged in `pkgs/` and added | never committed, [VERIFICATION.md](VERIFICATION.md#app-changes-2026-11-09) |
| 2026-11-09 | Audio routing reported as not working; live graph recorded, not fixed yet | [VERIFICATION.md](VERIFICATION.md#audio-routing-not-working-2026-11-09) |
| 2026-11-09 | Idle static on the speakers: HDA power saving off, analog output never suspended | `074d8f4`, `audio.nix` |
| 2026-11-09 | linuxmeeter uninstalled (package recipe removed too; it was never committed) | [VERIFICATION.md](VERIFICATION.md#app-changes-2026-11-09) |
| 2026-11-09 | Ulauncher replaced by Vicinae from its own flake + cachix, per its NixOS and KDE guides (Meta+Space, Meta+V clipboard, blur, focus, input server) | `45b44c5`, `plasma.nix`, DESIGN §4 |
| 2026-11-09 | Serein: minimum window size patched from 760×520 to 400×300 (hard-coded, no setting) | `73a10ec`, [VERIFICATION.md](VERIFICATION.md#problems-found-after-first-boot-2026-11-0809) #17 |
| 2026-11-09 | Obsidian: the weblivia vault's 19 plugins and 6 themes packaged (pinned to the Windows versions), settings and snippets declared, LiveSync settings seeded from a private file so it pulls the notes | `b3c127e`, `obsidian.nix`, `pkgs/obsidian-plugins`, [VERIFICATION.md](VERIFICATION.md#obsidian-vault-2026-11-09) |
| 2026-11-09 | VS Code: all 42 Windows extensions (+ nix-ide) declared from the marketplace at the Windows versions, instead of 8 from nixpkgs + Settings Sync | `b3c127e`, [VERIFICATION.md](VERIFICATION.md#vs-code-extensions-2026-11-09) |
| 2026-11-09 | TX-02 font for WezTerm and Obsidian: files in gitignored `dotfiles/fonts/`, linked into `~/.local/share/fonts` | `3d64e44`, `dotfiles/wezterm/config/fonts.lua` |
| 2026-11-09 | WezTerm: fancy tab bar on, with a close × on each tab (`show_close_tab_button_in_tabs = true`) and the title bar kept dark when unfocused; tab titles lost their rounded ends and own colours so the fancy bar draws the tabs | `3d64e44`, `dotfiles/wezterm/config/appearance.lua`, `events/tab-title.lua` |
| 2026-11-09 | Virtual desktops: 3 rows (one column) instead of 1, as changed in the KDE GUI, so the next switch keeps it | `11986c8`, `plasma.nix` |
| 2026-11-09 | Second Fedora sweep (RPM database + config diffs): Fedora's power settings (never sleep, dim 15 min, screen off 30 min) and Breeze for GTK apps added; the rest recorded | `31b3316`, `plasma.nix`, [FEDORA.md](FEDORA.md#second-pass-2026-11-09) |
| 2026-11-09 | Login screen from Fedora: SilentSDDM (`rei` preset, own background, JetBrainsMono, round avatar) via its flake's NixOS module; art pulled in with `requireFile` | `31b3316`, `flake.nix`, `desktop.nix`, [VERIFICATION.md](VERIFICATION.md#login-screen-2026-11-09) |
| 2026-11-09 | Windows settings sweep: mpv and plex-mpv-shim configs, OBS plugins, Chromium extensions, OpenRGB lowpink colors declared; logins and app state left as hand steps | `402a925`, `mpv.nix`, `apps.nix`, `services.nix`, [VERIFICATION.md](VERIFICATION.md#windows-settings-sweep-2026-11-09) |
| 2026-11-09 | Firefox: native sidebar showed next to Sidebery because Firefox 159 renamed `#sidebar-main` to `#sidebar-container`; `userChrome.css` hides both | `31b3316`, `dotfiles/firefox/userChrome.css`, [FIREFOX.md](FIREFOX.md#userchromecss-inactive-on-windows-active-on-nixos) |
| 2026-11-09 | EasyEffects: AirPods Max EQ preset (oratory1990) in `home/olivia/easyeffects.nix` | `31b3316`, [MODULES.md](MODULES.md#homeoliviaeasyeffectsnix) |
| 2026-11-09 | plex-mpv-shim: uneven motion on 24p content fixed with `interpolation=yes` + `tscale=oversample` (23.976 fps doesn't divide 165 Hz or 75 Hz; 0 frames were dropped, the cadence was uneven) | `31b3316`, `plex-mpv-shim/mpv.conf`, [MODULES.md](MODULES.md#homeoliviampvnix) |
| 2026-11-10 | MacBook: Nix installed (NixOS installer), nix-darwin bootstrapped as `macbook-olivia`; repo tooling and checks now also cover aarch64-darwin; desktop drv unchanged | `2256d0e`, [MACOS.md](MACOS.md), DESIGN §11 |
| 2026-11-10 | MacBook: home-manager added for `olivi` (generation 2), HM settings shared with the desktop via `homeManagerModule`; rollback rehearsed | `39c8bbd`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | Shared home-manager modules start: `home/shared/cli.nix` (12 CLI tools moved from `dev.nix`/`apps.nix`, desktop package set unchanged); the Mac gets its own CLI list (`home/darwin/cli.nix`) | `13f0528`, MODULES.md |
| 2026-11-10 | Shared home-manager modules for fish, git/ssh and neovim (desktop package set unchanged). The Mac's zsh, fish, git, ssh and dotfile links moved to home-manager, with Nix ahead of Homebrew on PATH; nixpkgs WezTerm replaces the 20240203 release, which rejected the config | `fa820cb`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | MacBook: Homebrew declared (28 casks, 24 brews, 3 trusted taps, Keynote); activation never upgrades or removes, so nothing changed on disk. Taphouse is no longer in the App Store and stays manual | `286d6f5`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | MacBook: docker uses the Nix buildx/compose plugins (Homebrew plugin dir removed from `~/.docker/config.json`). macOS defaults declared (Dock, Finder, trackpad, clock, Stage Manager); all 34 values matched the live ones, only the Dock list was rewritten | `a252741`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | MacBook: Touch ID for sudo from nix-darwin (`touchIdAuth` + `pam_reattach` for tmux); the hand-written `sudo_local` was recognised by hash and moved aside | `9607781`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | MacBook: JetBrainsMono Nerd Font from nixpkgs (`/Library/Fonts/Nix Fonts`); the 96 hand-installed copies moved to the backup folder | `996c8e0`, [MACOS.md](MACOS.md#checked) |
| 2026-11-10 | README: macOS section (rebuild, update inputs, add a package, roll back, bootstrap a fresh Mac); CLAUDE.md covers the Mac host | [README](../README.md) |
| 2026-11-10 | CI checks both hosts (`ubuntu-latest` + `macos-latest`). MacBook Homebrew cleanup: 21 formulae now from Nix uninstalled (plus 14 dependencies brew autoremoved); node and cloudflared kept for the services that use them | `fff9b9e`, `657592e`, [MACOS.md](MACOS.md#homebrew-cleanup-2026-11-10) |
| 2026-11-10 | MacBook: 7 stale cask records uninstalled (apps already gone). plugdata kept because its audio plugins are still installed. All docs brought up to date for the two-host repo | `20179f6`, [MACOS.md](MACOS.md#homebrew-cleanup-2026-11-10) |
| 2026-11-10 | Desktop work done in parallel (SilentSDDM, EasyEffects module, Plasma power/GTK, Firefox, plex-mpv-shim) committed as `desktop-wip` on the desktop and merged on the Mac. Conflicts in `home/olivia/default.nix` and four docs, kept both sides; desktop package sets identical to the branch (nix-diff: order only) | `07f7a8b`, [OPERATIONS.md](OPERATIONS.md#working-from-two-machines) |
| 2026-11-10 | Desktop pulled the merge, switched and rebooted: shared modules, SilentSDDM login screen, power and GTK settings, Windows sweep all live; hand checks passed; `desktop-wip` deleted | [VERIFICATION.md](VERIFICATION.md#still-to-check-by-hand) |
| 2026-11-10 | Review of the Mac config: everything evaluates, cached and CI-green. Fixes: the Mac now garbage-collects generations older than 14 days and optimises the store every Sunday (`nix.gc`, `nix.optimise`). mise was dropped from zsh. git `user.name = "olivia"` moved to `home/shared/git.nix`, so the desktop stops authoring as `Olivia`. | `d2fbf68`, [MACOS.md](MACOS.md#checked) step 13, DESIGN §11 |
| 2026-11-10 | Both machines switched to the review fixes. Hand checks passed: the two LaunchDaemons are loaded, zsh starts clean, and `git config user.name` is `olivia` on both. The desktop runs exactly the `8aeee79` build. | [MACOS.md](MACOS.md#checked) step 13 |
| 2026-11-10 | Any-machine deploy: hosts discovered from `hosts/` (Mac file renamed `darwin-configuration.nix`); `common.nix` for every NixOS and every Mac host; the desktop's Windows mount, RTC and OpenRGB in `hosts/desktop-olivia/machine.nix`; ROCm Ollama in `amd.nix`; new `nvidia.nix`, `disk-layout.nix`, opt-in `login-art.nix`; `home/linux` app layer; `hosts/wsl` (new `nixos-wsl` input); standalone `homeConfigurations`; `scripts/bootstrap.sh` + `bootstrap.ps1`. Desktop and Mac: same packages (order only) | `eb1d172`, DESIGN §12, [VERIFICATION.md](VERIFICATION.md#any-machine-deploy-2026-11-10) |
| 2026-11-10 | Top-level README rewritten around the machines: what each one is, setting one up, everyday use (switch, check, where things go, update, roll back) for every kind of machine, the manual installs (desktop, Mac, another distro, Windows) and the hand steps per machine. Outdated parts fixed: the Samsung backup warning (it's the NixOS disk now), the 15 s check time, "both machines" | [README](../README.md) |
| 2026-11-10 | Repo renamed `nixos-config` → `nix-config`: GitHub repo, the checkout path in every config (`programs.nh.flake`, the dotfile and font links), scripts and docs. The bootstrap override variables are now `BOOTSTRAP_REPO` / `BOOTSTRAP_DIR`. Desktop: only those paths change (nix-diff); `~/nixos-config` kept as a symlink until both machines switched | [VERIFICATION.md](VERIFICATION.md#still-to-check-by-hand) |
| 2026-11-12 | tidal-hifi blank window fixed: launched with `--no-sandbox` (upstream bug, fixed in 9.0) | [VERIFICATION.md](VERIFICATION.md#tidal-hifi-blank-window-2026-11-12) |
| 2026-11-12 | Plex: window no longer opens as a separate task entry with the Wayland icon; launcher installed as `tv.plex.Plex.desktop` to match the window's app ID (assert from 1.113.0); hand check passed | `a58fa5f`, `apps.nix`, [VERIFICATION.md](VERIFICATION.md#plex-separate-task-entry-2026-11-12) |
| 2026-11-12 | Main screen's taskbar pins declared (14 apps, set at every login by a desktop script; second screen unmanaged, Atomic Agent left out); three pins that pointed at `/nix/store` paths now use their launcher IDs | `plasma.nix`, MODULES.md, DESIGN §7 |
| 2026-11-13 | Fish greeting: pokemon-colorscripts replaced by krabby (generations 1–4, no mega/gigantamax/regional forms, built-in 1/128 shiny chance); both machines | `home/shared/shell.nix`, MODULES.md |

## Conventions

- Windows-side paths are written as they appear on Windows (`C:\Users\olivi\…`). The Windows user
  is `olivi`; the NixOS user is `olivia`. The macOS user is `olivi` too.
- "HM" = home-manager. "nixpkgs" = the pinned `nixos-26.05` branch unless stated otherwise (`nixpkgs-26.05-darwin` on the Mac).
- Comments in the `.nix` files explain the *what* briefly. These docs carry the full *why*.
