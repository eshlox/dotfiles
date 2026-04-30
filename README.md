# Dotfiles (Fedora VM)

Managed with [chezmoi](https://www.chezmoi.io/). Source tree maps to `$HOME`
via chezmoi's `dot_` prefix convention. Fedora-only — no OS templating.

Identity (name, email, GPG signing key) is **never stored in this repo**.
It lives in each VM's local chezmoi config (`~/.config/chezmoi/chezmoi.toml`),
outside the source tree.

## First-time setup on a fresh Fedora VM

```bash
# 1. Install required packages
sudo dnf install -y zsh tmux helix lazygit fzf git-delta jq just yazi chezmoi

# 2. Pure prompt (zsh)
mkdir -p ~/.zsh && git clone https://github.com/sindresorhus/pure.git ~/.zsh/pure
echo 'fpath+=$HOME/.zsh/pure' >> ~/.zshenv

# 3. tmux catppuccin plugin (referenced from tmux.conf)
git clone https://github.com/catppuccin/tmux.git \
  ~/.config/tmux/plugins/catppuccin/tmux

# 4. Pull dotfiles into chezmoi's source dir
chezmoi init https://github.com/<user>/dotfiles.git

# 5. Set local identity (required — see section below)
chezmoi edit-config

# 6. Preview, then apply
chezmoi diff
chezmoi apply
```

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
signingKey = "ABCDEF1234567890"  # optional — omit if no GPG signing on this VM
```

- `name` and `email` are **required**. Missing either causes `chezmoi apply` to
  fail with a descriptive error.
- `signingKey` is optional. When omitted, `gpgsign` is set to `false` in the
  rendered git config and no `signingkey` entry is written.

### What the template produces

With `signingKey` set:

```ini
[user]
    name       = "Your Name"
    email      = "you@example.com"
    signingkey = "ABCDEF1234567890"
[commit]
    gpgsign = true
```

Without `signingKey`:

```ini
[user]
    name  = "Your Name"
    email = "you@example.com"
[commit]
    gpgsign = false
```

### Verify rendered output before applying

```bash
chezmoi execute-template < ~/.local/share/chezmoi/dot_config/git/common.gitconfig.tmpl
```

### Per-VM GPG signing key (DVM workflow)

Generate a GPG key via DVM, then copy the fingerprint into the local config:

```bash
dvm gpg-key app
gpg --list-secret-keys --keyid-format=long
chezmoi edit-config   # add signingKey = "FINGERPRINT" under [data]
chezmoi apply
```

### What does NOT belong in this repo

- Real name, email, GPG signing key fingerprint
- SSH private keys or authorized_keys
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
| `.config/git/common.gitconfig`         | shared git settings rendered from template       |
| `.config/git/local.gitconfig`          | per-VM extras (NOT tracked, optional)            |
| `.config/helix/config.toml`            | Helix editor                                     |
| `.config/helix/languages.toml`         | per-language formatters                          |
| `.config/just/justfile`                | global recipes (lazygit/hx/helix/yazi)           |
| `.config/lazygit/config.yml`           | lazygit + AI-commit keybinding (`G`)             |
| `.config/tmux/tmux.conf`               | tmux                                             |
| `.local/bin/ai-commit`                 | local LLM commit-message generator               |

`ai-commit` expects a local `llama-server` running on `127.0.0.1:8080`.

## Source layout (chezmoi names)

| Source file                                    | Deploys to                              |
| ---------------------------------------------- | --------------------------------------- |
| `dot_config/git/common.gitconfig.tmpl`         | `~/.config/git/common.gitconfig`        |
| `dot_gitconfig`                                | `~/.gitconfig`                          |
