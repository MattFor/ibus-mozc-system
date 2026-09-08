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
| --------------------- | --------------------------------------------------------------- |
| `kana-switch`         | The kana switch;                                                |
| `kana-toggle`         | `kana-switch toggle` - Right Ctrl                               |
| `kana-width`          | `kana-switch width` - Insert                                    |
| `ibus-mozc-init`      | Brings the IBus/Mozc stack up and supervises it for the session |
| `ibus-health`         | Read-only diagnosis of whether the stack works                  |
| `ime-probe`           | Read-only diagnosis of one application, or of every open window |
| `ime-launch`          | Runs a program on the IM route it actually works with           |
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
gdbus                       (session bus check)
dbus-monitor  ss            (ime-probe)
libXtst.so.6 (optional; falls back to xdotool)
```

```sh
ibus list-engine | grep mozc     # must list mozc-jp
```

## Install

```sh
./install --all          # scripts, ~/.Xmodmap, autostart, Mozc keymap, XFCE shortcuts, launcher routes
./install --dry-run --all
```

Writes to `~/.local/bin`, `~/.Xmodmap`, `~/.config/autostart/`, `~/.config/mozc/config1.db` (with `--keymap`), and the
XFCE shortcuts (with `--shortcuts`). Everything overwritten is backed up as `<file>.bak-<timestamp>`.

```sh
export GTK_IM_MODULE=ibus QT_IM_MODULE=ibus XMODIFIERS=@im=ibus
export SDL_IM_MODULE=ibus CLUTTER_IM_MODULE=ibus
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
| ------------- | ------------------------- |
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
| ----------------------------------------------- | --------------------- |
| type `ka`, Right Ctrl, type `ki`                | かキ                  |
| from katakana: type `ka`, Right Ctrl, type `ki` | カき                  |
| katakana: Insert, type `ka`                     | ｶ                     |
| hiragana: Insert, type `ka`                     | ｶ                     |
| ｶﾀｶﾅ, Right Ctrl, Right Ctrl, type `ka`         | ｶ (width remembrance) |
| Japanese off: Right Ctrl, type `ka`             | カ                    |

Every one is a single press. Holding a key switches once, on release.

## Application support

Every application reaches IBus by one of these routes. The carrier is injected with XTEST at the server level. It
arrives as an ordinary key press whichever route the focused window uses.

| Route                          | Applications                                          | Status           |
| ------------------------------ | ----------------------------------------------------- | ---------------- |
| GTK3 `im-ibus.so`              | GTK3, Firefox, VTE terminals                          | works            |
| Qt `libibusplatform…plugin.so` | Qt5, Qt6                                              | works            |
| XIM via `ibus-x11`             | SDL games, Steam overlay, Java/Swing, Wine, legacy Qt | works            |
| XIM via `ime-launch`           | Electron/Chromium (Discord, VS Code, …)               | works            |
| XIM via a wrapped entry        | Steam's client UI (CEF in its own container)          | works            |
| Portal + env override          | flatpak applications                                  | works            |
| GTK4 `libim-ibus.so`           | GTK4 (zenity, GNOME apps)                             | works, see below |

`ibus-health` reports the stack. `ime-probe` reports one application against it.

```text
       win      pid  class                  route
[ ok ]   1    32166  discord                im-xim.so (Chromium on the XIM route)
[ ok ]   1    26870  steam                  im-xim.so (Chromium on the XIM route)
[ ok ]   1    32424  waterfox               native IBus module + own bus connection
[ ?  ]   2     1530  Xfce4-panel            GTK_IM_MODULE=ibus; no module loaded yet
```

### Electron does not talk to IBus

With `GTK_IM_MODULE=ibus`, an Electron application **does** load
`im-ibus.so`, **does** open an IBus input context, and **does** report focus and cursor position into it. It never
calls `ProcessKeyEvent`. Measured on Discord (Electron 42.9.0) with the quick switcher open and its text field focused:

```text
GTK_IM_MODULE=ibus    0 ProcessKeyEvent calls   nothing types
GTK_IM_MODULE=xim    40 ProcessKeyEvent calls   works
```

Every check passes - `ibus-health` green, engine `mozc-jp`, context alive and taking focus and Japanese still
never happens. There is nothing to repair at the IBus end; the application is dropping the keys before the IM layer
is able to register them. Switching to XIM, however, where `ibus-x11` handles the keys instead:

```sh
ime-launch discord          # picks xim for Electron trees, ibus for everything else
ime-launch --detect PROG    # print the route it would pick, change nothing
```

