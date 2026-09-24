# claude-terminal-setup

A tmux + Ghostty setup for running several Claude Code sessions side by side, on Linux.

## What's in here

- `tmux.conf`, goes to `~/.tmux.conf`
- `ghostty.config`, goes to `~/.config/ghostty/config`
- `bin/claude-fleet`, launches one tmux window per git worktree, each running `claude`
- `bin/ghostty-theme-cycle`, rotates the Ghostty theme forward or backward
- `bin/tmux-window-bar`, custom-rendered window tab bar, clickable and drag-to-reorder
- `bin/tmux-pane-bar`, custom-rendered pane tab bar, same
- `bin/tmux-pane-bar-move`, `bin/tmux-window-bar-move`, reorder helpers the bars call on drop
- `bin/tmux-bar-drag-update`, `bin/tmux-bar-drag-watchdog`, `bin/tmux-bar-drag-commit`, the drag machinery both bars share

## Install

```
cp tmux.conf ~/.tmux.conf
mkdir -p ~/.config/ghostty
cp ghostty.config ~/.config/ghostty/config
cp bin/* ~/.local/bin/
chmod +x ~/.local/bin/claude-fleet ~/.local/bin/ghostty-theme-cycle ~/.local/bin/tmux-*
```

Requirements: `tmux` 3.3 or newer (for `range=pane` support in status-format, needed for the clickable pane bar), Ghostty, `xdotool` (theme cycling and window geometry), `xinput` (release detection for drag and drop, see below). `xinput` ships with most X11 desktops already; check with `which xinput`.

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

Edit the `command =` line in `ghostty.config` to point at whatever directory you want Ghostty to drop you into by default. Also edit the `continuum_save.sh` path in `tmux.conf`'s `status-format[0]` line if `~/.tmux/plugins` is not where TPM installs plugins for you.

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

## Mouse

Both bars (window tabs on top, pane tabs underneath) work the same way:

- click an entry to switch to it
- double click to rename it
- right click a window tab for a menu (kill, rename, swap, respawn); right click a pane tab to kill it after confirming
- click and drag an entry onto another to reorder: a marker shows exactly where it will land, insert-before-this-item on the left half of the target, insert-after on the right half, past the last item to append at the end

Dragging a pane pill onto the window bar (or the reverse) does not cross-contaminate: whichever drag is actually in progress keeps driving its own bar, and the marker freezes rather than jumping to a nonsensical position while you're over the other bar's row.

## claude-fleet

```
claude-fleet <session-name> <worktree-path> [<worktree-path> ...]
```

Creates one tmux window per worktree given, named after the worktree's directory, each already running `claude` in that directory. Run it again with the same session name to jump back into it instead of recreating anything.

## Why two status lines

tmux only shows one window's content on screen at a time, so its top bar can only ever list windows, not panes, since panes from other windows aren't visible anyway. To also see and manage the panes inside the current window, this config adds a second status line underneath: same idea as the window bar, but scoped to whatever panes are currently on screen.

## Why both bars are custom scripts, not tmux's built-in window/pane list

tmux's pane iteration order (used by the `#{P:...}` format loop) is fixed at creation time. `swap-pane` swaps which pane's content sits at a given position, but explicitly does not reorder the underlying list, so dragging to reorder never had any effect on tmux's own pane bar no matter what was tried. `tmux-pane-bar` keeps its own ordered list of pane ids in a window option instead, self-healing on every render (drops dead panes, appends new ones), and renders from that. Windows do have a real, mutable, native order (`move-window` genuinely reorders them), so `tmux-window-bar` does not need this workaround, but it is still a custom script so both bars share the same drag machinery and visual style.

## How drag and drop actually works

tmux cannot reliably tell you when a mouse button was released. An app running in a pane (Claude Code included) can have its own mouse mode and swallow the release before any tmux binding sees it, and a release outside the terminal window generates no tmux event at all. Chasing this with more tmux-side event bindings does not converge: `MouseDragEnd1Status` only fires for a release that lands back on the status line, and there is no tmux event for "the button came up somewhere else."

So release detection does not use tmux events at all. `tmux-bar-drag-watchdog` starts once per drag and polls the physical pointer's actual button state directly, via `xinput query-state` against every detected pointer device, roughly every 30ms. This bypasses tmux and any app's mouse mode entirely; it is asking the X server what the hardware is doing, not asking tmux what it thinks happened. The instant the button reads as released, it commits using wherever the marker last showed and cleans up, regardless of where on (or off) screen that release physically happened.

`MouseDragEnd1Status`, `MouseUp1Pane` and `MouseUp1Border` also independently call the same commit logic (`tmux-bar-drag-commit`), so a release that does land somewhere tmux can see is handled instantly rather than waiting on the next poll tick. The state file both paths share means whichever one fires first wins and the other becomes a safe no-op.

One known limitation: if a drag leaves the exact row of the bar it started on and later returns to it, tmux stops delivering drag-motion events for that gesture and never resumes, even once the cursor is back over the bar. The marker freezes at its last position rather than continuing to track the cursor. The drop still lands correctly on release, using wherever it was frozen, so this is a display quirk rather than a functional one. Fixing it fully would mean polling cursor position the same way release detection already does, translating screen pixels into terminal columns, which is a bigger change than the rest of this file represents.

## Notes

- Claude Code sets its own pane title through a terminal escape sequence and will overwrite anything tmux's built-in pane title mechanism sets. Pane names here are stored as a separate tmux user option instead, so renames stick.
- `tmux-resurrect` and `tmux-continuum` are included as plugins so sessions survive closing the terminal and reboots. Install the plugins from inside tmux with `Ctrl-a I`.
