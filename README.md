# Dotfiles

Managed with [chezmoi](https://www.chezmoi.io/). Source tree maps to `$HOME`
via chezmoi's `dot_` prefix convention.

Two deployment profiles, selected by the chezmoi data field `role`:

| `role`        | Where         | What gets applied                            |
| ------------- | ------------- | -------------------------------------------- |
| `"vm"`        | Fedora VM     | everything **except** `.config/ghostty`      |
| unset         | macOS host    | **only** `.config/ghostty`                   |

The split exists because Ghostty (the terminal) runs on the host, while every
other tool (zsh, zellij, helix, lazygit, etc.) runs inside the VM. The selection
is implemented in `.chezmoiignore` as a template.

Identity (name, email) is **never stored in this repo**. It lives in each
machine's local chezmoi config (`~/.config/chezmoi/chezmoi.toml`), outside the
source tree.

## First-time setup on a fresh Fedora VM

```bash
# 1. Install required packages
sudo dnf install -y zsh helix lazygit fzf git-delta jq just yazi \
                    chezmoi starship bat
# `ghostty` is the terminal — install it from your host package manager
# (https://ghostty.org/) since it runs outside the VM.
# `zellij` isn't packaged for Fedora — grab the latest release tarball from
# https://github.com/zellij-org/zellij/releases and drop the binary at
# /usr/local/bin/zellij.

# 2. Pull dotfiles into chezmoi's source dir
chezmoi init https://github.com/<user>/dotfiles.git

# 3. Set local identity (required — see section below)
chezmoi edit-config

# 4. Preview, then apply
chezmoi diff
chezmoi apply

# 5. Register the Catppuccin Latte theme with bat (and delta)
bat cache --build
```

The bat cache step picks up `~/.config/bat/themes/Catppuccin Latte.tmTheme`,
which both `bat` and `delta` (via syntect) then resolve when configured with
`syntax-theme = "Catppuccin Latte"`. Re-run `bat cache --build` whenever the
`.tmTheme` file changes.

## First-time setup on the macOS host (Ghostty only)

```bash
# 1. Install Ghostty + a Nerd Font
brew install --cask ghostty font-jetbrains-mono-nerd-font

# 2. Install chezmoi
brew install chezmoi

# 3. Pull the same repo (no role data — host profile is the default)
chezmoi init https://github.com/<user>/dotfiles.git
chezmoi diff
chezmoi apply
```

`role` is left unset on the host, so `.chezmoiignore` ignores everything except
`.config/ghostty/config`. Updates to Ghostty config are managed via the same
repo as the VM dotfiles — single source of truth.

## Identity & private data (chezmoi templates)

Git config is a chezmoi template. It requires local data to render — `chezmoi
apply` will fail with a clear error if any required value is missing.

### Set identity on each VM

```bash
chezmoi edit-config
```

This opens `~/.config/chezmoi/chezmoi.toml` (outside the repo). Add:

```toml
[data]
name       = "Your Name"
email      = "you@example.com"
role       = "vm"                                  # optional — set on Lima VMs; omit on host
deployKey  = "~/.ssh/id_ed25519_dvm.pub"           # optional — .pub path; stripped for `ssh -i`
signingKey = "~/.ssh/id_ed25519_dvm_signing.pub"   # optional — .pub path used by git signing
```

- `name` and `email` are **required**. Missing either causes `chezmoi apply` to
  fail with a descriptive error.
- `role = "vm"` — set this on Lima VMs. It adds `/usr/sbin` and `/sbin` to
  `PATH` in `.zshrc` so Lima's port-forwarding (`iptables`) and file-sharing
  (`mount.fuse3`) helpers are discoverable. Lima may inject this block itself;
  owning it via chezmoi prevents it from being treated as an unmanaged mutation.
  Omit `role` (or set any other value) on the host — the block is not written.
- `deployKey` — **optional**. Pass the `.pub` path; the template strips `.pub`
  and wires the private key into `core.sshCommand` with `IdentitiesOnly=yes`,
  so git's SSH transport always uses exactly that key (ssh-agent won't offer
  others). Omit it and git falls back to default SSH behavior.
- `signingKey` — **optional**. Pass the `.pub` path; goes into
  `user.signingkey` and enables `[gpg] format = ssh`, plus `gpgsign = true`
  for commits and tags. Omit it and signing is fully disabled — no `[gpg]`,
  `[commit]`, or `[tag]` sections are written.

All four combinations work cleanly:

| `deployKey` | `signingKey` | Result                                          |
| ----------- | ------------ | ----------------------------------------------- |
| set         | set          | pinned SSH key for transport + SSH signing on   |
| set         | unset        | pinned SSH key, no signing                      |
| unset       | set          | default SSH (agent / `~/.ssh/config`) + signing |
| unset       | unset        | plain git config, no SSH or signing tweaks      |

## SSH keys for git (deploy + signing)

