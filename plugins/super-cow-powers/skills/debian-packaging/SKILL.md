---
name: super-cow-powers
description: >-
  Use when creating debian/ directories for Python projects, modifying debian/control
  or debian/rules, building .deb packages with dh-virtualenv or dpkg-buildpackage,
  debugging dh_shlibdeps or dh_strip errors, fixing postinst/prerm/postrm scripts,
  writing systemd .service files for Python daemons, auditing existing debian/
  packaging against Debian policy, troubleshooting conffile overwrites on upgrade,
  resolving lintian warnings about embedded libraries, or deciding between
  --use-system-packages and full vendoring. Use this skill even for simple questions
  about .deb packaging, dpkg, FHS paths, or maintainer script patterns — it
  enforces conventions that prevent subtle consistency bugs.
---

```
                 (__)
                 (oo)
           /------\/
          / |    ||
         *  /\---/\
            ~~   ~~
..."Have you mooed today?"...
```

# Debian Packaging — Convention Enforcer

This skill enforces consistent Debian packaging conventions for Python projects
using dh-virtualenv. It catches the subtle consistency errors and missing files that
are easy to overlook.

You already know how to create Debian packages. This skill tells you **the rules**
and makes sure you follow them.

## The Gate

Before completing any packaging operation — new packaging, audit, or modification —
you must pass every applicable checklist. If a box can't be checked, stop and fix it.
Don't ship packages that fail the gate.

```
BEFORE declaring packaging complete:

1. IDENTIFY: Which checklists apply? (all packages, daemon, CLI, audit)
2. CHECK: Walk through every item in each applicable checklist
3. VERIFY: Cross-file consistency checks pass (the #1 source of subtle bugs)
4. ONLY THEN: Declare the packaging complete

Skip a checklist = ship a broken package
```

## Always: Use dh-virtualenv

All Python projects use dh-virtualenv for packaging. Never use pybuild, dh-python,
or plain setuptools for these packages. The `debian/rules` file must use
`dh $@ --with python-virtualenv`.

---

## Checklists

### Checklist 1: Required Files (every package)

Every package must have these files. No exceptions.

- [ ] `control` — has Source, Package, Maintainer, Build-Depends, Architecture, Depends, Description
- [ ] `rules` — uses `dh $@ --with python-virtualenv`, is executable (`chmod +x`)
- [ ] `changelog` — valid format: package name, version, distribution, maintainer, RFC 2822 date
- [ ] `copyright` — DEP-5 format with `Format:` URL
- [ ] `source/format` — contains `3.0 (native)` or `3.0 (quilt)`
- [ ] `<pkg>.lintian-overrides` — has dh-virtualenv warning suppressions

Can't check all boxes? The package will either fail to build or fail lintian review.

### Checklist 2: Daemon packages (services)

In addition to Checklist 1, daemon packages need:

- [ ] `<pkg>.service` — has User=, Group=, ExecStart=, Restart=, and all security hardening directives (see Convention 4)
- [ ] `<pkg>.postinst` — has `#!/bin/sh`, `set -e`, `case "$1" in configure)`, idempotent user creation, `#DEBHELPER#`, `exit 0`
- [ ] `<pkg>.prerm` — contains ONLY `#DEBHELPER#` (no manual service stop)
- [ ] `<pkg>.postrm` — has `case "$1" in purge)` with user/directory cleanup, `#DEBHELPER#`
- [ ] `<pkg>.default` — environment file for systemd `EnvironmentFile=`
- [ ] `<pkg>.dirs` — lists `/etc/<pkg>`, `/var/log/<pkg>`, `/var/lib/<pkg>`
- [ ] `<pkg>.links` — maps venv entry point to `/usr/bin/<pkg>`
- [ ] `<pkg>.install` — installs config files, firewall rules, etc.

Optional: `<pkg>.logrotate` (only if service writes to `/var/log/` instead of journald).

Can't check all boxes? The service will either fail to start, fail to stop cleanly,
or leave orphaned users/directories on purge.

### Checklist 3: CLI tool packages

In addition to Checklist 1, CLI tools need:

- [ ] `<pkg>.links` — maps venv entry point to `/usr/bin/<pkg>`
- [ ] NO `.service`, `.postinst` (with user creation), `.prerm`, `.postrm`, `.default`, `.dirs` — these are daemon-only files

Creating daemon files for a CLI tool wastes disk space and confuses operators who
see a system user and service unit for what should be a simple command.

### Checklist 4: Cross-file consistency

This is where the subtle bugs hide. Run this on EVERY packaging operation.

- [ ] `--use-system-packages` in rules ↔ python3-* in Depends match (see Convention 2)
- [ ] Install root in rules (`/opt/venvs/` or `/usr/lib/`) ↔ paths in `.links` file match
- [ ] `ReadWritePaths=` in .service ↔ directories in `.dirs` match
- [ ] `ExecStart=` path in .service ↔ symlink target in `.links` match
- [ ] `EnvironmentFile=` in .service ↔ `.default` file exists
- [ ] `dh_strip --exclude=site-packages` override present in rules
- [ ] `dh_shlibdeps --exclude=site-packages` override present in rules

