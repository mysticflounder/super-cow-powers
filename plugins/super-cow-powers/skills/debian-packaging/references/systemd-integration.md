# Systemd Integration for Debian Packages

Patterns for integrating Python services with systemd via Debian packaging.

## Service File Template

```ini
[Unit]
Description=__PKG__ service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=__PKG__
Group=__PKG__
EnvironmentFile=-/etc/default/__PKG__
ExecStart=/usr/bin/__PKG__ --config /etc/__PKG__/config.yaml
Restart=on-failure
RestartSec=5

# Security hardening
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/log/__PKG__ /var/lib/__PKG__
ReadOnlyPaths=/etc/__PKG__

[Install]
WantedBy=multi-user.target
```

### Security Hardening Directives Explained

| Directive | What it does | When to modify |
|-----------|-------------|----------------|
| `NoNewPrivileges=yes` | Process can never gain new privileges (no setuid, no capability additions) | Almost never. Keep it. |
| `ProtectSystem=strict` | Mounts `/usr`, `/boot`, `/efi` read-only and prevents writing anywhere except explicitly allowed paths | Add `ReadWritePaths=` for each directory the service writes to |
| `ProtectHome=yes` | Makes `/home`, `/root`, `/run/user` inaccessible | Remove only if service genuinely needs home directory access |
| `PrivateTmp=yes` | Gives the service its own `/tmp` and `/var/tmp` isolated from other processes | Almost never remove |
| `ReadWritePaths=` | Whitelists directories for writing under `ProtectSystem=strict` | Add all directories the service creates files in |
| `ReadOnlyPaths=` | Explicit read-only access (documentation purpose under strict mode) | List config directories |

### Additional hardening (optional, for high-security services)

```ini
# Restrict network
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
# Restrict kernel interaction
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
# Restrict system calls
SystemCallFilter=@system-service
SystemCallArchitectures=native
# Lock down further
LockPersonality=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
MemoryDenyWriteExecute=yes
```

Only add these if you understand the implications. Some Python packages use
`ctypes` or `cffi` which require `MemoryDenyWriteExecute=no`.

### Network dependencies

- `After=network-online.target` + `Wants=network-online.target` — use for services
  that need network connectivity (API servers, clients)
- `After=network.target` — use for services that only need local sockets or don't
  need network at all

### ExecStart path

Depends on your install strategy:
- `/opt/venvs/` with `.links`: Use `/usr/bin/<entry-point>`
- `/usr/lib/` with `--install-suffix`: Use `/usr/lib/<pkg>/bin/<entry-point>`

## Maintainer Script Lifecycle

### Order of operations during install/upgrade

```
New install:          preinst install → unpack → postinst configure
Upgrade:              preinst upgrade → unpack → postinst configure <old-version>
Remove:               prerm remove → remove files → postrm remove
Purge (after remove): postrm purge
```

### postinst — Post-installation setup

```sh
#!/bin/sh
set -e

case "$1" in
    configure)
        # 1. Create system user (idempotent)
        if ! getent passwd __PKG__ >/dev/null 2>&1; then
            adduser --system --group --no-create-home \
                --home /nonexistent --shell /usr/sbin/nologin \
                __PKG__
        fi

        # 2. Set directory ownership
        chown __PKG__:__PKG__ /var/log/__PKG__
        chown __PKG__:__PKG__ /var/lib/__PKG__

        # 3. Generate secrets (GUARDED — do not overwrite on upgrade)
        if [ ! -f /etc/__PKG__/secret.key ]; then
            dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 -w0 > /etc/__PKG__/secret.key
            chown root:__PKG__ /etc/__PKG__/secret.key
            chmod 640 /etc/__PKG__/secret.key
        fi

        # 4. Reload firewall (optional)
        if systemctl is-active --quiet ferm 2>/dev/null; then
            systemctl reload ferm || true
        fi
        ;;
esac

#DEBHELPER#

exit 0
```

### System user creation patterns

**Daemon that doesn't need a home directory:**
```sh
adduser --system --group --no-create-home \
    --home /nonexistent --shell /usr/sbin/nologin __PKG__
```

**Daemon that needs a working directory:**
```sh
adduser --system --group --no-create-home \
    --home /var/lib/__PKG__ --shell /usr/sbin/nologin __PKG__
```

Key flags:
- `--system` — allocates UID in 100-999 range, no password aging
- `--group` — creates a group with the same name
- `--no-create-home` — don't create the home directory (we do it in `.dirs`)
- `--shell /usr/sbin/nologin` — prevent interactive login

