# Debian Policy & FHS Summary

Condensed from:
- https://www.debian.org/doc/debian-policy/ (Debian Policy Manual)
- https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html (Filesystem Hierarchy Standard 3.0)
- https://www.debian.org/doc/manuals/maint-guide/ (New Maintainers' Guide)

When in doubt, consult the full documents.

## Package Naming

- Lowercase letters, digits, `+`, `-`, `.` only
- Must start with an alphanumeric character
- Must be unique within the Debian archive
- Recommended convention for Python packages: use the project name, lowercased
- No script file extensions (don't name a package `myapp.py`)

## Version String Format

Full format: `[epoch:]upstream_version[-debian_revision]`

| Component | Format | Example | Notes |
|-----------|--------|---------|-------|
| epoch | Integer | `1:` | Only used to correct ordering mistakes. Omit unless needed. |
| upstream_version | Alphanumeric + `.` `+` `-` `~` `:` | `2.1.0` | Must start with a digit. |
| debian_revision | After the last `-` | `-1` | Omit for native packages. |

### Date-based versions
- Format: `YYYY.MM.DD` (periods as separators)
- For native packages, hyphens are prohibited in the version
- Epoch-based timestamps (e.g., `0.1772086571`) work but are unconventional

### Comparison rules
- Versions are compared left to right
- `~` sorts before anything (even empty string) — useful for pre-releases: `1.0~rc1` < `1.0`
- Letters sort before digits in the non-digit portion

## Filesystem Hierarchy Standard (FHS 3.0)

The FHS classifies files along two dimensions:

|            | Shareable              | Unshareable        |
|------------|------------------------|--------------------|
| **Static** | `/usr`, `/opt`         | `/etc`, `/boot`    |
| **Variable** | `/var/mail`, `/var/spool/news` | `/var/run`, `/var/lock` |

- **Shareable** files can be stored on one host and used on others
- **Static** files don't change without admin intervention; can live on read-only media

### Package-relevant directories

| Path | Purpose | Package rules |
|------|---------|--------------|
| `/etc/<pkg>/` | Configuration files | Auto-detected as conffiles by debhelper compat 13+. No binaries allowed under `/etc/`. |
| `/etc/default/<pkg>` | Environment file for systemd | POSIX sh format, sourced via `EnvironmentFile=` |
| `/etc/opt/<pkg>/` | Config for add-on software in `/opt/` | Required if package installs to `/opt/` |
| `/usr/bin/` | User commands | Symlink from venv entry point. No subdirectories allowed. |
| `/usr/sbin/` | System admin commands | Rarely needed for Python packages |
| `/usr/lib/<pkg>/` | Package-private files | Alternative venv install root. May also store internal binaries not for direct user execution. |
| `/usr/share/<pkg>/` | Architecture-independent data | Read-only, shareable across architectures, no executables |
| `/var/lib/<pkg>/` | Variable state data | Service working directory, databases. Must remain valid across reboots. Users should never modify these to configure the app. |
| `/var/log/<pkg>/` | Log files | Add logrotate configuration. Not shareable across hosts. |
| `/var/cache/<pkg>/` | Cached data | Regenerable — apps MUST handle cache files being deleted. Unlike `/var/spool`, cache can be safely removed without permanent loss. |
| `/var/opt/<pkg>/` | Variable data for `/opt/` packages | Required complement to `/opt/<pkg>/` for runtime data |
| `/opt/venvs/<pkg>/` | Self-contained venv | Common dh-virtualenv install root. Static data only — variable data goes in `/var/opt/`, config in `/etc/opt/`. |
| `/run/` | Runtime variable data | Cleared at boot. PID files go here (`<program>.pid`). Not writable by unprivileged users. `/var/run` may be a symlink to `/run`. |
| `/srv/` | Data served to other hosts | For services providing data (web, FTP). Not for internal state — use `/var/lib/` for that. Packages must not assume specific subdirectories exist. |
| `/tmp/` | Temporary files | Programs must NOT assume files persist between invocations. May be cleared at boot. |
| `/var/tmp/` | Temporary files preserved across reboots | Unlike `/tmp/`, must NOT be cleared at boot. Longer retention. |

### Prohibited locations

| Path | Rule |
|------|------|
| `/usr/local/` | Reserved for the local administrator. Packages MUST NEVER place files here. |
| `/bin/`, `/sbin/`, `/lib/` | Symlinks to `/usr/` counterparts on modern systems; only `base-files` may use them. |
| `/mnt/` | For temporary admin mounts only. Installation programs MUST NOT use `/mnt/`. |

### Root filesystem design

The root filesystem must be kept as small as possible. Applications MUST NOT create
new subdirectories in the root hierarchy. Non-essential utilities belong in `/usr/`.

## Configuration Files (conffiles)

### Automatic detection
Debhelper (compat 13+) automatically marks files installed to `/etc/` as conffiles.
You almost never need an explicit `debian/conffiles` file.

### dpkg behavior on upgrade
When upgrading, dpkg compares checksums:
- **User hasn't modified, new version differs:** Replace silently
- **User has modified, new version differs:** Prompt user to keep/replace
- **User has modified, new version same:** Keep user's version

### The postinst trap
Files created dynamically by postinst are NOT conffiles. dpkg doesn't track them.
If postinst recreates them unconditionally on upgrade, user changes are lost.

**Rule:** Always guard dynamic file creation:
```sh
if [ ! -f /etc/myapp/config.yaml ]; then
    create_default_config > /etc/myapp/config.yaml
fi
```

### When to use explicit debian/conffiles
Only needed for conffiles outside `/etc/` (extremely rare). If all your config is
under `/etc/`, debhelper handles everything.

### Never modify conffiles from maintainer scripts
dpkg tracks checksums. If a maintainer script modifies a conffile, dpkg thinks the
user changed it and will prompt on every upgrade.

## Control File

### Required fields (source stanza)

| Field | Description | Example |
|-------|-------------|---------|
| `Source` | Source package name | `myapp` |
| `Section` | Archive area | `net`, `utils`, `python`, `web`, `misc` |
| `Priority` | Installation priority | `optional` (almost always) |
| `Maintainer` | Name and email | `Name <email@example.com>` |
| `Build-Depends` | Build-time dependencies | `debhelper-compat (= 13), dh-virtualenv (>= 1.2)` |
| `Standards-Version` | Policy version | `4.6.2` |
| `Rules-Requires-Root` | Whether build needs root | `no` (use this unless you know otherwise) |

### Required fields (binary stanza)

| Field | Description | Example |
|-------|-------------|---------|
| `Package` | Binary package name | `myapp` |
| `Architecture` | Target arch | `any`, `all`, or specific like `amd64` |
| `Depends` | Runtime dependencies | `${misc:Depends}, python3` |
| `Description` | Synopsis + extended | See format below |

### Description format
```
Description: Short synopsis under 80 characters
 Extended description starting with a space.
 Each line indented by exactly one space.
 .
 Empty lines in extended description use a single dot.
```

- Synopsis: Don't repeat the package name, don't end with a period
- Extended: Explain what the package does and why someone would install it

### Dependency types

| Type | Meaning | When to use |
|------|---------|-------------|
| `Depends` | Required for operation; install blocked if unavailable | Runtime dependencies |
| `Recommends` | Strongly suggested; installed by default by apt | Packages that enhance core functionality |
| `Suggests` | Optional enhancements; not installed automatically | Nice-to-have extras |
| `Pre-Depends` | Must be installed AND configured before install begins | Rarely needed — only when preinst needs the package |
| `Conflicts` | Cannot coexist; old package removed first | Mutually exclusive alternatives |
| `Breaks` | Installed package damages the listed packages | When your upgrade breaks an older version of another package |
| `Provides` | Supplies a virtual package name | Implementing a common interface (e.g., `mail-transport-agent`) |
| `Replaces` | Overwrites files from listed packages | When taking over files that used to belong to another package |

Version constraints: `<<` (strictly less), `<=`, `=`, `>=`, `>>` (strictly greater).

### Dependency substitution variables
- `${misc:Depends}` — always include (debhelper-generated deps)
- `${shlibs:Depends}` — include if package has compiled shared libraries
- `${python3:Depends}` — for standard Python packaging (not dh-virtualenv)

## Changelog Format

```
package (version) distribution; urgency=level

  * Change description.

 -- Maintainer Name <email>  Day, DD Mon YYYY HH:MM:SS +ZZZZ
```

- Two spaces before `*` in change entries
- One space before `--` in trailer line
- Two spaces between email and date
- Date must be RFC 2822 format
- Distribution: `unstable`, `stable`, or suite name (e.g., `trixie`)
- Urgency: `low`, `medium`, `high`, `emergency`, `critical`

Generate entries with `dch` (from devscripts package) to avoid format mistakes.

## Copyright Format (DEP-5)

Machine-readable format:

```
Format: https://www.debian.org/doc/packaging-manuals/copyright-format/1.0/

Files: *
Copyright: YYYY Author Name
License: MIT
 Full license text or reference.

Files: debian/*
Copyright: YYYY Packager Name
License: MIT
 Same as above.
```

- `Format:` field is required and must be the exact URL above
- Group files by license
- For proprietary/private packages, use `License: Proprietary`
- For standard licenses (GPL, LGPL, Apache, BSD, etc.), reference `/usr/share/common-licenses/` instead of reproducing the full text

## Maintainer Scripts

### Required patterns
- Start with `#!/bin/sh` (not bash — must be POSIX-compatible, dash is `/bin/sh` on Debian)
- Include `set -e` immediately after shebang
- Include `#DEBHELPER#` token (debhelper inserts generated code here)
- End with `exit 0`

### Argument handling

**postinst:**
```sh
case "$1" in
    configure)
        # Your setup logic here
        ;;
esac
```

**postrm:**
```sh
case "$1" in
    purge)
        # Cleanup: remove users, data, logs
        ;;
    remove)
        # Lighter cleanup (optional)
        ;;
esac
```

### Idempotency requirement
All operations must be safe to run multiple times. Scripts may be called repeatedly
due to errors, interruptions, or dpkg recovery operations.

### Available commands
At postinst time, all dependencies are configured. At preinst/postrm time, only
essential packages and pre-dependencies are available — use only basic commands.

## File Permissions

Default permissions for files installed by packages:
- Regular files: `0644` (owner read/write, others read)
- Executables: `0755` (owner read/write/execute, others read/execute)
- Directories: `0755`
- All owned by `root:root` unless specific needs dictate otherwise

Use `dpkg-statoverride` mechanism for admin-customizable permissions.

## Building Packages

### Quick reference

| Command | What it does |
|---------|-------------|
| `dpkg-buildpackage -us -uc` | Build without signing (local testing) |
| `debuild` | Build + run lintian (recommended for development) |
| `pbuilder --build foo.dsc` | Build in clean chroot (catches missing Build-Depends) |
| `sbuild` | Modern chroot builder (used by Debian infrastructure) |
| `fakeroot debian/rules binary` | Quick rebuild without full source package (testing only) |
| `lintian *.changes` | Static analysis of built package |

### Build process

`dpkg-buildpackage` runs these steps in order:
1. `debian/rules clean` — clean previous build artifacts
2. `dpkg-source -b` — build the source package
3. `debian/rules build` — compile the software
4. `fakeroot debian/rules binary` — create `.deb` files
5. Generate `.dsc` and `.changes` files

### Output files (native package)

| File | Purpose |
|------|---------|
| `<pkg>_<ver>.dsc` | Source package description |
| `<pkg>_<ver>.tar.xz` | Source tarball (native) or `<pkg>_<ver>.debian.tar.xz` (non-native) |
| `<pkg>_<ver>_<arch>.deb` | Binary package |
| `<pkg>_<ver>_<arch>.changes` | Upload metadata |
| `<pkg>_<ver>_<arch>.buildinfo` | Build environment record |

### Tips

- Always run `lintian` after building — it catches most policy violations
- Use `pbuilder` or `sbuild` before releasing — they build in a clean chroot and catch undeclared Build-Depends
- For iterating locally, `fakeroot debian/rules binary` is fast but skips source package creation
- Sign with `debsign` before uploading to any repository
