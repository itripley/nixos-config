# Design decisions

Each section gives the decision, then **why**, with the alternatives that were rejected.
Facts referenced here are in [INVENTORY.md](INVENTORY.md).

## 1. Dual-boot on the Samsung 850 EVO

**Decision:** install NixOS on the whole Samsung 850 EVO 500GB and leave the Windows NVMe untouched.

**Why:**
- Several daily apps have no Linux version (Adobe CC, FL Studio, Avid, Clip Studio, Voicemeeter,
  some anti-cheat games). Keeping Windows means nothing is lost; see [MIGRATION.md](MIGRATION.md).
- A separate physical disk means no shrinking of the Windows partition and no shared ESP. Windows
  updates can't overwrite the Linux boot loader, and a mistake on one OS can't break the other.
- The Samsung disk had no drive letter, so it wasn't in active use by Windows.

**Rejected:** replacing Windows (loses the Windows-only apps); NixOS-WSL (no desktop, GPU or apps,
just a CLI environment).

**Risk:** the disk is not empty (two old partitions, probably Linux). The install erases it.

## 2. Disk layout (`hosts/desktop-olivia/disko.nix`)

| Choice | Why |
|---|---|
| **disko** (declarative partitioning) | The layout is code: reviewable before it runs, and reproducible if the disk is ever replaced. |
| Disk addressed **by-id with the serial** (`ata-Samsung_SSD_850_EVO_500GB_S3R3NF0J901761H`) | `/dev/sdX` / `/dev/nvmeXnY` names can change between boots. The serial can only ever match the Samsung, so disko cannot hit the Windows NVMe by mistake. |
| **GPT** (replacing the old MBR) | UEFI boot needs an EFI partition; GPT is the norm for it. |
| **Own 1 GB ESP** | Windows' ESP is 100 MB, too small for several NixOS kernels + initrds (each generation can need tens of MB). A separate ESP also keeps the two OSes independent. |
| **btrfs** with `@`, `@home`, `@nix`, `@swap` subvolumes | zstd compression (noticeable on `/nix`), cheap snapshots later if wanted, and `/home` kept separate from the root. |
| `compress=zstd,noatime` | Compression saves space and writes; `noatime` avoids a write on every read. |
| 16 GB swapfile on its own subvolume + **zram** | btrfs swapfiles must not be copy-on-write or snapshotted; a dedicated subvolume (which disko sets up) satisfies that. zram absorbs most swapping in compressed RAM first. |

New PCs get the same layout from `modules/nixos/disk-layout.nix`, with only the disk id set per
host (§12).

## 3. Boot

| Choice | Why |
|---|---|
| **systemd-boot** | Simple and well supported by NixOS; Secure Boot is off, so no signing tools (lanzaboote) are needed. |
| Windows via the **ASUS F8 boot menu** | systemd-boot only auto-detects Windows on its *own* ESP. Adding an entry for the other disk's ESP requires the EFI shell device handle, which can only be read on the real hardware. The option (`boot.loader.systemd-boot.windows`) is documented in the README for later. |
| `time.hardwareClockInLocalTime = true` (desktop only, `hosts/desktop-olivia/machine.nix`) | Windows keeps the hardware clock in local time; Linux defaults to UTC. Without this the clock is an hour off after every switch (in summer). Other machines keep UTC. |
| `linuxPackages_latest` | Newest fixes for RDNA3 graphics and Zen 4. |

## 4. Flake inputs and pinning (`flake.nix`, `flake.lock`)

