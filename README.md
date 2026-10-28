<div align="center">

# dots.sh

**A tiny dotfiles manager in a single bash script.**

Describe your symlinks in `dots.yaml`. Keep them in one repo, many repos, or a mix of both.

</div>

---

## Features

- **Declarative.** One `dots.yaml` says what links where.
- **Safe.** Never overwrites anything. If a destination already exists and isn't the right symlink, you get a warning and the file is left alone.
- **Single or multi repo.** Pull parts of your setup from other git repositories with a `remote:` entry. Those repos bring their own `dots.yaml`.
- **Reproducible.** Remotes are pinned in a `dots.lock` file, like `flake.lock` or `package-lock.json`.
- **Inline files.** Write a small config straight into `dots.yaml` with `content:`, no separate file needed.
- **Targeted.** Update or undo one entry without touching the rest.
- **Dependency-light.** Needs only `bash` (4.3+), `git` (for remotes) and GNU `realpath`/`readlink`. No `yq`, no Python.

## Install

It's a single file. Put it somewhere on your `PATH`:

```sh
install -Dm755 dots.sh ~/.local/bin/dots.sh
```

## Quick start

Create a `dots.yaml` at the root of your dotfiles repo:

```yaml
zsh:
    .zshrc:
        source: "./.zshrc"
        destination: "~/.zshrc"
    .zsh:
        source: "./.zsh"
        destination: "~/.zsh"

nvim:
    source: "./nvim/"
    destination: "~/.config/nvim/"
```

Then, from that directory:

```sh
dots.sh -n     # preview, changes nothing
dots.sh        # create the links
```

```
✓ zsh/.zshrc: /home/you/.zshrc -> /home/you/dotfiles/.zshrc
✓ zsh/.zsh: /home/you/.zsh -> /home/you/dotfiles/.zsh
✓ nvim: /home/you/.config/nvim -> /home/you/dotfiles/nvim
3 linked, 0 up to date, 0 warnings, 0 errors
```

Running it again is harmless: links that are already correct are counted as "up to date".

## `dots.yaml`

Every entry is a `source` and a `destination`. Entries can sit directly under a name, or be grouped under a name:

```yaml
nvim:                         # a single entry
    source: "./nvim/"
    destination: "~/.config/nvim/"

quickshell:                   # a group of entries
    modules:
        source: "./modules"
        destination: "~/.config/quickshell/modules"
    shell.lua:
        source: "/path/to/dotfiles/quickshell/shell.lua"
        destination: "~/.config/quickshell/shell.lua"
```

