## Setup:
NOTE: EL10+ ONLY

Resources:
https://forge.puppet.com/modules/jcpunk/weston  # module by Pat Riehecky (v 0.6.2+)
https://github.com/fermitools/fnal-vncpasswd  # packaged in Fermi context for EL10+

You will need the following classes attached:
  - weston
  - weston::vnc_server

Optionally:
  - stdlib::manage

In hiera put the following:

```yaml
weston::vnc_server::vnc_server_options:
  - "--address=::1"
  - "--disable-transport-layer-security"
weston::manage_weston_ini: false # /etc/xdg/weston/weston.ini comes from fermilab-conf_screenlock

stdlib::manage::create_resources:
  package:
    fermilab-conf_screenlock:
      ensure: installed
```

If there is **no physical monitor** on this host (all SSI systems?) you can further simplify hiera to:

```yaml
weston::vnc_server::vnc_server_options:
  - "--address=::1"
  - "--disable-transport-layer-security"
weston_ini_settings:
  core:
    idle-time: 900
  shell:
    locking: false
    clock-format: "minutes-24h"
  vnc:
    mode: "1280x720"
    resizeable: true
stdlib::manage::create_resources:
  package:
    fermilab-conf_screenlock:
      ensure: absent
```

Out of the box, weston will use your PAM stack for authentication.
If you're using authselect you should have stable `login` substacks.

To permit use of the fnal-vncpasswd binary you should update:
`/etc/pam.d/weston-remote-access`

This config probably works but you should evaluate your needs:

```conf
#%PAM-1.0
auth    [success=done default=ignore] pam_fnal_vncpasswd.so
auth    include login
account include login
```

## User interaction

Weston is the only modern desktop that has retained a VNC backend in RHEL.
If you need to run individual GUI applications, `waypipe` might be better.

Weston can be unhappy with multiple simultaneous VNC users of the same desktop.

### Configuration

Each user should customize `~/.config/weston.ini` to fit their needs.
This is a good starting point:

```ini
[shell]
locking=false

[vnc]
mode=1280x720
resizeable=true
```

Locking must be enabled if there are non-VNC users of Weston.

If fnal-vncpasswd is configured on your host for authentication, you can set a VNC password
with the `fnal-vncpasswd` command. You should review its manpages for more information.

The password set with `fnal-vncpasswd` cannot be used on the Weston lockscreen(`locking=true`).

### System VNC sessions
These VNC sessions are configured by the system admin via puppet.

You should fill out the following to request a System VNC session:

```yaml
weston::vnc_server::vnc_sessions:
  username:
    displaynumber: *explicit number* or "next available"
    ensure: XXXXX
    enable: XXXX
```

To have Puppet actively manage the VNC service state and try guarantee it is running, request:

```yaml
ensure: running
enable: true
```

If you prefer to manage the service lifecycle manually (start/stop outside of Puppet), request:

```yaml
ensure: NULL
enable: false
```

Then you will be able to run:

```shell
systectl restart weston-vncserver@`whoami`.service
```

start and stop should work just fine too, but mind what you asked for in puppet.

### User VNC sessions
These can be started by any user without system admin configuration.

End users will start VNC sessions with:

```shell
systemctl --user start weston-user-vnc@$PORT.service
```
stop and restart work just fine too.

Some VNC clients require you to toggle on/off shared cursor before anything (including rendering) will occur.

If the user is permitted to run services at system start (ie. linger) then you can enable it.

If your session is getting killed after logout you can run it instead like:

```shell
systemd-run --user --scope --remain-after-exit \
  --unit=weston-user-vnc@$PORT.scope \
  /usr/bin/systemctl --user start weston-user-vnc@$PORT.service
```

For simplicy you might want to build a wrapper around the more advanced execution and just use that.
