# super-cow-powers

A Claude Code skill for Debian packaging. Conventions, templates, and validation checklists — with deep dh-virtualenv support for Python projects.

```
                 (__)
                 (oo)
           /------\/
          / |    ||
         *  /\---/\
            ~~   ~~
..."Have you mooed today?"...
```

## What it does

When you ask Claude Code to create or audit `debian/` directories for Python projects, this skill ensures:

- All required files are present (control, rules, changelog, copyright, source/format, lintian-overrides)
- Cross-file consistency (e.g., `--use-system-packages` in rules matches python3-* in Depends)
- Maintainer scripts follow correct patterns (idempotent postinst, debhelper-only prerm)
- Service files include security hardening directives
- dh-virtualenv build overrides prevent common build failures

## Prerequisites

Install the packages needed for Debian package development:

```bash
# Essential — you need all of these
sudo apt install build-essential debhelper dh-virtualenv devscripts \
    python3 python3-venv python3-dev python3-setuptools \
    fakeroot lintian

# Recommended — clean-room builds and changelog management
sudo apt install pbuilder sbuild dch

# Optional — useful tools
sudo apt install dpkg-dev quilt git-buildpackage
```

| Package | What it does |
|---------|-------------|
| `build-essential` | gcc, make, and friends — needed for any compiled code |
| `debhelper` | The `dh` command framework and all `dh_*` helpers |
| `dh-virtualenv` | The `dh_virtualenv` helper that builds Python venvs inside .deb packages |
| `devscripts` | `debuild`, `debsign`, `dch`, `uscan`, and other maintainer tools |
| `fakeroot` | Run commands with fake root privileges for package building |
| `lintian` | Static analysis — catches policy violations in built packages |
| `pbuilder` | Build in a clean chroot — catches undeclared Build-Depends |
| `sbuild` | Modern chroot builder used by Debian infrastructure |
| `dpkg-dev` | `dpkg-buildpackage`, `dpkg-source`, and low-level tools |
| `quilt` | Patch management for `3.0 (quilt)` source format |
| `git-buildpackage` | `gbp` suite for managing Debian packages in git repos |

## Installation

Copy the `super-cow-powers/` directory into your Claude Code skills directory:

```bash
cp -r super-cow-powers/ ~/.claude/skills/super-cow-powers/
```

Or if using a plugin, place it under your plugin's `skills/` directory.

## Structure

```
super-cow-powers/
├── SKILL.md                    # Main skill — conventions and validation checklist
├── references/
│   ├── dh-virtualenv-patterns.md   # Install roots, vendoring, build overrides
│   ├── systemd-integration.md      # Service files, maintainer scripts, firewalls
│   └── debian-policy-summary.md    # Naming, versions, FHS, conffiles, control format
├── templates/                  # Ready-to-use file templates with __PKG__ placeholders
│   ├── rules
│   ├── control
│   ├── service
│   ├── postinst
│   ├── prerm
│   ├── postrm
│   ├── changelog
│   ├── copyright
│   ├── source-format
│   ├── lintian-overrides
│   ├── dirs
│   ├── links
│   ├── install
│   ├── default
│   └── logrotate
└── evals/
    └── evals.json              # Test cases for skill validation
```

## Contributing

Contributions are welcome, including AI-generated PRs. All PRs must meet these requirements:

### General rules

- **Test before submitting.** If your change modifies conventions or templates, run the evals (`evals/evals.json`) against the updated skill and confirm they still pass.
- **One concern per PR.** Don't bundle unrelated convention changes, template fixes, and reference updates in a single PR.
- **Explain the why.** Convention changes need justification — link to Debian policy, dh-virtualenv docs, or a real-world failure that motivated the change.

### Rules for AI-generated PRs

We gladly accept AI-generated PRs. They are held to the same standard as human PRs, plus:

- **Disclose the tooling.** State which AI tool generated the PR (Claude Code, Copilot, etc.) in the PR description.
- **No hallucinated policy.** Every claim about Debian policy, dpkg behavior, or dh-virtualenv must be verifiable against official documentation. If you cite a policy section, quote the relevant text.
- **No speculative conventions.** Don't add conventions that "seem like a good idea" without evidence they prevent real issues. This skill is opinionated by design — each opinion must be grounded.
- **Verify templates build.** If you modify a template, confirm that a package built with the modified template actually builds with `dpkg-buildpackage`. "It looks right" is not sufficient.
- **Don't inflate.** Resist the urge to add tangentially related content, extra sections, or "helpful" expansions that weren't asked for. The skill is intentionally concise.

## License

This project is licensed under the GNU General Public License v3.0 — see [LICENSE](LICENSE) for details.
