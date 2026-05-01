# Podman Quadlets Reference for SLAM systems

Requires: EL9.7+/EL10.1+

Quadlets provide systemd-native container management. This document references the
upstream podman-quadlets documentation for authoritative guidance. Refer to the
links below for complete examples, syntax, and configuration options.

**Primary References:**
- [Podman Quadlet Documentation](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
  Complete reference for all quadlet unit types and directives
- [Podman Container Unit](https://docs.podman.io/en/latest/markdown/podman-container.5.html)
  Container-specific configuration options
- [Podman Pod Unit](https://docs.podman.io/en/latest/markdown/podman-pod.5.html)
  Pod-specific configuration options

## Prerequisites for Rootless Containers

Rootless Podman requires subordinate user IDs (subuids) and group IDs (subgids).
See [subuid(5) / subgid(5)](https://man7.org/linux/man-pages/man5/subuid.5.html)
and [Podman Rootless Mode](https://docs.podman.io/en/latest/markdown/podman.1.html#rootless-mode-configuration)
for setup and verification procedures.

Key requirements:
- Each user needs entries in `/etc/subuid` and `/etc/subgid`
- Entries must exist before quadlet startup
- Systemd lingering must be enabled for persistent user containers
- Containers run with delegated UID range, not as root

## Container Units (.container)

Create container units following the quadlet syntax. See
[Podman Container Unit](https://docs.podman.io/en/latest/markdown/podman-container.5.html)
for complete directives and examples.

Place files in `~/.config/containers/systemd/` (user) or `/etc/containers/systemd/` (system).

Essential directives:
- `Image` - Fully qualified image name (e.g., `quay.io/namespace/image:tag`)
- `ContainerName` - Container name (optional)
- `PublishPort` - Port mapping syntax
- `Volume` - Mount paths
- `Environment` - Environment variables
- `RestartPolicy` - `always`, `on-failure`, `never`
- `Exec` - Override entrypoint
- `WorkingDir` - Working directory
- `User` - UID or username

Management:
```bash
systemctl --user daemon-reload           # Reload quadlets
systemctl --user start name.container    # Start
systemctl --user stop name.container     # Stop
systemctl --user enable name.container   # Auto-start
journalctl --user -u name.container -f   # View logs
podman ps                                # List containers
```

## Kubernetes YAML Units (.kube)

Podman quadlets support Kubernetes YAML manifests. See
[Podman kube play](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html)
for YAML syntax and supported resource types.

Essential directives:
- `File` - Path to Kubernetes YAML file
- `Inline` - Inline YAML content (alternative to File)

Supported Kubernetes resources: Pod, Deployment, DaemonSet, Job, PersistentVolumeClaim,
ConfigMap, Secret

Management:
```bash
systemctl --user daemon-reload      # Reload quadlets
systemctl --user start name.kube    # Deploy
systemctl --user stop name.kube     # Undeploy
systemctl --user enable name.kube   # Auto-start
journalctl --user -u name.kube -f   # View logs
podman pod ls                       # List pods
podman ps -a                        # List all containers
```

## Pod Units (.pod)

Pods group multiple containers sharing network namespace, storage, and lifecycle.
All containers in a pod share the same IP address and can communicate via localhost.

See [Podman Pod Unit](https://docs.podman.io/en/latest/markdown/podman-pod.5.html)
for complete pod configuration.

Essential directives:
- `PodName` - Pod name
- `PublishPort` - Expose ports at pod level
- `Volume` - Named volumes for pod
- `Network` - Network mode

Related `.container` directives:
- `Pod` - Attach container to pod
- `PartOf` - Declare pod dependency

Management:
```bash
systemctl --user daemon-reload     # Reload quadlets
systemctl --user start name.pod    # Start pod (all containers)
systemctl --user stop name.pod     # Stop pod (all containers)
systemctl --user enable name.pod   # Auto-start
journalctl --user -u name.pod -f   # View logs
podman pod ls                      # List pods
podman ps -a                       # List containers
```

Containers within same pod access each other at `localhost:port` without port publishing.

## Network Units (.network)

Create custom networks for container isolation and communication. See
[Podman Network](https://docs.podman.io/en/latest/markdown/podman-network.1.html)
for network configuration options.

Essential directives:
- `NetworkName` - Network name
- `Driver` - `bridge` (default), `macvlan`, `ipvlan`
- `Subnet` - CIDR notation
- `Gateway` - Gateway IP
- `Dns` - DNS servers
- `Internal` - Restrict external access

Reference in containers with `Network=network-name` and declare dependency:
```
Requires=name.network
After=name.network
```

Management:
```bash
systemctl --user daemon-reload      # Reload quadlets
systemctl --user start name.network # Create network
systemctl --user enable name.network # Auto-create
journalctl --user -u name.network -f # View logs
podman network ls                   # List networks
podman network inspect net-name     # Inspect network
```

Networks must exist before containers referencing them start.

## Volume Units (.volume)

Named volumes persist container data and can be shared between containers.
See [Podman Volume](https://docs.podman.io/en/latest/markdown/podman-volume.1.html)
for volume management.

Essential directives:
- `VolumeName` - Volume name
- `Label` - Metadata labels (optional)
- `Driver` - Volume driver

Mount syntax in containers:
- Named volume: `Volume=name:/path:rw`
- Bind mount: `Volume=/host/path:/container/path:ro`
- tmpfs: `Volume=tmpfs:/tmp:tmpfs`

Declare dependencies:
```
Requires=name.volume
After=name.volume
```

Management:
```bash
systemctl --user daemon-reload      # Reload quadlets
systemctl --user start name.volume  # Create volume
systemctl --user enable name.volume # Auto-create
journalctl --user -u name.volume -f # View logs
podman volume ls                    # List volumes
podman volume inspect vol-name      # Inspect volume
podman volume rm vol-name           # Delete volume
```

Volumes must exist before containers mount them.

## Port Publishing and Networking

See [Podman Container Run](https://docs.podman.io/en/latest/markdown/podman-run.1.html)
for port publishing syntax and networking options.

Port publishing syntax:
- TCP (default): `PublishPort=8080:80`
- UDP: `PublishPort=53:53/udp`
- Bind to interface: `PublishPort=127.0.0.1:8080:80`
- Multiple ports: List each `PublishPort` line

Container DNS options:
- `Dns` - DNS servers
- `DnsSearch` - Search domains
- `Hostname` - Container hostname

Network modes:
- `Network=network-name` - Use custom network
- `Network=host` - Use host network
- `Network=bridge` - Default bridge
- `Network=none` - No networking

## Docker Compatibility Socket

Podman provides a socket emulating Docker's interface. See [podman-socket](https://docs.podman.io/en/latest/markdown/podman-system-service.1.html)
for socket configuration and Docker compatibility details.

Socket locations:
- Rootless: `$XDG_RUNTIME_DIR/podman/podman.sock`
- System: `/run/podman/podman.sock`

Enable socket:
```bash
systemctl --user enable --now podman.socket  # Rootless
sudo systemctl enable --now podman.socket    # System
```

Use Docker CLI with Podman:
```bash
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock
docker ps  # Now uses Podman socket
```

## Systemd User Lingering

User lingering keeps systemd user manager running after logout, enabling persistent
user containers. See [loginctl(1)](https://www.freedesktop.org/software/systemd/man/loginctl.html)
for details.

Enable lingering:
```bash
loginctl enable-linger           # Current user
sudo loginctl enable-linger user # Specific user
```

Check status:
```bash
loginctl show-user $USER | grep Linger
systemctl --user status
systemctl status user@$(id -u).service
```

Required for persistent rootless containers.
Required for rootless containers to start on boot.

## Environment Variables and Configuration

Set container environment variables in quadlet files.

Container directives:
- `Environment=KEY=value` - Single variable
- `EnvironmentFiles=path` - Load from file

Example:
```ini
[Container]
Environment=DEBUG=true
Environment=LOG_LEVEL=info
EnvironmentFiles=/path/to/config.env
```

For sensitive data, mount config files as read-only volumes with restricted
permissions, or use proper container secret management (e.g., systemd credentials).

Verify in running container:
```bash
podman inspect container-name | jq '.Config.Env'
podman exec container-name env
```

See [Podman Container](https://docs.podman.io/en/latest/markdown/podman-container.5.html)
for environment variable syntax.

## Automatic Image Updates

The `podman-auto-update.timer` periodically checks for newer container images
and restarts containers if updates are found. See
[podman-auto-update(1)](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html)
for configuration details.

Enable auto-update in container:
```ini
[Container]
Image=quay.io/image:latest
AutoUpdate=image
```

Manage timer:
```bash
systemctl --user enable --now podman-auto-update.timer
systemctl --user status podman-auto-update.timer
systemctl --user start podman-auto-update.service  # Manual trigger
journalctl --user -u podman-auto-update.service -f
```

Customize schedule (rootless):
```bash
mkdir -p ~/.config/systemd/user/podman-auto-update.timer.d/
cat > ~/.config/systemd/user/podman-auto-update.timer.d/override.conf << 'EOF'
[Timer]
OnCalendar=daily
OnBootSec=15min
EOF
systemctl --user daemon-reload
systemctl --user restart podman-auto-update.timer
```

## Troubleshooting

Common issues and diagnostics:

**Unit not found after daemon-reload:**
```bash
# Check file location
ls -la ~/.config/containers/systemd/

# Validate quadlet syntax
systemd-analyze --user --generators=true verify name.container

# View generated systemd service
systemctl --user cat generated-service
```

**Image pull timeouts:**
```bash
# Check service logs
journalctl --user -u service-name -n 30

# Test pull manually
podman pull image:tag

# Increase timeout
[Service]
TimeoutStartSec=900
```

**Permission denied / cannot create container:**
```bash
# Check subuid/subgid allocation
grep "^$(id -u):" /etc/subuid /etc/subgid
```

**Network not found:**
```bash
# Verify network unit running
systemctl --user status app.network
podman network ls

# Add proper dependencies
Requires=app.network
After=app.network
```

**Containers stop on logout:**
```bash
# Check and enable lingering
loginctl show-user $USER | grep Linger
loginctl enable-linger
```

**Service logs and status:**
```bash
journalctl --user -u service-name -f
systemctl --user status service-name
podman logs container-name
podman inspect container-name | jq '.State'
```

## Health Checks and Resource Limits

See [Podman Container](https://docs.podman.io/en/latest/markdown/podman-container.5.html)
for health check and resource limit directives.

Health checks:
- `HealthCmd` - Command to run
- `HealthInterval` - Check frequency
- `HealthRetries` - Failures before unhealthy
- `HealthStartPeriod` - Grace period after start
- `HealthTimeout` - Command timeout

Resource limits:
- `CpuShares` - CPU weight
- `MemoryMax` - Memory limit
- `PidsLimit` - Process limit
- `MemorySwapMax` - Swap limit

Monitor resources:
```bash
podman stats container-name
podman inspect container-name | jq '.HostConfig'
```

## System vs Rootless Containers

| Aspect | System | Rootless |
|--------|--------|----------|
| Location | `/etc/containers/systemd/` | `~/.config/containers/systemd/` |
| Start/Stop | `sudo systemctl` | `systemctl --user` |
| Security | Runs as root | Process isolation via subuids |
| Persistence | Always running | Requires linger |
| Use Case | System services | User applications, development |

Rootless requires subuid/subgid allocation.

## Best Practices

Key considerations for container deployments:

- Use fully qualified image names: `registry.example.com/namespace/image:tag`
- Pin image versions in production (avoid `:latest`)
- Set resource limits (prevent resource exhaustion)
- Enable restart policies: `RestartPolicy=always` or `on-failure`
- Declare dependencies with `Requires=` and `After=`
- Use read-only root filesystem when possible
- Run as non-root user when possible
- Mount only necessary volumes
- Enable health checks for critical services
- Monitor logs regularly via journalctl

See [Podman Container](https://docs.podman.io/en/latest/markdown/podman-container.5.html)
for all available security and configuration options.

---

## Essential References

**Quadlets and Container Management:**
- [podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) - Quadlet unit syntax and directives
- [podman-container(5)](https://docs.podman.io/en/latest/markdown/podman-container.5.html) - Container unit options
- [podman-pod(5)](https://docs.podman.io/en/latest/markdown/podman-pod.5.html) - Pod unit options
- [podman-kube-play(1)](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html) - Kubernetes YAML support
- [podman-auto-update(1)](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html) - Image auto-update timer
- [podman-network(1)](https://docs.podman.io/en/latest/markdown/podman-network.1.html) - Network creation and management
- [podman-volume(1)](https://docs.podman.io/en/latest/markdown/podman-volume.1.html) - Volume management

**User/Rootless Configuration:**
- [subuid(5) / subgid(5)](https://man7.org/linux/man-pages/man5/subuid.5.html) - User/group ID delegation
- [Podman Rootless Mode](https://docs.podman.io/en/latest/markdown/podman.1.html#rootless-mode-configuration) - Rootless setup
- [loginctl(1)](https://www.freedesktop.org/software/systemd/man/loginctl.html) - User lingering

**Systemd Integration:**
- [systemd.unit(5)](https://www.freedesktop.org/software/systemd/man/systemd.unit.html) - Unit file syntax
- [systemd.service(5)](https://www.freedesktop.org/software/systemd/man/systemd.service.html) - Service options
- [systemctl(1)](https://www.freedesktop.org/software/systemd/man/systemctl.html) - Service control
- [journalctl(1)](https://www.freedesktop.org/software/systemd/man/journalctl.html) - Log viewing

## Debugging and Logs

Access logs via systemd or container directly:

```bash
# Systemd service logs
journalctl --user -u service-name -n 50
journalctl --user -u service-name -f            # Follow
journalctl --user -u service-name --since "1h"  # Time range

# Container logs
podman logs container-name -n 50
podman logs container-name -f --timestamps
podman logs container-name --since 10m

# Container inspection
podman inspect container-name | jq '.State'
podman inspect container-name | jq '.Config.Env'
podman inspect container-name | jq '.Mounts'

# Resource usage
podman stats container-name
```

See [journalctl(1)](https://www.freedesktop.org/software/systemd/man/journalctl.html)
and [podman-logs(1)](https://docs.podman.io/en/latest/markdown/podman-logs.1.html)
for complete options.

## Quick Reference

**Setup and Configuration:**
```bash
# Check rootless prerequisites
grep "^$(id -u):" /etc/subuid /etc/subgid          # User ID delegation
loginctl show-user $USER | grep Linger              # Lingering status
loginctl enable-linger                              # Enable persistence
```

**Quadlet Files:**
```bash
mkdir -p ~/.config/containers/systemd/              # Create directory
systemd-analyze --user --generators=true verify file.container  # Validate
systemctl --user daemon-reload                      # Reload quadlets
podman quadlet list                                 # List loaded quadlets
podman quadlet print file.container                 # Show generated service
```

**Service Control (Rootless):**
```bash
systemctl --user start service-name.container       # Start
systemctl --user stop service-name.container        # Stop
systemctl --user enable service-name.container      # Enable at boot
systemctl --user status service-name.container      # Check status
systemctl --user cat service-name.container         # Show generated unit
```

**Container Operations:**
```bash
podman ps                                           # List running
podman ps -a                                        # List all
podman logs container-name -f                       # Follow logs
podman inspect container-name | jq '.State'         # Check state
podman exec container-name command                  # Execute command
podman stats container-name                         # Resource usage
```

**Logging and Debugging:**
```bash
journalctl --user -u service-name.container -f      # Follow systemd logs
journalctl --user -u service-name.container -n 50   # Last 50 lines
systemctl --user status service-name.container -l   # Full status
```

**Networks and Volumes:**
```bash
podman network ls                                   # List networks
podman volume ls                                    # List volumes
podman network inspect network-name                 # Inspect network
podman volume inspect volume-name                   # Inspect volume
```
