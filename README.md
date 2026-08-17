# ibus-mozc-system

Scripts and X11/XFCE configuration for Japanese input with IBus and Mozc on a keyboard with no dedicated Japanese keys.

Two keys, two axes. Right Ctrl switches hiragana ↔ katakana; Insert switches full-width ↔ half-width. One press
each, from any state; mid-word, mid-conversion, or with Japanese input off.

```text
              full-width        half-width
hiragana      ひらがな          (does not exist)
katakana      カタカナ          ｶﾀｶﾅ
```

Mozc has no half-width hiragana, so from hiragana Insert goes to half-width katakana. The katakana width is remembered
across Right Ctrl.  
It does not replace IBus or Mozc. It fills the gaps between IBus, Mozc, X11 and XFCE.

## Parts

| File                  | Purpose                                                         |
|-----------------------|-----------------------------------------------------------------|
| `kana-switch`         | The kana switch;                                                |
| `kana-toggle`         | `kana-switch toggle` - Right Ctrl                               |
| `kana-width`          | `kana-switch width` - Insert                                    |
| `ibus-mozc-init`      | Brings the IBus/Mozc stack up and supervises it for the session |
| `ibus-health`         | Read-only diagnosis of whether input actually works             |
| `ibus-candidate-tidy` | Parks the empty candidate box IBus leaves on screen             |
| `mozc-keymap`         | Reads/writes Mozc's custom keymap without the GUI               |
| `mozc/keymap.tsv`     | The Mozc keymap the kana switching depends on                   |
| `Xmodmap`             | Installed as `~/.Xmodmap`                                       |
| `install`             | Installs all of the above for the current user                  |
| `launchers/`          | XFCE autostart entries                                          |

## Requirements

Linux, X11 (not Wayland), XFCE for the shortcuts, IBus, Mozc with the IBus engine (`mozc-jp`), and a layout `xmodmap`
can modify.

```text
ibus  ibus-daemon  ibus-x11  xdotool  xmodmap  xset  pgrep  pkill
flock  timeout  setsid  nohup  Python 3  libX11.so.6  libc.so.6
libXtst.so.6 (optional; falls back to xdotool)
```

```sh
ibus list-engine | grep mozc     # must list mozc-jp
```

## Install

```sh
./install --all          # scripts, ~/.Xmodmap, autostart, Mozc keymap, XFCE shortcuts
./install --dry-run --all
```

Writes to `~/.local/bin`, `~/.Xmodmap`, `~/.config/autostart/`, `~/.config/mozc/config1.db` (with `--keymap`), and the
XFCE shortcuts (with `--shortcuts`). Everything overwritten is backed up as `<file>.bak-<timestamp>`.

```sh
export GTK_IM_MODULE=ibus QT_IM_MODULE=ibus XMODIFIERS=@im=ibus
ibus-daemon -drxR >"$HOME/.local/state/ibus-daemon.log" 2>&1 &
```

## Keyboard mapping

```text
keycode 105 = Katakana          Right Ctrl -> XFCE grab -> kana-toggle
keycode 118 = Hiragana          Insert     -> XFCE grab -> kana-width
keycode 103 = Eisu_toggle       half-width katakana carrier
keycode 135 = Zenkaku_Hankaku   Menu       -> English/Japanese, inside Mozc
clear control / add control = Control_L
```

```sh
xmodmap ~/.Xmodmap && kana-switch norepeat
```

`ibus-mozc-init`, the `Japanese keys` autostart entry and `install` all do this automatically. Only needed by hand
after running `xmodmap`.

## XFCE shortcuts

Settings -> Keyboard -> Application Shortcuts:

| Command       | Key (shown as the keysym) |
|---------------|---------------------------|
| `kana-toggle` | Right Ctrl (`Katakana`)   |
| `kana-width`  | Insert (`Hiragana`)       |

Apply `~/.Xmodmap` before recording them. `./install --shortcuts` writes both with `xfconf-query`.

The scripts inject `Muhenkan`, `Hiragana_Katakana` and `Eisu_toggle` rather than `Katakana` / `Hiragana`, because XFCE
has grabbed the latter and injecting one would re-invoke the shortcut.

## Autostart

```sh
cp launchers/*.desktop ~/.config/autostart/
```

`ibus-mozc-init --watch` runs for the whole session: fast polling and `~/.Xmodmap` re-application during login, then
15 s polling, plus a full `ibus-health` probe every 5 minutes. `ibus-daemon` supervises almost nothing - `-R` covers
only the panel and config modules - so the engine and `ibus-x11` die independently and never come back, often hours
after login.

## Testing

```text
Right Ctrl -> hiragana <-> katakana
Insert     -> full-width <-> half-width katakana
Menu       -> English <-> Japanese
```

