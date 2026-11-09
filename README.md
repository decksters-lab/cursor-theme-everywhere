# cursor-theme-everywhere

One cursor theme, everywhere: every desktop, every compositor, and the login screen.

Linux has no single "cursor theme" setting. The pointer you see comes from a pile of
independent layers that don't read each other: environment variables, the XDG `default`
theme, GTK settings, dconf, KDE's `kcminputrc`, per-compositor configs, and the greeter's
own config. If any one of them names a different theme, that's the one you get in some
window, session or login screen. `cursor-theme` keeps all of them in agreement from a
single saved choice, and `cursor-theme status` tells you which layer disagrees.

It is one bash script with no dependencies beyond coreutils, awk and sed. The project is
`cursor-theme-everywhere`; the command it installs is `cursor-theme`.

## Install

```sh
git clone https://github.com/decksters-lab/cursor-theme-everywhere.git
cd cursor-theme-everywhere
install -Dm755 cursor-theme ~/.local/bin/cursor-theme   # or symlink it, see below
```

Make sure `~/.local/bin` is on your `PATH`. To keep it updatable from the clone, symlink
instead: `ln -s "$PWD/cursor-theme" ~/.local/bin/cursor-theme`.

## Use

```sh
cursor-theme list                       # installed cursor themes
cursor-theme status Bibata-Modern-Ice   # audit everything, change nothing, no root
cursor-theme set Bibata-Modern-Ice 24   # apply everywhere (asks for sudo/doas once)
cursor-theme sync                       # re-apply the saved theme, user files only, no root
cursor-theme                            # interactive picker
```

Run `status` first on an unfamiliar machine. Every line is a place that names a cursor
theme; red lines disagree with the theme you asked about. After `set`, log out and back
in. Then `status` should be all green (apart from files you chose to leave alone).

The saved choice lives in `~/.config/cursor-theme/state`. `sync` is cheap and idempotent,
so it is safe to call from a compositor's autostart if some component keeps resetting
your settings.

## What it changes

**System layer** (one privilege prompt, `sudo` then `doas`; override with
`CT_ELEVATE=doas|sudo|pkexec`):

| What | Where |
| --- | --- |
| Theme copied to the one directory every client and greeter searches by default (parents named in `Inherits=` are copied too) | `/usr/share/icons/<theme>` |
| System-wide fallback theme | `/usr/share/icons/default/index.theme` |
| Session environment, read by PAM at login | `/etc/environment`: `XCURSOR_THEME`, `XCURSOR_SIZE`, `HYPRCURSOR_THEME`, `HYPRCURSOR_SIZE` |
| Greeter config, if present | noctalia-greeter `greeter.toml` `[cursor]`; regreet `regreet.toml` `[GTK]` |

**User layer** (never needs root):