`ime-launch` sets this per process, so the rest of the session keeps the native IBus module and its on-the-spot
preedit.

P.S Do not set `GTK_IM_MODULE=xim` globally to "fix" this as it downgrades every actual GTK3 program.

### Steam's UI has no module to load

Electron loads `im-ibus.so` and ignores it; Steam's client UI never gets
one. Since the sniper container became mandatory, `steamwebhelper` (CEF) runs inside pressure-vessel, and the GTK3 in
there ships no ibus immodule at all:

```text
$ ls /proc/$(pgrep -x steamwebhelper | head -1)/root/usr/lib/*/gtk-3.0/*/immodules/
im-am-et.so  im-broadway.so  im-cedilla.so  im-cyrillic-translit.so  im-inuktitut.so
im-ipa.so    im-multipress.so  im-thai.so   im-ti-er.so  im-ti-et.so  im-viqr.so
im-wayland.so  im-waylandgtk.so  im-xim.so
```

The session's `GTK_IM_MODULE=ibus` **is** however inherited into the container, it reads correctly in
`/proc/<steamwebhelper>/environ`. It just names nothing there and GTK falls back to `GtkIMContextSimple` without a
word. The immodules cache in the container lists `"xim"` and `"wayland"`, no `"ibus"`, and the process holds no IBus
connection by any route: its only sockets are the two D-Bus buses, at-spi, the X11 socket and Steam's own shared
memory.

`im-xim.so` **is** in there, and XIM reaches `ibus-x11` over the X11 socket the container already has.

```text
$ SteamLinuxRuntime_sniper/run-in-sniper -- ldd .../immodules/im-xim.so
libgtk-3.so.0 => /lib/x86_64-linux-gnu/libgtk-3.so.0                     the container's GTK
libX11.so.6   => /usr/lib/pressure-vessel/overrides/lib/.../libX11.so.6  the session's Xlib
                                                                         no unresolved deps
$ ... XMODIFIERS=@im=ibus python3 -c '<XOpenIM>'
locale     : en_US.UTF-8
XMODIFIERS : @im=ibus
XOpenIM()  : 0x3115fa90       ibus-x11, opened from inside the container
```

So it is the same route as Electron, set the same way but it has to be set on the process that starts the
container, which is the Steam client itself, which is started from a desktop entry:

```sh
ime-launch --wrap-desktop steam     # ~/.local/share/applications/steam.desktop
ime-launch --unwrap-desktop steam   # put it back
```

Games launched from Steam inherit `GTK_IM_MODULE=xim` from the client. They reach IBus over
XIM already, through SDL or Wine, and neither reads that variable.

### Custom launcher parameters

| Parameter                           | Does                                                                   |
| ----------------------------------- | ---------------------------------------------------------------------- |
| `ime-launch PROG [ARGS...]`         | runs PROG on the route it needs, and nothing else on it                |
| `ime-launch --route xim\|ibus PROG` | forces one, for a program the detection gets wrong                     |
| `ime-launch --detect PROG`          | prints `route<TAB>resolved-path`, starts nothing                       |
| `ime-launch --wrap-desktop NAME`    | routes `NAME.desktop` - a copy in `~/.local/share/applications`        |
| `ime-launch --unwrap-desktop NAME`  | removes that copy (or the prefix, for an entry the app owns)           |
| `ime-launch --wrap-all`             | every installed entry that needs it, plus the Chromium-shaped flatpaks |
| `--dry-run`                         | with either wrap mode: list what would change, change nothing          |
| `./install --wrappers`              | `--wrap-all` as an install step; `--all` includes it                   |

### GTK4 hangs on one D-Bus name

GTK4 input is decided entirely by whether `org.freedesktop.IBus` has an owner on the session bus.

ibus's GTK4 module gates `filter_keypress` behind `_daemon_is_running`, and the only thing that ever sets it is a
`g_bus_watch_name()` on that name on the session bus:

```c
if (!_daemon_is_running)
    return gtk_im_context_filter_keypress (ibusimcontext->slave, event);
```

### flatpak

A flatpak gets neither the session's IM environment nor a route to IBus. One global override covers every app:

```sh
flatpak override --user \
    --talk-name=org.freedesktop.portal.IBus \
    --env=GTK_IM_MODULE=ibus --env=QT_IM_MODULE=ibus \
    --env=XMODIFIERS=@im=ibus --env=SDL_IM_MODULE=ibus
```

