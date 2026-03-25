# dh-virtualenv Patterns

Patterns for packaging Python projects using dh-virtualenv, derived from real projects
and dh-virtualenv best practices.

## Install Root Strategies

### Strategy 1: /opt/venvs/ (recommended default)

Clean separation from system Python. The venv lives entirely under `/opt/venvs/<pkg>/`.

```makefile
# debian/rules
export DH_VIRTUALENV_INSTALL_ROOT=/opt/venvs

override_dh_virtualenv:
	dh_virtualenv --python /usr/bin/python3 --builtin-venv \
		--upgrade-pip --preinstall "setuptools>=70" --preinstall "wheel" \
		--extra-pip-arg "--no-cache-dir"
```

Requires a `.links` file to expose entry points:
```
# debian/<pkg>.links
/opt/venvs/<pkg>/bin/<entry-point> /usr/bin/<entry-point>
```

### Strategy 2: /usr/lib/ with --install-suffix

Venv installs to `/usr/lib/<pkg>/`. More FHS-conventional but entry points are in
`/usr/lib/<pkg>/bin/` which isn't on PATH.

```makefile
# debian/rules
export DH_VIRTUALENV_INSTALL_ROOT = /usr/lib

override_dh_virtualenv:
	dh_virtualenv \
		--install-suffix <pkg> \
		--python /usr/bin/python3 \
		--builtin-venv \
		--preinstall setuptools \
		--preinstall wheel
```

Entry points are at `/usr/lib/<pkg>/bin/<entry-point>`. Reference this path directly
in the `.service` file's `ExecStart=`, or create a `.links` file.

### When to choose which

| Factor | /opt/venvs/ | /usr/lib/ |
|--------|-------------|-----------|
| FHS compliance | Technically /opt is for add-on software | Standard location |
| Simplicity | Cleaner — one obvious location | Need --install-suffix |
| Convention | Common in dh-virtualenv docs | Common in Debian ecosystem |
| System packages | Works fine | Required for --use-system-packages |

**Default to `/opt/venvs/`** unless you have a reason to use `/usr/lib/`.

## System Packages vs Full Vendoring

### Full vendoring (default)

All pip dependencies are bundled inside the venv. The package is self-contained.

```makefile
override_dh_virtualenv:
	dh_virtualenv --python /usr/bin/python3 --builtin-venv \
		--upgrade-pip --preinstall "setuptools>=70" --preinstall "wheel"
```

In `debian/control`, runtime Depends should only include:
```
Depends: ${misc:Depends}, python3
```

Do NOT list python3-* packages in Depends — they would be installed but unused.

### System packages (--use-system-packages)

The venv inherits system Python packages. Use when dependencies have heavy native
components that are painful to compile from source (numpy, onnxruntime, etc.).

```makefile
override_dh_virtualenv:
	dh_virtualenv \
		--install-suffix <pkg> \
		--python /usr/bin/python3 \
		--builtin-venv \
		--use-system-packages \
		--preinstall setuptools \
		--preinstall wheel
```

In `debian/control`, the system packages MUST appear in BOTH stanzas:
```
Build-Depends:
 ...,
 python3-numpy,
 python3-aiohttp

Depends: ${misc:Depends},
 python3,
 python3-numpy,
 python3-aiohttp
```

### The consistency trap

If `--use-system-packages` is NOT in rules but python3-* packages appear in Depends,
those packages are installed but ignored by the venv. This wastes disk space and
creates confusion about what's actually providing the dependency.

**Validation rule:** Check that `--use-system-packages` in rules matches python3-*
entries in Depends.

## Common Build Overrides

### dh_strip

Strip removes debug symbols from binaries. pip-installed wheels contain pre-built
binaries that strip doesn't understand.

```makefile
# Exclude pip-installed packages from stripping
override_dh_strip:
	dh_strip --exclude=site-packages
```

Or skip entirely:
```makefile
override_dh_strip:
	@echo "Skipping dh_strip"
```

### dh_shlibdeps

Checks shared library dependencies. Bundled .so files from pip wheels trigger errors
because their dependencies aren't tracked by dpkg.

```makefile
# Exclude site-packages from shlibdeps checking
override_dh_shlibdeps:
	dh_shlibdeps --exclude=site-packages
```

Or ignore missing info:
```makefile
override_dh_shlibdeps:
	dh_shlibdeps --dpkg-shlibdeps-params=--ignore-missing-info
```

### dh_strip_nondeterminism

Normalizes timestamps in archives for reproducible builds. Can cause issues with
some wheel formats.

```makefile
override_dh_strip_nondeterminism:
	true
```

## Standard Preinstalls

Always preinstall setuptools and wheel to ensure pip can build packages:

```makefile
--preinstall "setuptools>=70" --preinstall "wheel"
```

The `>=70` pin for setuptools avoids issues with older versions that don't support
modern pyproject.toml-based projects.

## Standard Build-Depends

Every dh-virtualenv package needs these:
```
Build-Depends:
 debhelper-compat (= 13),
 dh-virtualenv (>= 1.2),
 python3,
 python3-venv,
 python3-dev,
 python3-setuptools
```

Add if building packages with native extensions:
```
 libffi-dev,
 libssl-dev,
 build-essential
```

## The .links File

Creates symlinks during package installation. Essential for `/opt/venvs/` strategy
to expose entry points on PATH.

Format: `<source> <destination>` (one pair per line, absolute paths)

```
/opt/venvs/myapp/bin/myapp /usr/bin/myapp
```

For `/usr/lib/` strategy, you can either:
1. Use `.links` to create `/usr/bin/` symlinks, or
2. Reference `/usr/lib/<pkg>/bin/<entry-point>` directly in the service file

## The .install File

Copies files from the source tree into the package. Each line: `<source> <destination-dir>`

```
config.yaml /etc/myapp/
debian/myapp-ufw /etc/ufw/applications.d/
debian/myapp-ferm.conf /etc/ferm/ferm.d/
```

For files that need custom permissions or ownership, use `dh_install` override
in `debian/rules` with `install -m` commands instead.

## Lintian Overrides

dh-virtualenv packages trigger several lintian warnings that are expected and safe
to suppress. Create `debian/<pkg>.lintian-overrides`:

```
# dh-virtualenv bundles pip-installed packages — these are expected
__PKG__: embedded-library *
__PKG__: extra-license-file *
__PKG__: unusual-interpreter *
__PKG__: script-with-language-extension *

# Venv contains pre-built shared objects from pip wheels
__PKG__: hardening-no-fortify-functions *
__PKG__: hardening-no-pie *
__PKG__: library-not-linked-against-libc *
__PKG__: shared-library-lacks-prerequisites *

# /opt/venvs/ is not standard FHS but is conventional for dh-virtualenv
__PKG__: dir-or-file-in-opt *
__PKG__: file-in-unusual-dir *
```

Only include overrides for warnings your package actually generates. Build once,
check `lintian` output, then add the relevant overrides.

## Complete rules File Example

```makefile
#!/usr/bin/make -f
export DH_VIRTUALENV_INSTALL_ROOT=/opt/venvs

%:
	dh $@ --with python-virtualenv

override_dh_virtualenv:
	dh_virtualenv --python /usr/bin/python3 --builtin-venv \
		--upgrade-pip --preinstall "setuptools>=70" --preinstall "wheel" \
		--extra-pip-arg "--no-cache-dir"

override_dh_strip:
	dh_strip --exclude=site-packages

override_dh_shlibdeps:
	dh_shlibdeps --exclude=site-packages
```