| Key | Meaning |
| --- | --- |
| `source` | What to link to. Relative paths are resolved against the folder of the `dots.yaml` that contains them. Absolute paths work too. |
| `destination` | Where the symlink goes. `~`, `$HOME` and trailing slashes are fine. |
| `content` | Text to put in the file instead of linking an existing one. See [Inline content](#inline-content). Wins over `source`. |
| `remote` | A git repository to clone. See [Remote repositories](#remote-repositories). |
| `ref` | Optional branch or tag for a `remote`. |
| `post_update` | Optional shell command to run after a `remote` is updated. |

Each entry is shown by its path in the file, such as `zsh/.zshrc` or `quickshell/modules`. Those are the names you pass to `--update` and `--undo`.

### What happens when linking

| Destination is... | Result |
| --- | --- |
| missing | parent folders are created, the symlink is made |
| a symlink to the right source | nothing to do |
| a symlink somewhere else | **warning**, left alone |
| a real file or folder | **warning**, left alone |

If any warning or error was reported, the exit code is `1`.

## Inline content

For small files, skip the separate source file and put the text in `dots.yaml`:

```yaml
git:
    .gitconfig:
        content: |
            [user]
                name = Me
                email = me@example.com
            [core]
                editor = nvim
        destination: "~/.gitconfig"
```

`dots.sh` writes the text to `~/.local/share/dots/.content/<entry path>` (here `.content/git/.gitconfig`) and symlinks the destination to it, exactly as it would for a `source`. Edit the text in `dots.yaml` and run `dots.sh` again: the stored file is rewritten and the existing link keeps working.

- `content` needs a `destination` and no `source`. If both `source` and `content` are set, you get a warning and `content` is used.
- Use a block scalar (`|`) for multi-line text. It follows the usual YAML rules:

  | Style | Result |
  | --- | --- |
  | `\|` | lines kept as written, one trailing newline |
  | `\|-` | same, no trailing newline |
  | `\|+` | same, trailing blank lines kept |
  | `>` / `>-` | lines folded into paragraphs |
  | `"a\nb"` | single line, `\n` `\t` `\"` `\\` are expanded |
  | `'a\nb'` | single line, taken literally |

- A plain one-line value has no trailing newline. Use `|` if you want one.
- `content: ""` creates an empty file.
- The stored files are not deleted when you remove an entry or run `--undo`. The symlink is removed, the file stays in `.content/`.
- Content works inside remote repositories too, and is stored under the remote's name.

## Remote repositories

Point an entry at a git repository instead of a local path:

```yaml
quickshell:
    remote: "github:user/repo"
```

The repository must have a `dots.yaml` in its root. It gets cloned into `~/.local/share/dots/quickshell`, and its `dots.yaml` is processed like any other, with relative sources resolved inside the clone.

Supported remote formats:

| Format | Expands to |
| --- | --- |
| `github:user/repo` | `https://github.com/user/repo.git` |
| `gitlab:user/repo` | `https://gitlab.com/user/repo.git` |
| `codeberg:user/repo` | `https://codeberg.org/user/repo.git` |
| `https://…`, `ssh://…`, `git@host:…` | used as-is |
| `/some/path`, `~/some/path` | a local repository |

Pin a branch or tag with `ref`:

```yaml
quickshell:
    remote: "github:user/repo"
    ref: "stable"
```

### Mixing local and remote

Local entries and remotes can live in the same file, and a remote repository can itself contain further `remote:` entries (up to 8 levels deep):

```yaml
zsh:
    source: "./.zshrc"
    destination: "~/.zshrc"

nvim:
    remote: "github:me/nvim-config"

quickshell:
    remote: "github:me/quickshell-config"
    modules:                     # an extra local link next to the remote
        source: "./quickshell-overrides"
        destination: "~/.config/quickshell/overrides"
```

### The lock file

The first time a remote is cloned, its exact commit is recorded in `dots.lock`, next to your main `dots.yaml`:

```yaml
# dots.lock - generated by dots.sh, do not edit
"quickshell":
    remote: "github:user/repo"
    url: "https://github.com/user/repo.git"
    commit: "94bb49fd2afde0ceec0ab0556b8780c3f57487ef"
```

Commit it. On another machine, `dots.sh` clones each remote at the recorded commit, so you get the same setup everywhere. The lock is only used while the remote's URL and `ref` still match what's in `dots.yaml`.

Plain `dots.sh` never moves a remote forward. Only `update` does.

## Commands

```
dots.sh [options] [command] [names...]
```

| Command | What it does |
| --- | --- |
| `link` *(default)* | Create the symlinks. Clones missing remotes at their locked commit. |
| `update [names]` | Re-download remotes, rewrite `dots.lock`, relink, run `post_update` commands. |
| `status` | Show the state of every link and remote. Changes nothing. |
| `unlink [names]` | Remove symlinks that point at their configured source. |

| Option | What it does |
| --- | --- |
| `-c, --config FILE` | Use this `dots.yaml` (default `./dots.yaml`, or `$DOTS_CONFIG`). |
| `--update NAME` | Same as `update NAME`. Repeatable. |
| `-u, --undo NAME` | Same as `unlink NAME`. Repeatable. |
| `-n, --dry-run` | Show what would happen, change nothing. |
| `-h, --help` / `-V, --version` | Help and version. |

### Updating one remote

```sh
dots.sh --update quickshell
```

This:

1. clones the repository fresh into a temporary folder and swaps it in only if that succeeds. A failed download leaves your current clone untouched,
2. checks out the latest commit (or your `ref`),
3. updates that entry in `dots.lock`,
4. relinks,
5. runs the remote's `post_update` command, if there is one and the commit changed.

Every other remote stays at its locked commit. Names also match everything nested under them. Plain `dots.sh update` updates every remote.

```
re-downloading quickshell (https://github.com/user/repo.git)
quickshell: 456e2bffddd8 -> 94bb49fd2afd
running post_update for quickshell: ./build.sh
```

Re-downloading discards any local edits inside the clone in `~/.local/share/dots/`. Make your changes in the upstream repo instead.

### Post-update commands

Set `post_update` next to a `remote:` in **your** `dots.yaml`:

```yaml
quickshell:
    remote: "github:user/repo"
    post_update: "./build.sh && pkill -USR1 quickshell"
```

The command runs through `bash -c` inside the clone, after the new links are created, and only when an update actually changed the commit. These variables are available to it:

| Variable | Value |
| --- | --- |
| `DOTS_NAME` | entry name, e.g. `quickshell` |
| `DOTS_DIR` | path of the clone |
| `DOTS_OLD_COMMIT` | previous commit (empty on the first update) |
| `DOTS_COMMIT` | new commit |

> **Why only in your file?** A remote repository can't make `dots.sh` run code on your machine just because you updated it. You decide what runs.

If the command fails, you'll see `post_update failed` and the exit code is `1`. The lock has already moved to the new commit by then.

### Undoing one entry

```sh
dots.sh --undo quickshell            # everything under quickshell
dots.sh --undo zsh/.zshrc            # a single entry
dots.sh unlink nvim zsh              # several at once
dots.sh -n --undo quickshell         # preview first
```

Only symlinks that point at their configured source are removed. Real files, links pointing elsewhere, and links that don't exist are skipped. Remote clones and `dots.lock` are left as they are.

### Checking state

```sh
dots.sh status
```

```
✓ zsh/.zshrc: /home/you/.zshrc
✓ nvim: /home/you/.config/nvim
remote quickshell @ 94bb49fd2afd
✓ quickshell/modules: /home/you/.config/quickshell/modules
- quickshell/shell.lua: not linked (/home/you/.config/quickshell/shell.lua)
```

It also warns if a clone is on a different commit than `dots.lock` says.

## Environment

| Variable | Default | Purpose |
| --- | --- | --- |
| `DOTS_CONFIG` | `./dots.yaml` | Config file to use |
| `DOTS_DATA` | `$XDG_DATA_HOME/dots` or `~/.local/share/dots` | Where remotes are cloned |

## Good to know

- The YAML parser is built into the script and understands only the nested `key: value` form shown here. Quotes and `#` comments are fine. Block scalars (`|`, `>`) work for any value, which is what `content` uses. Lists, anchors and other multi-line forms are not supported, and you'll get an "unsupported yaml" notice for such lines.
- Symlinks are created with absolute paths.
- `dots.lock` lives next to the main `dots.yaml`, not inside the clone folder.
- Remote clones are named after their entry (`quickshell/sub` becomes `quickshell-sub`), so two entries pointing at the same repository get separate clones.
