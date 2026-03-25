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

When you ask Claude Code to create or audit Debian packages, this skill enforces conventions that prevent subtle bugs:

- **Required files** — ensures every `debian/` directory has control, rules, changelog, copyright, source/format, and lintian-overrides
- **Cross-file consistency** — catches mismatches between rules, control, service files, dirs, and links that would pass lintian but fail at runtime
- **Maintainer scripts** — enforces idempotent postinst, debhelper-only prerm, correct `#DEBHELPER#` placement, and the conffile trap guard
- **Service hardening** — requires `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, and `ReadWritePaths` in every systemd unit
- **FHS compliance** — references the Filesystem Hierarchy Standard 3.0 for correct file placement
- **Build overrides** — ensures `dh_strip` and `dh_shlibdeps` exclusions are present for dh-virtualenv packages

The skill covers general Debian packaging (policy, FHS, dpkg behavior, conffiles, maintainer scripts, build commands) and includes specialized support for Python projects packaged with dh-virtualenv.

## Prerequisites

Install the packages needed for Debian package development:

```bash
# Essential — core packaging tools
sudo apt install build-essential debhelper devscripts \
    fakeroot lintian dpkg-dev

# For Python projects using dh-virtualenv
sudo apt install dh-virtualenv python3 python3-venv \
    python3-dev python3-setuptools

# Recommended — clean-room builds and changelog management
sudo apt install pbuilder sbuild

# Optional — useful tools
sudo apt install quilt git-buildpackage
```

| Package | What it does |
|---------|-------------|
| `build-essential` | gcc, make, and friends — needed for any compiled code |
| `debhelper` | The `dh` command framework and all `dh_*` helpers |
| `devscripts` | `debuild`, `debsign`, `dch`, `uscan`, and other maintainer tools |
| `fakeroot` | Run commands with fake root privileges for package building |
| `lintian` | Static analysis — catches policy violations in built packages |
| `dpkg-dev` | `dpkg-buildpackage`, `dpkg-source`, and low-level tools |
| `dh-virtualenv` | Builds Python virtualenvs inside .deb packages |
| `pbuilder` | Build in a clean chroot — catches undeclared Build-Depends |
| `sbuild` | Modern chroot builder used by Debian infrastructure |
| `quilt` | Patch management for `3.0 (quilt)` source format |
| `git-buildpackage` | `gbp` suite for managing Debian packages in git repos |

## Installation

Copy the skill directory into your Claude Code skills directory:

```bash
cp -r super-cow-powers/ ~/.claude/skills/super-cow-powers/
```

Or if using a plugin, place it under your plugin's `skills/` directory.

## What's inside

```
super-cow-powers/
├── SKILL.md                         # Conventions and gated validation checklists
├── references/
│   ├── debian-policy-summary.md     # Naming, versions, FHS 3.0, conffiles, dependency
│   │                                  types, control format, build commands
│   ├── dh-virtualenv-patterns.md    # Install roots, vendoring, build overrides
│   └── systemd-integration.md       # Service files, maintainer scripts, firewalls
└── templates/                       # Ready-to-use file templates (__PKG__ placeholders)
    ├── control, rules, changelog, copyright, source-format
    ├── service, postinst, prerm, postrm, default
    ├── dirs, links, install, lintian-overrides
    └── logrotate
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

## Roadmap

- **Programmatic cross-file validation** — the cross-file consistency checks (Checklist 4) are currently enforced by skill instructions to the model, not by a script. A `scripts/validate-debian.sh` that mechanically verifies install root vs links paths, ReadWritePaths vs dirs, EnvironmentFile vs default, etc. would make the checks reliable regardless of model compliance.

## License

This project is licensed under the GNU General Public License v3.0 — see [LICENSE](LICENSE) for details.
