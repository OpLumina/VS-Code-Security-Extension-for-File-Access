# vmask.sh

A privacy shield for VS Code. Before launching the editor, `vmask.sh` mounts an empty `tmpfs` filesystem over any paths you want hidden, making them appear blank to VS Code (and any extensions, language servers, or plugins running inside it). When you close VS Code the mounts are automatically removed and your original files are restored.

---

## How it works

`tmpfs` is an in-memory filesystem. Mounting it over a directory doesn't delete or move anything, it just shadows the real contents for the lifetime of the mount. The moment the mount is removed, the original files reappear exactly as they were.

---

## Requirements

- Linux or WSL (Windows Subsystem for Linux)
- `sudo` access (required for `mount` / `umount`)
- VS Code installed with the `code` CLI on your `PATH`
- Bash 4+

---

## Usage

```bash
./vmask.sh --mask <filepath> --allow <filepath> <folder/file to open with VS>
```

| Argument | Default | Description |
|---|---|---|
| `--mask FILE` | `.vsignore` (next to script) | File listing paths to mask |
| `--allow FILE` | `.vsallow` (next to script) | File listing paths to exempt from masking |
| `TARGET_DIR` | `.` (current directory) | Directory to open in VS Code |

### Examples

```bash
# Open current directory, using default .vsignore / .vsallow
./vmask.sh

# Open a specific project
./vmask.sh ~/projects/myapp

# Use custom mask and allow files
./vmask.sh --mask ~/dotfiles/my.vsignore --allow ~/dotfiles/my.vsallow ~/projects/myapp
```

---

## Configuration files

Both files use the same format: one path per line. Lines starting with `#` and blank lines are ignored.

### `.vsignore`, paths to mask

```
# Mask your SSH keys
/home/alice/.ssh

# Mask all files matching a glob
/home/alice/secrets/*

# Mask a specific config file
/home/alice/.aws/credentials
```

**Wildcard support:** Lines containing `*` are expanded as shell globs using `compgen -G`. Each matching path is resolved and added to the mask list individually. Wildcards are expanded safely, no `eval` is used, so shell injection via crafted filenames is not possible.

### `.vsallow`, exceptions to the mask list

Paths listed here are never masked, even if they match an entry (including a wildcard entry) in `.vsignore`. The allow-list is processed first so it always wins.

```
# Keep this one key accessible even if .ssh/* is masked
/home/alice/.ssh/allowed_key
```

---

## Path handling

- **Absolute paths** are resolved with `realpath -m`, which normalises `..` and trailing slashes without requiring the path to exist at parse time.
- **WSL Windows paths** (e.g. `C:\Users\alice\secret`) are automatically converted to their Unix equivalents via `wslpath -u` before processing.
- Paths that don't exist on disk at mount time are skipped with a notice rather than causing an error.
- Paths that are already mounted (e.g. from a previous session that didn't clean up) are detected and skipped rather than double-mounted.

---

## Session lifecycle

1. Allow-list is loaded from `.vsallow`.
2. Mask list is built from `.vsignore`, with wildcard expansion and allow-list filtering applied.
3. `tmpfs` is mounted over each path in the mask list.
4. VS Code is launched with `code --wait`, the script blocks until the editor window closes.
5. On exit (including `Ctrl+C` or unexpected termination), a `trap` fires and unmounts all paths **in reverse order**, ensuring nested mounts are removed cleanly.

---

## Exit codes

`vmask.sh` forwards VS Code's exit code to the calling shell, so it can be used in scripts:

```bash
./vmask.sh ~/project || echo "VS Code exited with an error"
```

---

## Caveats

- **Sudo prompt:** The first `mount` call will trigger a `sudo` password prompt if your session has expired. You can avoid repeated prompts by running `sudo -v` before launching the script.
- **Nested paths:** If you mask a parent directory and a child directory, mount ordering matters. The current implementation mounts in hash-map iteration order (non-deterministic). If ordering is critical for your setup, list paths explicitly rather than relying on wildcards to capture both parent and child.
- **Does not affect processes already running:** If VS Code or a language server is already open with a file loaded, masking its source path won't unload it from memory.
- **tmpfs contents are writable:** VS Code can write new files into a masked directory during the session. Those files disappear when the mount is removed. This is intentional, it means no writes can leak back to the real path, but be aware if an extension tries to cache something there.
