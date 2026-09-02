---
name: vnc-chrome-portal
description: "marslo's TigerVNC/XFCE server — Chrome upload dialog fix, portal disabled at runtime, and session gotchas"
metadata: 
  node_type: memory
  type: project
  originSessionId: 5f94dbe6-4044-4055-9efe-312e2f7e6ba8
  modified: 2026-08-29T08:04:01.702Z
---

Host `sj4dl360n4u20`, Ubuntu 22.04, TigerVNC + XFCE, display `:1`, linger enabled for marslo. Working dir `~/.vnc`.

**Migrated 2026-08-29:** marslo's `:1` now runs via the official `tigervncserver@:1.service` (PAM login session → `XDG_RUNTIME_DIR` provided by logind). Options live in `~/.vnc/tigervnc.conf` (Perl: `$geometry="1920x1080"; $depth="24"; $localhost="no"; 1;`); DPI via `xrandr --dpi 120` in `~/.vnc/xstartup` (official unit has no `-dpi`). `/etc/tigervnc/vncserver.users` maps `:1=marslo`. The old custom `/etc/systemd/system/vncserver@.service` is **disabled but kept** (file preserved, not deleted). Rollback: `sudo systemctl disable --now tigervncserver@:1 && sudo systemctl enable --now vncserver@1`.

**Chrome file-upload dialog opened at `/`** because Chrome (>=123) delegates the file dialog to `xdg-desktop-portal` (+ `xdg-desktop-portal-gtk`) whenever the portal is reachable, and that GTK backend ignores Chrome's own `last_directory`. Chrome's `~/.config/google-chrome/Default/Preferences` `selectfile.last_directory` was always correct — Chrome just doesn't consult it in portal mode.

**Fix (applied 2026-08-28), packages kept installed:** disable the portal at runtime by shadowing its D-Bus activation. File `~/.local/share/dbus-1/services/org.freedesktop.portal.Desktop.service` with `Exec=/bin/false` and NO `SystemdService=` line. `~/.local/share/dbus-1/services` outranks `/usr/share/...`, so portal activation runs `/bin/false`, fails, and Chrome falls back to its built-in GTK dialog (which reads `last_directory`). Verified: `gdbus ... Peer.Ping` on the session bus returns `Spawn.ChildExited ... status 1`. Revert = delete that file + restart the VNC session.

**Key gotcha — why `systemctl --user mask xdg-desktop-portal` does NOT work here:** the VNC session runs on a `dbus-launch` bus (`~/.vnc/xstartup` does `unset DBUS_SESSION_BUS_ADDRESS; exec dbus-launch --exit-with-session startxfce4`), and `session.conf` has no systemd activation, so the portal is activated *directly* by dbus-daemon, bypassing systemd. Masking the systemd user unit is bypassed; shadowing the D-Bus service file is the reliable, bus-agnostic lever.

**Same-day cleanups (only needed to make the now-disabled portal work):** `xstartup` `XDG_CURRENT_DESKTOP=XFCE:GNOME` -> `XFCE` (the `:GNOME` existed only to match `gtk.portal`'s `UseIn=gnome` so the FileChooser backend got selected); removed `~/.config/autostart/fix-portal-env.desktop` (it only pushed DISPLAY/XAUTHORITY/XDG_CURRENT_DESKTOP into the systemd --user env for the portal — session-bus services already inherit DISPLAY from xstartup). gnome-keyring secrets unaffected: `org.freedesktop.secrets` is on-demand D-Bus activated.

**Consolidated runbook (2026-08-29):** published as an Artifact — https://claude.ai/code/artifact/ca35953c-c638-4875-bc28-89f19d32b83f (source in scratchpad `vnc-runbook.html` / `vnc-runbook.md`). Covers multi-user VNC via official `tigervncserver@` + `/etc/tigervnc/vncserver.users` (`:2=ubuntu`, `:3=iliad`), per-user `~/.vnc/tigervnc.conf` (Perl syntax; `$localhost` defaults to "yes"/localhost-only under VncAuth), the portal fix, SSH-tunnel hardening, and VNC password recovery (`~/.vnc/passwd` is reversible fixed-key DES → decrypt with OpenSSL key `E84AD660C4721AE0`). To update it, republish with that URL.

**Unrelated unit bug (fixed same day):** `ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1` — systemd runs no shell, so `> /dev/null 2>&1` became literal argv, so the pre-kill always failed and `systemctl restart` aborted with "A Xtigervnc server is already running". Fixed by dropping the redirect: `ExecStartPre=-/usr/bin/vncserver -kill :%i`. If a restart ever still fails, run `vncserver -kill :1` (and if needed rm `/tmp/.X1-lock` `/tmp/.X11-unix/X1`) first.