Two independent SSH keys per VM, both optional. You always pass the `.pub`
path in chezmoi data — the template handles the public/private distinction:

| Purpose         | chezmoi data field | Wired into git via               | What template does          |
| --------------- | ------------------ | -------------------------------- | --------------------------- |
| Push/fetch auth | `deployKey`        | `core.sshCommand = ssh -i …`     | strips `.pub` for `ssh -i`  |
| Commit signing  | `signingKey`       | `user.signingkey` + `gpg.format` | uses `.pub` path as-is      |

Use one key for both (point both fields at the same `.pub`) or two completely
separate keys. Requires git ≥ 2.34 for SSH signing.

### One-time setup per VM

1. **Generate keys** (one or two, your choice):
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_dvm         -C "deploy: you@example.com"
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_dvm_signing -C "signing: you@example.com"
   ```

2. **Register on GitHub**:
   - `id_ed25519_dvm.pub` → **Authentication key** (Settings → SSH and GPG keys)
   - `id_ed25519_dvm_signing.pub` → **Signing key** (same page, separate entry —
     GitHub treats auth and signing as distinct purposes even for the same key
     bytes)

3. **Populate the local allowed-signers file** so `git log --show-signature`
   verifies your own commits locally:
   ```bash
   echo "you@example.com $(cat ~/.ssh/id_ed25519_dvm_signing.pub)" \
     >> ~/.ssh/allowed_signers
   ```

4. **Point chezmoi at the keys**:
   ```bash
   chezmoi edit-config   # set deployKey and/or signingKey under [data]
   chezmoi apply
   ```

5. **Verify**:
   ```bash
   ssh -T git@github.com                    # uses deployKey via core.sshCommand
   git commit --allow-empty -m "test sig"
   git log --show-signature -1              # checked against allowed_signers
   ```

### Verify rendered output before applying

```bash
chezmoi execute-template < ~/.local/share/chezmoi/dot_config/git/common.gitconfig.tmpl
```

### What does NOT belong in this repo

- Real name, email
- SSH private keys, public keys, or `authorized_keys`
- `~/.ssh/allowed_signers` (contains your email + public key)
- API keys, tokens, passwords
- Machine-specific paths

These stay in `~/.config/chezmoi/chezmoi.toml` or other local, untracked files.

## Per-VM local git overrides

`~/.gitconfig` includes both the rendered common config and an optional local
override file:

```ini
[include]
    path = ~/.config/git/common.gitconfig   # rendered from template, tracked
[include]
    path = ~/.config/git/local.gitconfig    # NOT tracked, per-VM extras
```

`local.gitconfig` is silently ignored by git if it doesn't exist. Use it for
any machine-specific git settings beyond what chezmoi data covers. If you ever
add it to chezmoi accidentally, run:

```bash
chezmoi forget ~/.config/git/local.gitconfig
```

## Daily use

```bash
chezmoi diff                    # preview pending changes
chezmoi apply                   # apply them
chezmoi add ~/.config/foo/bar   # start managing a new file
chezmoi cd                      # cd into the source dir
chezmoi git -- pull --rebase    # pull updates from another VM
chezmoi git -- push             # push your changes
chezmoi edit ~/.zshrc           # edit the source-tree version
chezmoi data                    # show all template variables (local + built-in)
chezmoi edit-config             # edit ~/.config/chezmoi/chezmoi.toml
```

## Layout (destination paths)

| Path                                   | Purpose                                          |
| -------------------------------------- | ------------------------------------------------ |
| `.zshrc`                               | shell config + `just` wrapper                    |
| `.gitconfig`                           | includes common + local                          |
| `.gitignore_global`                    | OS / editor junk ignored everywhere              |
| `.config/git/common.gitconfig`         | shared git settings rendered from template       |
| `.config/git/local.gitconfig`          | per-VM extras (NOT tracked, optional)            |
| `.config/ghostty/config`               | terminal — owns the color palette for everything |
| `.config/helix/config.toml`            | Helix editor                                     |
| `.config/helix/languages.toml`         | per-language formatters                          |
| `.config/just/justfile`                | global recipes (lazygit/hx/yazi)                 |
| `.config/lazygit/config.yml`           | lazygit + AI-commit keybinding (`G`)             |
| `.config/zellij/config.kdl`            | zellij multiplexer                               |
| `.config/bat/config`                   | bat (better cat) — light theme                   |
| `.config/starship.toml`                | starship prompt — minimal preset                 |
| `.local/bin/ai-commit`                 | local LLM commit-message generator               |

`ai-commit` expects a local `llama-server` running on `127.0.0.1:8080`.

## Source layout (chezmoi names)

| Source file                                    | Deploys to                              |
| ---------------------------------------------- | --------------------------------------- |
| `dot_zshrc.tmpl`                               | `~/.zshrc`                              |
| `dot_config/git/common.gitconfig.tmpl`         | `~/.config/git/common.gitconfig`        |
| `dot_gitconfig`                                | `~/.gitconfig`                          |