| Do this                                         | Expect                |
|-------------------------------------------------|-----------------------|
| type `ka`, Right Ctrl, type `ki`                | かキ                    |
| from katakana: type `ka`, Right Ctrl, type `ki` | カき                    |
| katakana: Insert, type `ka`                     | ｶ                     |
| hiragana: Insert, type `ka`                     | ｶ                     |
| ｶﾀｶﾅ, Right Ctrl, Right Ctrl, type `ka`         | ｶ (width remembrance) |
| Japanese off: Right Ctrl, type `ka`             | カ                     |

Every one is a single press. Holding a key switches once, on release.

## Application support

Every application reaches IBus by one of four routes. The carrier is injected with XTEST at the server level, so it
arrives as an ordinary key press whichever route the focused window uses. Verified on real widgets, single and multi-line:

| Route                          | Applications                                          | Status            |
|--------------------------------|-------------------------------------------------------|-------------------|
| GTK3 `im-ibus.so`              | GTK3, Electron/Chromium, Firefox, VTE terminals       | works             |
| Qt `libibusplatform…plugin.so` | Qt5, Qt6                                              | works             |
| XIM via `ibus-x11`             | SDL games, Steam overlay, Java/Swing, legacy Qt/Motif | works             |
| GTK4 `libim-ibus.so`           | GTK4                                                  | broken, see below |

`ibus-health` reports all four.

## Commands

```sh
ibus-mozc-init [--watch]          # run once / supervise the session
ibus-health [--quiet]             # full report / exit code only

kana-switch toggle|width|hiragana|katakana|half-katakana
kana-switch status                # tracked mode, width, and how much to trust it
kana-switch keys                  # keycodes, modifiers, auto-repeat state
kana-switch norepeat              # disable auto-repeat on the shortcut keys
kana-switch --verbose toggle      # narrate one press

mozc-keymap show|diff FILE|apply FILE [--restart]
```

## Files

```text
~/.local/state/ibus-mozc-init.log        startup, repair and health events
~/.local/state/ibus-x11.log              XIM server output
~/.local/state/ibus-daemon.log           fallback daemon start
${XDG_RUNTIME_DIR:-/tmp}/ibus-mozc-init.lock
${XDG_RUNTIME_DIR:-/tmp}/mozc-kana-state[.lock]
~/.config/mozc/config1.db.bak-*          mozc-keymap backups
```

`mozc-kana-state` holds the mode and a fingerprint of the Mozc session it was recorded against. Read it with
`kana-switch status`; the fingerprint only means anything next to the processes running now.

## Troubleshooting

Engine wrong. `ibus engine` should say `mozc-jp`; run `ibus-mozc-init`. If it keeps being cleared, leave
`ibus-mozc-init --watch` running.

`ibus-x11` missing. Run `ibus-mozc-init`. If it dies again see `~/.local/state/ibus-x11.log` - that file exists
because `ibus-daemon` starts the XIM server with both descriptors on `/dev/null`. Set `IBUS_X11_BIN` if it lives
somewhere unusual.

`XOpenIM()` fails while `ibus-daemon` runs. `ibus-health`; look at `XIM selection` and `XOpenIM()`. A stale
`@server=ibus` advertisement makes `ps` look fine while every XIM client fails. The watcher's deep probe covers it.

Kana key works only sometimes. `kana-switch keys`. If keycode 105 is missing from the `Katakana` line, or listed
under modifiers, `~/.Xmodmap` was dropped - `xmodmap ~/.Xmodmap && kana-switch norepeat`. If it says `AUTO-REPEAT ON`,
run `kana-switch norepeat`. Then check `mozc-keymap diff mozc/keymap.tsv`. On a slow machine raise the settle time:
`KANA_SETTLE_MS=120`.

Toggles backwards. `kana-switch status`. `confidence: assumed` means the stored mode was discarded, which is
correct after Mozc restarts, press again, every press is absolute. Persistently backwards means the Mozc keymap is
not installed.

Insert does nothing. Check `kana-switch keys` lists a keycode for `Eisu_toggle`; if not, `~/.Xmodmap` is not
applied. Then `mozc-keymap diff mozc/keymap.tsv`.

Empty candidate box. `pgrep -af 'ibus-candidate[-]tidy'`, then `ibus-candidate-tidy --verbose`. Threshold is 60 px;
override with `IBUS_TIDY_HEIGHT_MAX`. Disable with `touch ~/.config/ibus-candidate-tidy.disabled`.

## Removing

```sh
rm -f ~/.local/bin/{kana-switch,kana-toggle,kana-width,ibus-mozc-init,ibus-health,ibus-candidate-tidy,mozc-keymap}
rm -f ~/.config/autostart/ibus-mozc.desktop "$HOME/.config/autostart/Japanese keys.desktop"
rm -f ~/.Xmodmap
pkill -x ibus-mozc-init
pkill -f 'ibus-candidate[-]tidy'
```

Restore Mozc's previous keymap from the newest `~/.config/mozc/config1.db.bak-*`, then `pkill -x mozc_server`. Remove
the XFCE shortcuts by hand.

## License

MIT. See [LICENSE](LICENSE).

Created by MattFor - 2026
