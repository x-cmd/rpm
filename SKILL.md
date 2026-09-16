---
name: rpm
description: The x-cmd RPM (Fedora / RHEL / openSUSE) repository — flat Packages/ + repodata/ layout, noarch RPM, mirrored verbatim to https://www.x-cmd.com/rpm/. Use when navigating this repo's structure, identifying which file to edit for a given task, or understanding the sync model.
metadata: type=rpm-repository, scope=stable, refresh=daily-mirror, signed=no
---

# x-cmd/rpm — repo organization reference

> A structural map of this repository. Not a how-to-install
> guide (that's in `README.md` / `README.cn.md` / `docs/0-install.*`).
> This file is for an agent or maintainer coming back to the
> repo and asking "where do I edit X?".

## What this repo is

A canonical yum/dnf/zypper RPM repository for the x-cmd
package. The contents are pure RPM data — `Packages/*.rpm`
plus `repodata/` — alongside the human-facing metadata
(`x-cmd.repo`, `README*.md`, `docs/`) that lives alongside
any RPM repo.

The whole tree is mirrored verbatim to
<https://www.x-cmd.com/rpm/> on a daily cadence. `dnf`,
`yum`, and `zypper` read `Packages/` + `repodata/`; humans
see the rest.

## Top-level layout

```
.
├── Packages/                    # RPM payloads (the .rpm files)
│   └── x-cmd_0.9.9_all.rpm
├── repodata/                    # createrepo_c metadata
│   ├── repomd.xml
│   ├── <sha>-primary.xml.gz
│   ├── <sha>-filelists.xml.gz
│   └── <sha>-other.xml.gz
├── x-cmd.repo                   # ready-to-paste dnf/yum config
├── README.md                    # English README (GitHub renders)
├── README.cn.md                 # Chinese README (GitHub renders)
├── SKILL.md                     # ← this file: structural reference
├── CONTRIBUTING.md              # maintainer runbook (release, signing, incidents)
└── docs/                        # user-facing bilingual docs (see below)
```

## Where to edit what

| If you want to… | Edit |
| --- | --- |
| Add a new `.rpm` version | `Packages/x-cmd_<ver>-<rel>.noarch.rpm` (drop in), then regenerate metadata (see CONTRIBUTING.md) |
| Update the repo snippet | `x-cmd.repo` |
| Refresh the repodata | run `createrepo_c` (see CONTRIBUTING.md) → updates `repodata/` |
| Change the user-facing install doc | `docs/0-install.{md,cn.md,en.md,llms.md,faq.yml}` |
| Add a new docs topic | a new `docs/<n>-<topic>.{md,cn.md,en.md,llms.md,faq.yml}` set |
| Update English README | `README.md` |
| Update Chinese README | `README.cn.md` |
| Change the GPG key, signing, or `gpgcheck` policy | `CONTRIBUTING.md` (runbook) + `docs/2-security.*` (user-facing) |
| Fix a broken release | `CONTRIBUTING.md` → "Incident playbook" |

## Conventions

This repo mirrors the `x-cmd/deb` and `x-cmd/cve` repos'
conventions. The short version:

- **Flat layout, not Debian's `pool/` + `dists/`** — RPM
  doesn't have Debian's per-suite / per-component matrix;
  `Packages/` + `repodata/` are at the root.
- **Single suite, noarch** — x-cmd is `noarch` so one RPM
  serves every architecture (x86_64, aarch64, ppc64le,
  s390x, etc.). No per-arch indices needed.
- **Numbered prefix on docs topics** — `0-install`,
  `1-packaging`, `2-security`, `3-stats`. Lower numbers lead.
- **Five file variants per topic** —
  `.md` (default = English), `.cn.md` (Chinese), `.en.md`
  (explicit English), `.llms.md` (LLM-friendly structured),
  `.faq.yml` (FAQ structured).
- **YAML frontmatter** on every `.md` and `.llms.md`:
  `x-title`, `x-desc`, `x-sidebar`, `x-keywords`, `x-json-ld`
  (TechArticle schema).
- **Bilingual parity** — every English doc has a Chinese
  counterpart with the same filename stem.
- **No build step** for `docs/` — the markdown files ARE the
  deliverable.

## `docs/` layout

```
docs/
├── 0-install.{md, cn.md, en.md, llms.md, faq.yml}    # how to install x-cmd
├── 1-packaging.{md, cn.md, en.md, llms.md, faq.yml}  # how to build .rpm + host repo
├── 2-security.{md, cn.md, en.md, llms.md, faq.yml}   # gpgcheck=0 reality + GPG plan
├── 3-stats.{md, cn.md, en.md, llms.md, faq.yml}      # current repo snapshot
└── assets/                                            # images, badges (reserved)
```

Each numbered topic covers one user question:

- `0` — "how do I install x-cmd?"
- `1` — "how do I host my own RPM repo?"
- `2` — "is this safe to use in production?"
- `3` — "what does this repo currently publish?"

## Sync model

```
github.com/x-cmd/rpm  (this repo, source of truth)
        │
        │  daily mirror (read-only sync)
        ▼
https://www.x-cmd.com/rpm/  (CDN — what dnf / yum / zypper hit)
```

- This repo's `Packages/*.rpm` and `repodata/` are mirrored
  verbatim.
- No build step on the CDN side — files land as committed.
- The CDN is the canonical source for RPM clients; this
  repo is the canonical source for "what we promised to
  publish."
- Diff this repo against the live mirror to diagnose stale
  content.

## Tools you might need

- `rpmbuild` — to build a `.rpm` from a `.spec`
  (`rpmbuild -ba myapp.spec`). **Not on macOS by default** —
  use the `fedora:latest` container block in CONTRIBUTING.md.
- `createrepo_c` — to regenerate `repodata/`. **Not on macOS
  by default** — same Fedora container.
- `gpg` + `rpm --addsign` — only relevant once signing goes
  live.
- `rpm` — to inspect packages locally (`rpm -Kv`,
  `rpm -ql`, `rpm -qi`).
- `git`, `find`, `sha256sum` — everywhere.

## When in doubt

- **The maintainer wants to cut a release** → `CONTRIBUTING.md`.
- **The user wants to install x-cmd** → `docs/0-install.md`.
- **Someone asks about signing/security** → `docs/2-security.md`.
- **Someone asks "what's in this repo"** → `docs/3-stats.md`.
- **Someone asks "how do I package my own"** → `docs/1-packaging.md`.

## Related repos

- `x-cmd/release` — upstream x-cmd build pipeline; this is
  where the `.rpm` is produced before being mirrored here.
- `x-cmd/deb` — sibling Debian APT repo →
  `https://www.x-cmd.com/deb/`. Different layout
  (`pool/` + `dists/`), same docs convention.
- `x-cmd/cve` — sibling CVE/CWE data repo; same docs
  convention.
- `x-cmd/x-cmd` — x-cmd shell project.