| What | Where |
| --- | --- |
| XDG fallback theme | `~/.local/share/icons/default/index.theme` (and `~/.icons/default` if it exists) |
| GTK 2/3/4 | `settings.ini`, `~/.gtkrc-2.0` (if present) |
| GNOME and Cinnamon | gsettings/dconf |
| KDE Plasma | `kcminputrc` via `kwriteconfig6`/`5`, live-applied if Plasma is running |
| Xfce | xfconf `xsettings` channel (`/Gtk/CursorThemeName`, `/Gtk/CursorThemeSize`): live through `xfconf-query` while `xfsettingsd` runs, otherwise `xsettings.xml` is edited directly |
| X11 / xsettingsd | `~/.Xresources`, `xsettingsd.conf` (only if present) |
| Wayfire | `~/.config/wayfire.ini` `[input]` |
| labwc | `~/.config/labwc/environment`, and `~/.config/xfce4/labwc/environment` (the private copy Xfce's Wayland session starts labwc with) |
| Hyprland (`.conf`), MangoWM, sway, niri | rewritten **only** where the config already pins a cursor |

Compositor configs are deliberately not touched unless they already name a cursor. The
design is that a compositor with no pin follows `/etc/environment` and the `default`
theme, which is how COSMIC and a Hyprland with no pin behave, and the tool does not
invent keys in files it doesn't understand. If you pin the cursor in a format it can't
rewrite (for example a Hyprland Lua config), `set` prints the lines to edit by hand and
`status` flags them when they disagree. Note: I removed the cursor choice i previously had configured for wayfire,mango and hyprrland from wayfire.ini, mangowms .conf, and hyprland.lua on my own system.This allows /etc/environment to hold the variable for all 3 but this is not something you HAVE to do... it just eliminates the need to edit either of the 3 by hand. 

When run inside Hyprland or sway it also applies the theme live (`hyprctl setcursor`,
`swaymsg seat`).

## Safety

- The first time a file is changed, the original is saved as `FILE.pre-cursor-theme`.
  It is never overwritten and backups don't accumulate.
- Writes are in place, so file owner and mode are preserved. A greeter config keeps
  belonging to the greeter user.
- Nothing is changed if the content is already right, so re-running is a no-op.
- It never restarts greetd. A new greeter process starts at your next logout, and
  restarting greetd from inside a session ends that session.
- Theme names are validated against `[A-Za-z0-9._+-]`.
- Theme copies in `/usr/share/icons` carry a `.cursor-theme-managed` marker. A directory
  without the marker (for example, one installed by a package) is never overwritten.

### Revert

Restore any `*.pre-cursor-theme` file over its original, delete `/usr/share/icons/<theme>`
if it contains `.cursor-theme-managed`, and remove the `XCURSOR_*` / `HYPRCURSOR_*` lines
from `/etc/environment`.

## Why these choices

- **`/usr/share/icons`, not `/usr/local/share/icons`.** The default Xcursor search path
  is `~/.icons:/usr/share/icons:/usr/share/pixmaps`. `/usr/local` is not on it, so a
  theme there is only found if `XCURSOR_PATH` reaches every client, and it doesn't reach
  a greeter. Putting the theme in `/usr/share/icons` needs no path at all.
- **Hyprland needs `HYPRCURSOR_*` too.** Hyprland draws cursors in the hyprcursor format
  when a theme ships one (Bibata does). With only `XCURSOR_*` set it can still end up
  showing a different theme. Setting `HYPRCURSOR_THEME`/`HYPRCURSOR_SIZE` fixed that on
  my machine, where this was developed on.
- **The greeter can't read your home.** A theme that exists only in
  `~/.local/share/icons` is invisible to the login screen, which runs as another user.

## Troubleshooting

**`status` shows a mismatch I just fixed.** The "this shell has `XCURSOR_THEME=...`"
warning is normal until you log in again. Environment changes apply at login.

**Wrong pointer in Hyprland only, even though its environment is right.** Make sure
`HYPRCURSOR_THEME` is set (this tool does it) and that no `env`/`hl.env` line in your
Hyprland config sets another theme. A quick check of what Hyprland received:
`tr '\0' '\n' < /proc/$(pgrep -x Hyprland)/environ | grep -i cursor`.

**Arrow instead of a text I-beam over text fields.** Some themes ship without a `text`
cursor. If the theme has `xterm`, `ln -s xterm <theme>/cursors/text` and re-run `set`.
The tool warns about this when it sees it.

**Xfce keeps the old pointer.** Xfce's settings daemon (`xfsettingsd`) publishes its own
cursor setting to every GTK/X client and overrides `settings.ini`, so setting only the GTK
files has no visible effect there. `cursor-theme` updates the xfconf values; if you ran
`set` from another session, run `cursor-theme sync` inside Xfce or log out and back in.
`status` shows an `xfce xsettings` line when an Xfce config exists.

**Xfce's Wayland session ignores the theme.** `startxfce4 --wayland` runs labwc with its own
config directory, `~/.config/xfce4/labwc`, and never reads `~/.config/labwc`. That directory's
`environment` file ships with `XCURSOR_THEME=Adwaita`. `cursor-theme` updates it (theme and size)
when it exists; log out and back in to the Xfce Wayland session afterwards.

**Greeter shows `(dir not readable as you)` in `status`.** That is expected; the greeter's
state directory is usually readable only by the greeter user. Check with
`sudo grep -A3 '^\[cursor\]' /var/lib/noctalia-greeter/greeter.toml`.

**`/usr/share/icons/default/index.theme` keeps reverting.** On some distributions a
package owns that file and an upgrade can replace it. Re-run `cursor-theme set <theme>`.
The tool warns when it detects package ownership (pacman).

**Two `[cursor]` tables in a greeter config.** That's a TOML error and the greeter ignores
the file. `set` warns if it sees it; fix by hand.

## Status and limits

Developed and used on Artix Linux (dinit) with greetd and noctalia-greeter, and verified
there in Hyprland. Other layers (Plasma's `kcminputrc`, Xfce's xfconf settings, regreet's
key name, niri/sway/mango rewrites) have had less real-world use. If a layer misbehaves for
you, `status` output and an issue are very welcome.

Not handled: other greeters' private config formats (tuigreet has no cursor; gtkgreet
follows GTK settings of its own user), Hyprland Lua config rewriting, and hyprcursor-only
themes without a classic `cursors/` directory (the greeter and most toolkits need that).

## Environment variables

| Variable | Meaning |
| --- | --- |
| `CT_ELEVATE` | Force the privilege tool: `sudo`, `doas` or `pkexec`. |
| `CT_SYS` | Prefix for system paths. For testing against a fake root; leave unset normally. |

## License

MIT, see [LICENSE](LICENSE).