### prerm — Pre-removal

For most packages, this should contain ONLY `#DEBHELPER#`:

```sh
#!/bin/sh
set -e

#DEBHELPER#

exit 0
```

**Why:** `dh_installsystemd` generates service stop/disable commands and inserts them
at `#DEBHELPER#`. If you manually stop the service, you're duplicating what debhelper
already does, and the manual stop may race with debhelper's generated code.

**Exception:** If you have non-service cleanup that must happen before file removal
(rare), add it before `#DEBHELPER#`.

### postrm — Post-removal

```sh
#!/bin/sh
set -e

case "$1" in
    purge)
        # Remove system user and group
        if getent passwd __PKG__ >/dev/null 2>&1; then
            deluser --system __PKG__ || true
        fi
        if getent group __PKG__ >/dev/null 2>&1; then
            delgroup --system __PKG__ || true
        fi

        # Remove data and log directories
        rm -rf /var/log/__PKG__
        rm -rf /var/lib/__PKG__

        # Config directory is removed by dpkg (conffiles) or:
        rm -rf /etc/__PKG__
        ;;
esac

#DEBHELPER#

exit 0
```

**purge vs remove:**
- `remove` — package files are deleted but config files remain. Light cleanup only.
- `purge` — everything goes. Remove users, groups, data, logs, config.

### #DEBHELPER# token placement

The token is replaced at build time by debhelper-generated code. Placement matters:

| Script | Place #DEBHELPER# | Why |
|--------|------------------|-----|
| postinst | AFTER your logic | Service should start after user/dirs/config are ready |
| prerm | AS the logic | debhelper handles service stop |
| postrm | AFTER purge cleanup | debhelper handles systemd unit file cleanup |

## /etc/default/ Environment Files

The environment file provides runtime configuration without modifying the service file.

```sh
# /etc/default/__PKG__
# Configuration for __PKG__ service
# This file is sourced by systemd via EnvironmentFile=

# Bind address and port
__UPPER_PKG___HOST=0.0.0.0
__UPPER_PKG___PORT=8080

# Log level
__UPPER_PKG___LOG_LEVEL=info
```

### How it works

1. Service file includes: `EnvironmentFile=-/etc/default/__PKG__`
2. The `-` prefix means systemd won't fail if the file doesn't exist
3. Variables are available to `ExecStart=` via `${VARIABLE_NAME}`
4. The file is installed by the package (via `.install` or `dh_install`)
5. If installed to `/etc/`, it's automatically a conffile (dpkg protects it)

### Rules for /etc/default files

- POSIX sh format: `KEY=value` (no export, no complex syntax)
- Comments start with `#`
- The service must have sane defaults if the file is missing
- Never source this file from maintainer scripts — it's only for systemd

## Firewall Integration (Optional)

### ferm

Install a rules fragment to `/etc/ferm/ferm.d/`:

```
# debian/<pkg>-ferm.conf
@def $__UPPER_PKG___PORT = 8080;

chain INPUT proto tcp dport $__UPPER_PKG___PORT {
    mod state state (NEW) ACCEPT;
}
```

Install via `.install` file:
```
debian/__PKG__-ferm.conf /etc/ferm/ferm.d/
```

Reload in postinst:
```sh
if systemctl is-active --quiet ferm 2>/dev/null; then
    systemctl reload ferm || true
fi
```

### ufw

Install an application profile to `/etc/ufw/applications.d/`:

```
[__PKG__]
title=__PKG__ service
description=__PKG__ network service
ports=8080/tcp
```

Reload in postinst:
```sh
if command -v ufw >/dev/null 2>&1 && ufw status | grep -q "^Status: active"; then
    ufw app update __PKG__ || true
fi
```

## Idempotency Patterns

Every operation in maintainer scripts must be safe to repeat:

| Operation | Idempotent pattern |
|-----------|-------------------|
| Create user | `if ! getent passwd pkg; then adduser ...; fi` |
| Create directory | `mkdir -p /path` |
| Create file | `if [ ! -f /path ]; then ...; fi` |
| Set ownership | `chown user:group /path` (always safe to repeat) |
| Set permissions | `chmod 0640 /path` (always safe to repeat) |
| Remove user | `if getent passwd pkg; then deluser ...; fi` |
| Remove directory | `rm -rf /path` (always safe to repeat) |
| Reload service | `systemctl reload svc \|\| true` |