| Input | Why |
|---|---|
| `nixpkgs` → **nixos-26.05** (stable) | Fewer surprises for a first NixOS system. Stable still gets Firefox and security updates. |
| `nixpkgs-unstable`, exposed as `pkgs.unstable.*` | Only for fast-moving tools where a release-branch version is too old. On the desktop that's `claude-code`; on the Mac it's croc, colima, docker-buildx, kubernetes-helm and ente-cli, kept at least as new as Homebrew had them. One input for both hosts. |
| `home-manager` (release-26.05) | User-level config (apps, dotfiles, Firefox, Plasma) in the same repo, applied in the same rebuild. One input for both hosts: a NixOS module on the desktop, a nix-darwin module on the Mac. |
| `plasma-manager` | Declares KDE settings (theme, keyboard, shortcuts) that would otherwise live in hand-edited `~/.config` files. |
| `disko` | See §2. |
| `nix-flatpak` | Declarative Flatpaks for apps that aren't in nixpkgs. The list is empty since 2026-11-09 (JDownloader and Stremio removed); kept so new ones are one line. |
| `serein` flake ([ViceVerse-cz/Serein](https://github.com/ViceVerse-cz/Serein)) | Native Rust Discord client, not in nixpkgs. Its own flake targets nixos-unstable, so its `nixpkgs` follows our `nixpkgs-unstable` (same channel, no extra copy). Built from source: not in any binary cache. No release tags, so `flake.lock` pins a commit; `nix flake update serein` moves it. Needs a `dontCargoUseSystemLibs` override (VERIFICATION #15, the same failure codex had). |
| `claude-desktop` flake ([tomsch/claude-desktop-nix](https://github.com/tomsch/claude-desktop-nix)) | Anthropic's official Claude Desktop Linux beta isn't in nixpkgs. The flake repackages the `.deb` from Anthropic's apt repo (`downloads.claude.ai`), pinned by hash; its CI bumps it every 6 hours after checking the repo's GPG signature. Its `nixpkgs` follows our `nixpkgs-unstable`. The wrapper runs Electron with `--no-sandbox` (Chromium's setuid sandbox can't work from the store) and Wayland flags. `nix flake update claude-desktop` picks up new versions. |
| `artcraft` flake ([ipeglin/artcraft-nix](https://github.com/ipeglin/artcraft-nix)) | Unofficial packaging of the ArtCraft apps (storytold), not in nixpkgs. Repackages the upstream release AppImages (autoPatchelf, no FHS sandbox), pinned by upstream's sha256, with GitHub/GitLab mirrors as fallback. Its NixOS module builds against our `pkgs`; `nixpkgs` follows ours only so the lock has no extra copy. Only PhotoCraft and FilmCraft are enabled (`desktop.nix`). New releases arrive with `nix flake update artcraft`. |
| `nixpkgs-darwin` → **nixpkgs-26.05-darwin** | The Mac's stable nixpkgs (§11). The `-darwin` channel only advances once Hydra's darwin builds pass, so the Mac gets binary-cache hits. Same release as `nixpkgs`. |
| `nix-darwin` (nix-darwin-26.05) | The Mac's system config (§11). Its `nixpkgs` follows `nixpkgs-darwin`, the pairing its README prescribes. |
| `vicinae` flake ([vicinaehq/vicinae](https://github.com/vicinaehq/vicinae)) | The launcher's own flake, set up per [its NixOS guide](https://docs.vicinae.com/nixos). nixpkgs has the package but no NixOS module: without upstream's module the input server lacks `cap_dac_override` (a `security.wrappers` entry), so paste-to-window and snippet expansion fall back to copying. Its Home Manager module replaces home-manager's `programs.vicinae`. Its `nixpkgs` deliberately does **not** follow ours, so builds match `vicinae.cachix.org` (added to `nix.settings`); the cost is its own nixpkgs and Qt in the closure. `nix flake update vicinae` picks up new releases. |

| `nixos-wsl` flake ([nix-community/NixOS-WSL](https://github.com/nix-community/NixOS-WSL), `main`) | NixOS inside WSL, for Windows machines (`hosts/wsl`, §12). The community project the NixOS wiki points to; its module provides the WSL integration (boot, `/etc/wsl.conf`, the Windows GPU driver, Start-menu launchers). `nixpkgs` follows ours. Added 2026-11-10 at `e45b24b` (2026-10-26). `nix flake update nixos-wsl` moves it. |
| `silent-sddm` flake ([uiriansan/SilentSDDM](https://github.com/uiriansan/SilentSDDM)) | The SDDM theme Fedora used for its login screen; not in nixpkgs. Upstream's own NixOS module (`programs.silentSDDM`) builds the theme against our `pkgs` with a preset, extra backgrounds and setting overrides, and wires up SDDM. Its `nixpkgs` follows ours only so the lock has no extra copy. Pinned at `d179078` (2026-11-04), 86 commits past Fedora's `e334d92`: auto-scaling, a Qt 6.12 video-background fix, the module itself. Every setting Fedora changed still exists there. `nix flake update silent-sddm` moves it. |

Every input except nix-flatpak and Vicinae follows our `nixpkgs` (Serein and Claude Desktop follow
`nixpkgs-unstable`, nix-darwin follows `nixpkgs-darwin`; Vicinae keeps its own for the binary cache,
see above), so there's one package set per channel instead of several copies. `flake.lock` pins exact revisions, so evaluation is reproducible until you choose to update.

**Unfree / insecure:** `allowUnfree = true` (Steam, Resolve, VS Code, Plex, …). One insecure package
is permitted, `openssl-1.1.1w`, because **Sublime Text 4 still links the end-of-life OpenSSL 1.1**.
It's scoped by exact name; remove it together with `sublime4` if you stop using Sublime. Both
settings live in one `nixpkgsConfig` in `flake.nix`, used for stable and `pkgs.unstable` alike. A
blanket `allowUnfree` rather than an `allowUnfreePredicate` list is deliberate: about 20 apps are
unfree, and on a single-user desktop the list would just be upkeep.

## 5. home-manager as a NixOS module

**Decision:** home-manager runs inside the NixOS config (`home-manager.nixosModules.home-manager`)
with `useGlobalPkgs` and `useUserPackages`. On the Mac it runs the same way inside nix-darwin
(`home-manager.darwinModules.home-manager`), with the same settings (`homeManagerModule` in
`flake.nix`).

**Why:** one command (`nh os switch`, or `darwin-rebuild switch` on the Mac) updates system and user config together. They always share
one nixpkgs and its settings (unfree, overlays). `backupFileExtension = "hm-backup"` means a file
already in the way gets renamed instead of failing the activation.

## 6. Dotfiles: live links, not store copies

**Decision:** WezTerm, Neovim and fastfetch configs are linked with `mkOutOfStoreSymlink` into the
checkout at `~/nix-config`, instead of being copied into the read-only Nix store. The links are in
`home/shared/default.nix`, so both machines use the same files from `home/olivia/dotfiles/`. Where
the platforms differ (the WezTerm font size, the fish path), the Lua config branches on the
platform.

**Why:**
- LazyVim writes `lazy-lock.json` into its config directory; a read-only store copy breaks plugin updates.
  The lock is in the repo and shared by both machines, so they run the same plugin versions once it's
  committed and pulled.
- Edits apply immediately, with no rebuild for a colour or keybind tweak.
- The files stay version-controlled in this repo.

**Trade-off:** the repo must live at `~/nix-config` on every machine (`scripts/bootstrap.sh` and
the README's manual installs put it there). It was `~/nixos-config` until 2026-11-10, when the repo
was renamed.
`userChrome.css` is the exception: it's a normal Nix-managed file because it's tiny and changes rarely.

## 7. Desktop: KDE Plasma 6 on Wayland

**Why Plasma:** the closest to the Windows workflow (taskbar, start menu, system tray). It has
built-in replacements for many Windows utilities: KWin tiling (AquaSnap/PowerToys), KRunner
(Raycast), per-app volume (EarTrumpet), Spectacle (ShareX), DDC brightness (Twinkle Tray). KDE
Connect was already in use, and Plasma handles 165 Hz + VRR well on AMD.

Carried over: pt+us layouts from Windows. The look and keys come from the old Fedora install
([FEDORA.md](FEDORA.md)): Catppuccin Macchiato + Klassy, Krohnkite tiling on Meta+H/J/K/L, and
a launcher on Meta+Space (Ulauncher there, Vicinae here), so layouts switch with Alt+Shift. The
login screen is Fedora's too: SilentSDDM's `rei` preset with its edits (`desktop.nix`). (The first version used Breeze Dark and
the Windows komorebi Alt+H/J/K/L binds; Fedora's setup replaced both.)

The panels themselves aren't declared: plasma-manager's `panels` option rebuilds every panel from
scratch at login, so every widget's settings would have to live in the repo. Only the main screen's
taskbar pins are declared, written into the existing panel by a small desktop script at each login
(MODULES `plasma.nix`).

## 8. Apps: same app first, then native replacement, then escape hatches

Order of preference for every Windows app:
1. **The same app from nixpkgs** (most of them).
2. **A native Linux replacement** when the app is Windows-only (e.g. Voicemeeter → PipeWire sinks).
3. **A third-party flake** when the app has a maintained Nix packaging outside nixpkgs (Serein,
   Claude Desktop, ArtCraft, Vicinae; §4). Each one is pinned in `flake.lock`, and its nixpkgs follows ours
   (or our unstable), except Vicinae's, which keeps its own to hit its binary cache. These flakes are outside code that runs at build time, so `nix flake update
   <name>` only after a look at what changed. Prefer flakes that repackage the vendor's own
   release files pinned by hash (Claude Desktop, ArtCraft) or build from source (Serein).
4. **Flatpak** when it exists for Linux but isn't packaged in nixpkgs or a flake. (None at the moment.)
5. **AppImage** (binfmt enabled, so they run directly) for one-off downloads like Helium or chaiNNer.
6. **Stays on Windows**, since it's a dual-boot.

**2026-11-09 cleanup:** 44 apps from the original Windows and Fedora lists were removed at Olivia's
request (Zen, the Flatpaks, most chat clients, the image/audio tools, Wine and the Windows-plugin
tooling, LM Studio, codex, opencode, Warehouse, …; the full list is in MODULES `apps.nix`). Config that
only existed for them went too (the zen-browser input, Meta+F Thunar, Vesktop autostart, libvirtd,
the codex build fix). MIGRATION.md marks each one "removed 2026-11-09", so the Windows → Linux
mapping still shows what used to cover each app.

## 9. Secrets stay out of the repo

The Nix store is **world-readable**, and this repo may be pushed somewhere. So:
- SMB credentials live in `/etc/nixos/smb-secrets` (root, mode 600), created by hand during install.
- SSH private keys are copied from the Windows drive after install, never committed.
- The Firefox profile (logins, cookies) is copied by a script, never committed.
- Obsidian's LiveSync and Paperless settings (`data.json`: CouchDB login, E2EE passphrase, API token)
  are copied by hand into `~/.config/obsidian-seed/` and only from there into the vault. LiveSync's
  own encryption of that file uses a fixed default passphrase, so it counts as plain text. Plugin
  code and the other settings are in the repo, pinned by hash.
- WezTerm wallpapers, the Plasma theme artwork (`themes/art/`) and the TX-02 font (`dotfiles/fonts/`)
  are gitignored. They're third-party art and a licensed font, not secrets, but shouldn't be republished.
  The login screen's art (`themes/art/sddm/`) has to be in the store, because the greeter runs as
  `sddm` and can't read `/home/olivia`. A flake can't see gitignored files, so `desktop.nix` uses
  `requireFile`: the repo holds only the hash, and the file is added once with `nix store add-file`.
  That's fine for art (the store is world-readable, which is why it never works for secrets). Once the
  system uses it, the store path is a GC root like any other.
- **On the Mac,** the same rules apply: SSH keys are copied by hand, and the gh token lives in the
  keychain (`gh auth login`). The Cloudflare tunnel token stays in its LaunchDaemon plist, and the
  VS Code API key in `settings.json`, which isn't managed. None of them is in the repo.
- If a secret ever has to be declarative, the plan is **sops-nix**: one tool for NixOS, nix-darwin
  and home-manager, with only the encrypted file in the repo. Nothing needs it yet.

**Root-equivalent access** (checked in the 2026-11-09 audit):
- `nix.settings.trusted-users` is left at `root` only. Trusting `@wheel` would let anything running
  as olivia become root without sudo's password (by setting a substituter or build hook).
- The `docker` group is root-equivalent too (it can mount `/` into a container). It's **kept on
  purpose**, so Docker works without sudo. Rootless Docker (`virtualisation.docker.rootless`) is the
  alternative if that trade-off ever stops being worth it.
- The root password set by `nixos-install` is kept: `systemd-boot.editor = false` blocks
  `init=/bin/sh`, so root's password is the way into emergency mode short of the USB installer.

## 10. Repo hygiene

- `.gitattributes` forces **LF line endings**. The repo is edited on Windows but run on Linux,
  where CRLF breaks shell scripts.
- Flakes only see **git-tracked files**, so the verify script runs `git add -A` first.
- `hardware-configuration.nix` started as a placeholder that evaluated, so the config could be
  checked before the real hardware scan existed. The install replaced it with the generated one.
- Upstream build failures are fixed in `overlays/build-fixes.nix`, not by commenting packages out,
  so the app list in `apps.nix` stays the source of truth.
- Packages written for this repo live in `pkgs/<name>/package.nix` and reach the package set through
  `overlays/packages.nix`. That way they build on their own (`nix build .#<name>`) and can be
  overridden like any nixpkgs package.
- **Checks instead of full CI builds.** `nix flake check` evaluates the whole system and lints the
  repo (statix, deadnix, nixfmt) in about 15 s. GitHub Actions runs the same on every push, on
  `ubuntu-latest` (desktop) and `macos-latest` (the Mac), since each runner checks its own system. CI
  doesn't build the system: it's 40+ GiB, and unfree apps like Resolve aren't in the binary cache.
  `nix fmt` formats everything except the generated `hardware-configuration.nix`. statix's
  "repeated keys" warning is off (`statix.toml`), so each setting stays next to its own comment.
- **No `system.autoUpgrade`.** Updates are manual (`nh os switch --update`, then commit
  `flake.lock`), so a broken upstream change never lands unattended on a desktop that's also used
  for work.

## 11. macOS host: nix-darwin in the same flake

**Decision:** the MacBook is `darwinConfigurations.macbook-olivia` in this repo, not a separate
repo. Cross-platform home-manager modules (shell, git, editor, CLI) are shared; system and
platform-specific modules stay separate. Details, inventory and progress: [MACOS.md](MACOS.md).

**Why the same repo:** shared modules only stay shared if both hosts evaluate them from one
`flake.lock`. The checks, lint, CI, overlays, `nixpkgsConfig` and this doc set carry over. The cost
is coordination: a change to a shared module also changes the desktop, so every refactor is checked
by comparing the desktop's toplevel `drvPath` before and after. The Mac can cross-evaluate the
x86_64-linux system without building it.

- **Nix installer:** the official NixOS installer ([NixOS/nix-installer](https://github.com/NixOS/nix-installer)),
  checked 2026-11-10. It installs upstream Nix like the desktop, has a real uninstaller and survives
  macOS upgrades.
  - Not Determinate: its installer now installs only Determinate Nix, which needs `nix.enable = false`
    (or its own module), with settings in `nix.custom.conf`. That would manage Nix differently from
    the desktop's `nix.settings`.
  - Not Lix: nix-darwin replaces it with upstream Nix anyway unless `nix.package = pkgs.lix`.
- **nix-darwin manages Nix** (`nix.enable`, the default). The daemon, `nix.conf` and the build
  users come from the config, the same way `nix.settings` works on NixOS. `trusted-users` stays
  root-only, as on the desktop.
- **`nixpkgs-26.05-darwin`**, not `nixos-26.05`, for the Mac: nix-darwin's documented pairing, and
  binary-cache coverage for darwin. home-manager stays one input (release-26.05). As a nix-darwin
  module with `useGlobalPkgs`, it uses the Mac's package set.
- **`system.stateVersion = 7`**, the highest nix-darwin 26.05 supports (its template still says 6).
  Never change it afterwards.
- **`system.primaryUser = "olivi"`.** Since 2025, nix-darwin activation runs as root, and
  user-level options (homebrew, `system.defaults`) apply to this user.
- **Activation needs root:** `sudo darwin-rebuild switch`. As on the desktop, Olivia runs the switch.
- **Touch ID's `sudo_local` stayed hand-written until its own step (9, now nix-darwin managed).** nix-darwin manages
  `/etc/pam.d/sudo_local` by default, with empty contents unless `touchIdAuth` is set. Leaving it
  managed would have blocked the first switch, and it would drop Touch ID if ever overwritten.
- **Homebrew stays for GUI apps and some build tools**, declared with nix-darwin's `homebrew`
  module. Casks keep Spotlight, code signing and the apps' own updaters working; brew/Xcode builds
  expect `/opt/homebrew`. No nix-homebrew: Homebrew itself stays a normal install. Activation never
  upgrades or removes anything (`cleanup = "none"`), so declaring the existing apps changed nothing
  on disk, and removals stay a manual, asked-for step. Formulae that services run by path (node for
  openclaw, cloudflared for the tunnel) stay in Homebrew even though Nix has them.
- **Shared home-manager modules, platform files around them.** `home/shared/` holds what works the
  same on both (CLI tools, fish, git/ssh, Neovim, the dotfile links). `home/olivia/` (desktop) and
  `home/darwin/` (Mac) import it and add their own: gitFull + libsecret and gcc on the desktop; zsh,
  gh credentials and Mac-only tools on the Mac. The usernames differ (`olivia` / `olivi`), so
  usernames and home paths only appear in the per-host files.
- **Nix before Homebrew on PATH.** zsh's profile and fish's init run `brew shellenv` first and then
  put the Nix profiles in front, so Nix's git, node and gh win while Homebrew-only tools stay
  reachable.
- **WezTerm from nixpkgs**, not Homebrew's cask: the repo's config uses options newer than
  Homebrew's 20240203 release, and the desktop runs the same build. It's a plain package, not
  `programs.wezterm`, for the same reason as on the desktop: the config folder is live-linked.
- **macOS defaults: only captured values.** `system.defaults` declares just the settings that
  already differed from the macOS defaults, checked one by one before the switch, so adopting them
  changed nothing. Anything declared is reverted on the next switch if changed in System Settings.
- **Garbage collection with nix-darwin's `nix.gc` and `nix.optimise`, not nh.** nix-darwin has no
  `programs.nh`, and home-manager's only cleans the user's own profiles. The root job is enough:
  as a NixOS or nix-darwin module, home-manager makes no profile generations of its own (only a
  `current-home` GC root; checked in its 26.05 activation script and on the desktop), so
  `nix-collect-garbage --delete-older-than 14d` on the system profile also frees old home
  generations. The 14 days match the desktop, but nix-collect-garbage has no "keep 10" floor: after
  14 days without a switch, only the current generation is left to roll back to.
  - **Weekly `nix-store --optimise` instead of `auto-optimise-store`.** nix-darwin refuses
    `auto-optimise-store` unless Nix has the store-corruption fix (2.32.5 or newer). The Mac's
    2.34.9 has it, but the scheduled job doesn't depend on the Nix version, and it's what
    nix-darwin's own option text suggests.
- **Touch ID taken over by hash.** nix-darwin refuses to overwrite an unknown `/etc` file; the old
  `sudo_local` was declared with its exact `knownSha256Hashes`, so the switch moved it aside safely.
  `pam_reattach` makes Touch ID work in tmux.

## 12. Any machine: a folder per host, three layers, one bootstrap script

**Decision (2026-11-10):** the repo deploys to any new machine. Each machine is a folder in
`hosts/`, which `flake.nix` picks up without being edited: a folder with
`darwin-configuration.nix` is a Mac, one with `configuration.nix` is NixOS. What a machine gets
depends on how much of it the repo can manage:

| Machine | Output | What it gets |
|---|---|---|
| x86_64 PC with NixOS | `nixosConfigurations.<name>` | everything: Plasma, the apps, gaming, services |
| Windows | `nixosConfigurations.wsl` (NixOS-WSL) | NixOS inside WSL: the CLI layer and the Linux apps through WSLg |
| Apple Silicon Mac | `darwinConfigurations.<name>` | nix-darwin: Homebrew apps, macOS defaults, Touch ID |
| any other Linux, a WSL distro, a Mac that can't be taken over | `homeConfigurations."<user>@<system>"` | standalone home-manager: CLI tools and dotfiles, plus the GUI apps on x86_64 Linux |

`scripts/bootstrap.sh` (`bootstrap.ps1` on Windows) works out which case it's in and runs the
steps, including writing the new host's folder.

**Why:**
- **Per-machine files only where the hardware needs them.** A NixOS PC needs its own
  `hardware-configuration.nix` and disk id; WSL needs neither, so one `hosts/wsl` serves every
  Windows machine. The script writes the per-machine files, so adding a machine needs no edit to
  `flake.nix`. The file-name rule is why the MacBook's host file became `darwin-configuration.nix`.
- **What only the desktop has left the shared modules** for `hosts/desktop-olivia/machine.nix`:
  the Windows NVMe mount, the local-time RTC (dual boot), OpenRGB and the lowpink colours. The user,
  Nix settings, nh and nix-ld moved from the desktop's `configuration.nix` to `common.nix`, which
  every NixOS host imports. GPU support is a module a host imports: `amd.nix` (now also ROCm
  Ollama; elsewhere Ollama runs on the CPU) or `nvidia.nix`. The script picks one from the PCI
  vendor ids.
- **The login-screen art is opt-in** (`login-art.nix`): a fresh checkout doesn't have the
  gitignored files, and `requireFile` would fail the build. Without it the `rei` preset shows its
  own video background.
- **New PCs use the desktop's disk layout** (`disk-layout.nix`) with only the disk id set per
  host. The mounts go by partition label, so the installed system doesn't depend on the device
  path. Adopting an already-installed NixOS keeps its filesystems (the hardware config is generated
  with them), and `boot.nix` only scrubs btrfs where there is btrfs.
- **One Linux app layer** (`home/linux/`): apps, dev tools, Firefox, Obsidian, mpv, EasyEffects.
  The desktop, WSL and other distros import the same list, so an app added once reaches all of
  them. Plasma, Flatpak and the WARP tray need NixOS underneath and stay in `home/olivia/`. Adding
  the layer only reordered the desktop's package lists (nix-diff: same contents).
- **GUI apps on other distros through home-manager's `targets.genericLinux.gpu`**, not nixGL. It
  installs Mesa at `/run/opengl-driver` with a single `sudo non-nixos-gpu-setup`, so apps run
  unwrapped, as on NixOS, and no extra input is needed. NVIDIA's proprietary driver there needs
  its exact version and hash per machine (`targets.genericLinux.gpu.nvidia`); that isn't
  automated. The GUI layer is x86_64-only, because so are many of its apps (Resolve, Steam, Plex,
  Termius, Claude Desktop); ARM Linux gets `home/shared/`.
- **Standalone outputs are per user and system.** A flake can't read `$USER` without `--impure`,
  so the usernames are listed (`standaloneUsers`). A new one is a single-word addition.
- **NixOS-WSL for Windows** (the `nixos-wsl` input): no per-machine files, and it's the same NixOS
  module system, so `common.nix` and `home/linux/` apply unchanged. The GPU goes through the
  Windows driver (`wsl.useWindowsDriver`). §1's reason against it still holds for the desktop,
  which has a real Linux disk; this is for Windows machines that don't.
- **The installer path stops twice before wiping.** The disk has to be typed by name, and then
  disko asks too. The whole system is evaluated before anything is touched, and a hostname that's
  already in `hosts/` reinstalls that host with its own disk settings.

**Rejected:**
- One generic NixOS host for every PC (all-hardware initrd, no per-machine files): it can't adopt
  an existing install, every PC would share a hostname, and nothing would record which machines
  exist.
- nixos-facter instead of `nixos-generate-config`: one more tool on the installer for a few
  auto-detected settings. The generated file is what the desktop already uses.
- `disko-install`: it builds in the installer's RAM-backed store, too small for a 40 GiB system.
  `nixos-install` builds straight into the target (`--store /mnt`).
- nixGL: it wraps every GUI app and needs another input (see above).
- Intel Macs: nixpkgs 26.05 is the last release for `x86_64-darwin`, supported until the end of
  2026.

**Limits:** BIOS-only PCs (systemd-boot needs UEFI), hybrid NVIDIA laptops (`hardware.nvidia.prime`
bus ids by hand), non-x86_64 NixOS PCs, and native Windows apps (Nix doesn't run on Windows).
