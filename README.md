# claude-terminal-setup

A tmux + Ghostty setup for running several Claude Code sessions side by side, on Linux.

## What's in here

- `tmux.conf`, goes to `~/.tmux.conf`
- `ghostty.config`, goes to `~/.config/ghostty/config`
- `bin/claude-fleet`, launches one tmux window per git worktree, each running `claude`
- `bin/ghostty-theme-cycle`, rotates the Ghostty theme forward or backward

## Install

```
cp tmux.conf ~/.tmux.conf
mkdir -p ~/.config/ghostty
cp ghostty.config ~/.config/ghostty/config
cp bin/claude-fleet bin/ghostty-theme-cycle ~/.local/bin/
chmod +x ~/.local/bin/claude-fleet ~/.local/bin/ghostty-theme-cycle
```

Requirements: `tmux` 3.3 or newer (for the clickable pane bar, needs `range=pane` support), Ghostty, `xdotool` (for the theme cycle shortcut to apply itself without a manual reload).

On Ubuntu, tmux in the default apt repos is often older than 3.3. Install a current build instead:

```
sudo snap install tmux --classic
sudo apt remove tmux
```

Removing the apt package matters: if both are installed, whichever comes first on PATH wins, and scripts calling plain `tmux` can end up talking to the wrong binary.

Ghostty and xdotool:

```
sudo snap install ghostty --classic
sudo apt install xdotool
```

Edit the `command =` line in `ghostty.config` to point at whatever directory you want Ghostty to drop you into by default.

## tmux keybindings

Prefix is `Ctrl-a`.

| Key | Action |
|---|---|
| `Ctrl-a \|` | vertical split, plain shell |
| `Ctrl-a -` | horizontal split, plain shell |
| `Ctrl-a \` | vertical split, launches claude |
| `Ctrl-a _` | horizontal split, launches claude |
| `Alt+v` | vertical split, plain shell, no prefix needed |
| `Alt+s` | horizontal split, plain shell, no prefix needed |
| `Alt+Ctrl+v` | vertical split, launches claude, no prefix needed |
| `Alt+Ctrl+s` | horizontal split, launches claude, no prefix needed |
| `Alt+Left/Right/Up/Down` | move between panes |
| `Alt+H` / `Alt+L` | previous/next window |
| `Ctrl-a c` | new window |
| `Ctrl-a T` | tile all panes into a grid |
| `Ctrl-a V` | stack all panes in one column |
| `Ctrl-a N` | name the current pane |
| `Ctrl-a r` | reload the config |
| `Alt+t` | cycle to the next Ghostty theme and apply it |
| `Alt+Shift+t` | cycle to the previous theme and apply it |

Mouse:

- click a pane to focus it
- double click a pane's border to rename it
- click a window's name in the top status line to switch to it, double click to rename it
- in the second status line (the pane bar): click an entry to jump to that pane, double click to rename it, right click to kill it after confirming

## claude-fleet

```
claude-fleet <session-name> <worktree-path> [<worktree-path> ...]
```

Creates one tmux window per worktree given, named after the worktree's directory, each already running `claude` in that directory. Run it again with the same session name to jump back into it instead of recreating anything.

## Why two status lines

tmux only shows one window's content on screen at a time, so its top bar can only ever list windows, not panes, since panes from other windows aren't visible anyway. To also see and manage the panes inside the current window, this config adds a second status line underneath: same idea as the window bar, but scoped to whatever panes are currently on screen.

Clicking things in that second line needs a tmux build with `range=pane` support in its status-format syntax, added in tmux 3.3. Older versions can still show the pane bar, it just won't be clickable.

## Notes

- Claude Code sets its own pane title through a terminal escape sequence and will overwrite anything tmux's built-in pane title mechanism sets. Pane names here are stored as a separate tmux user option instead, so renames stick.
- `tmux-resurrect` and `tmux-continuum` are included as plugins so sessions survive closing the terminal and reboots. Install the plugins from inside tmux with `Ctrl-a I`.