A single cross-file mismatch can cause the package to install fine but the service
to crash at runtime — the hardest kind of bug to diagnose.

### Checklist 5: Maintainer script safety

Every maintainer script must pass:

- [ ] Shebang is `#!/bin/sh` (POSIX, not bash — dash is the default /bin/sh on Debian)
- [ ] `set -e` immediately after shebang
- [ ] `#DEBHELPER#` token present
- [ ] `exit 0` at end
- [ ] All operations are idempotent (safe to run on every upgrade, not just first install)
- [ ] Generated config files guarded with `if [ ! -f ... ]` (the conffile trap — see Convention 3)

| Script | #DEBHELPER# placement | Why |
|--------|----------------------|-----|
| postinst | AFTER your logic | Service starts after setup is complete |
| prerm | AS the logic | Debhelper handles service stop |
| postrm | AFTER purge cleanup | Debhelper cleans up systemd units |

### Checklist 6: Auditing existing packages

When auditing a `debian/` directory, follow this process:

```
1. READ: Every file in debian/ before reporting issues
2. CROSS-CHECK: Run Checklist 4 against the actual file contents
3. VERIFY: Only report issues you can point to in the actual text
4. NEVER: Invent whitespace errors, missing fields, or issues you can't prove

Hallucinated audit findings are worse than missing real ones —
they waste time and erode trust.
```

---

## Conventions

### Convention 1: File Naming

Maintainer scripts and service files use the package-name prefix:
`<pkg>.postinst`, `<pkg>.prerm`, `<pkg>.postrm`, `<pkg>.service`, etc.

Not bare `postinst`, `prerm`, etc. — bare names only work for single-binary source
packages and are ambiguous when a source package produces multiple binaries.

### Convention 2: System Packages Consistency

This is the #1 consistency error in dh-virtualenv packages.

```
IF rules has --use-system-packages:
    → python3-* packages MUST appear in BOTH Build-Depends AND Depends
    → These are real runtime dependencies the venv inherits

IF rules does NOT have --use-system-packages:
    → NO python3-* packages in Depends (except bare python3)
    → The venv bundles everything — system packages would be installed but ignored
```

When auditing: cross-reference `debian/rules` and `debian/control` for this
mismatch first. It's the most common issue.

### Convention 3: Maintainer Scripts

**prerm: Do NOT manually stop services**

```sh
#!/bin/sh
set -e
# This is ALL that should be here.
# dh_installsystemd inserts service stop/disable at #DEBHELPER#.
# Manual systemctl stop is redundant and can race with debhelper.
#DEBHELPER#
exit 0
```

**postinst: Guard everything**

Every file creation and user creation must be idempotent. The postinst runs on
every upgrade, not just first install.

```sh
# User creation — guard with getent
if ! getent passwd __PKG__ >/dev/null 2>&1; then
    adduser --system --group --no-create-home \
        --home /nonexistent --shell /usr/sbin/nologin __PKG__
fi

# THE CONFFILE TRAP
# Files created by postinst are NOT conffiles. dpkg won't protect them.
# Without this guard, user changes get overwritten on every upgrade.
if [ ! -f /etc/__PKG__/secret.key ]; then
    generate_secret > /etc/__PKG__/secret.key
fi
```

### Convention 4: Service Security Hardening

Every `.service` file must include these directives. They are non-negotiable defaults
because a compromised service running without them has write access to the entire
filesystem.

```ini
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/log/__PKG__ /var/lib/__PKG__
ReadOnlyPaths=/etc/__PKG__
```

Use `EnvironmentFile=-/etc/default/__PKG__` (the `-` prefix means don't fail if missing).

### Convention 5: dh-virtualenv Build Overrides

Every `debian/rules` must include these overrides. Without them, `dh_strip` and
`dh_shlibdeps` will fail on the pre-built .so files inside pip wheels — this is a
guaranteed build failure, not a theoretical risk.

```makefile
override_dh_strip:
	dh_strip --exclude=site-packages

override_dh_shlibdeps:
	dh_shlibdeps --exclude=site-packages
```

---

## Templates

Templates for all files are in the `templates/` directory. Use `__PKG__` as
placeholder for the package name.

## Reference Files

Read these only when you need the details they cover:

- `references/dh-virtualenv-patterns.md` — Install root strategies (/opt/venvs/ vs /usr/lib/),
  vendoring decisions, complete rules examples, .links and .install patterns
- `references/systemd-integration.md` — Service file template, maintainer script lifecycle,
  user creation patterns, firewall integration, idempotency patterns
- `references/debian-policy-summary.md` — Naming rules, version format, FHS paths,
  conffile handling, control file format, dependency types, changelog/copyright format,
  how to build packages