`ibus-portal` is what carries IBus into the sandbox; `XMODIFIERS` additionally lets anything that speaks XIM reach
`ibus-x11` over the X11 socket the app already has. Electron shipped as a flatpak needs
`--env=GTK_IM_MODULE=xim` on that app specifically, for the reason above `ime-launch --wrap-all` finds those by
their markers and writes it:

```sh
flatpak override --user --env=GTK_IM_MODULE=xim <app>   # what it writes
flatpak override --user --reset <app>                   # what undoes it
```

### Fullscreen and games

| Case                                      | Typing Japanese | Right Ctrl (XFCE global grab) |
| ----------------------------------------- | --------------- | ----------------------------- |
| fullscreen, no keyboard grab              | works           | fires                         |
| fullscreen + exclusive seat keyboard grab | works           | fires                         |

### The leftover candidate box

When a conversion ends, `ibus-ui-gtk3` shrinks its candidate window to just the two page arrows and leaves it mapped,
on screen, until the hide arrives up to a second and a half later. Measured transitions of that window while
typing one word:

```text
3.55s  mapped  on-screen  111x144     real candidate list
3.58s  mapped  on-screen  150x144     still real
4.47s  mapped  on-screen   98x44      <- leftover, empty but for the arrows
5.88s  unmapped                       IBus finally hides it
```

`ibus-candidate-tidy` moves that leftover off screen.

```sh
ibus-candidate-tidy --verbose        # narrate every park and restore
IBUS_TIDY_HEIGHT_MAX=60              # taller than this is a real list
IBUS_TIDY_SWEEP=2.0                  # safety sweep; IBUS_TIDY_POLL still read
touch ~/.config/ibus-candidate-tidy.disabled   # turn it off
```

## Commands

```sh
ibus-mozc-init [--watch]          # run once / supervise the session
ibus-health [--quiet]             # full report / exit code only

ime-probe                         # does the focused window reach IBus?
ime-probe --window ID             # ...that one instead
ime-probe --watch [SECS]          # watch while you type, rather than injecting
ime-probe --list                  # every process holding an IBus connection
ime-probe --audit                 # every window on the desktop, and its route

ime-launch PROGRAM [ARGS...]      # run it on the route it works with
ime-launch --route xim PROGRAM    # force XIM
ime-launch --detect PROGRAM       # print the route, change nothing
ime-launch --wrap-desktop NAME    # route NAME.desktop through ime-launch
ime-launch --unwrap-desktop NAME  # undo that
ime-launch --wrap-all [--dry-run] # every entry that needs it, flatpaks included

kana-switch toggle|width|hiragana|katakana|half-katakana
kana-switch status                # tracked mode, width, and how much to trust it
kana-switch keys                  # keycodes, modifiers, auto-repeat state
kana-switch norepeat              # disable auto-repeat on the shortcut keys
kana-switch --verbose toggle      # narrate one press

mozc-keymap show|diff FILE|apply FILE [--restart]
```

## Files

```text
~/.xprofile                              session IM environment (no daemon start)
~/.local/share/flatpak/overrides/global  IM environment for flatpak applications
~/.local/share/applications/*.desktop    entries wrapped with --wrap-desktop
~/.local/share/flatpak/overrides/<app>   per-app route for a Chromium-shaped flatpak
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

**Japanese does not work in one particular application**, and everything else is fine.

```sh
ime-probe            # with the caret already in a text field in that window
ime-probe --watch    # ...then type, if injecting is not convincing
```

It counts whether the keys reach IBus at all. `0 ProcessKeyEvent calls` from a window whose text field really was
focused means the application is dropping them before the IM layer. `ime-launch` it. Anything above zero means the
routing is fine and the question is a Mozc mode or keymap one, so go to `kana-switch status` and `mozc-keymap diff`.

Note that no toolkit offers keys to an input method while the caret is outside a text box, so probing a window's
sidebar or channel list reports zero on a perfectly healthy program. This is a false-positive!

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

Empty candidate box, or one that keeps re-appearing. `pgrep -af 'ibus-candidate[-]tidy'`, then
`ibus-candidate-tidy --verbose` and watch it. Threshold is 60 px; override with `IBUS_TIDY_HEIGHT_MAX`. If a
real candidate list gets parked, that threshold is too high for your font size. Disable entirely with
`touch ~/.config/ibus-candidate-tidy.disabled`.

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

MIT [LICENSE](LICENSE).

By MattFor
