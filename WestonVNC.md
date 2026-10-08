## Setup:
NOTE: EL10+ ONLY

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

**NOTE**: Locking must be enabled if there are non-VNC users of Weston.

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
