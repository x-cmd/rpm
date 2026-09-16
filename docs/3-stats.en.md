---
x-title: Repository stats and what the repo publishes
x-desc: The exact set of RPM files and repodata this RPM repo publishes at any given moment — sizes, checksums, and the suite-level repomd.xml metadata.
x-sidebar: Stats
x-keywords: RPM, stats, package version, noarch, checksums, repomd.xml, primary.xml, filelists.xml, other.xml, sha256
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Repository stats and what the repo publishes'
      inLanguage: 'en'
      about: 'Repository metadata, sizes, and checksums'
---

# Repository stats and what the repo publishes

> The canonical, exact-pinned snapshot of what's in this repo
> right now. Sizes and checksums are read straight from the
> committed files; if they change, this page changes with them.

## Suite overview

| Item | Value |
| --- | --- |
| Origin | x-cmd |
| Label | x-cmd |
| Suite | (single, flat) |
| Architectures | `noarch` (one RPM serves all) |
| Source | x-cmd / x-cmd |
| Metadata compression | gzip (`--general-compress-type=gz`) |
| Metadata generator | `createrepo_c` |

The repository has a **flat layout** — there is no per-suite
subdirectory like Debian's `dists/stable/`. `Packages/` and
`repodata/` live at the root.

## Package snapshot

| Field | Value |
| --- | --- |
| Package | `x-cmd` |
| Version | `0.9.9` |
| Release | `1` |
| Architecture | `noarch` |
| Maintainer | Li Junhao <l@x-cmd.com> |
| Group | (unset) |
| License | Apache-2.0 |
| URL | https://www.x-cmd.com |
| Filename | `Packages/x-cmd_0.9.9_all.rpm` |
| Size | 4,363,684 bytes (~4.16 MB) |
| Source RPM | (none — pre-built binary release) |

The package is `noarch`, so one RPM serves every RPM-supported
architecture (x86_64, aarch64, ppc64le, s390x, etc.).

## Payload (`Packages/`)

| File | Size |
| --- | --- |
| `Packages/x-cmd_0.9.9_all.rpm` | 4,363,684 |

One binary RPM, mirrored to all architectures.

## Metadata (`repodata/`)

Every metadata file's SHA-256 is referenced from `repomd.xml`
(the suite-level manifest). Sizes are bytes.

| File | Size |
| --- | --- |
| `repodata/repomd.xml` | 1,546 |
| `repodata/b7faf596ebb1c2510ba2cf8082b84e0ec1dc4b1a77fa2d15047a72a3b2c29c50-primary.xml.gz` | 717 |
| `repodata/497dcdc88383d492c7f7733e5b30cd810b77d212e55a85fdec7b9a7be4a275d2-filelists.xml.gz` | 261 |
| `repodata/75c04b581370f1d3fb262dec445aafa3073db147fa3113759592ed91274ca213-other.xml.gz` | 320 |

What each file contains:
- `repomd.xml` — entry point; references every other metadata
  file with its SHA-256 and timestamp
- `primary.xml.gz` — package metadata (Name, Version,
  Requires, size, location, etc.)
- `filelists.xml.gz` — per-package file lists (what `/opt/x-cmd/...`
  files the RPM installs)
- `other.xml.gz` — changelogs, dependency resolutions, etc.

## Repository metadata (`repomd.xml` excerpt)

```xml
<repomd>
  <revision>...</revision>
  <data type="primary">
    <checksum type="sha256">b7faf596ebb1c2510ba2cf8082b84e0ec1dc4b1a77fa2d15047a72a3b2c29c50</checksum>
    <open-checksum type="sha256">...</open-checksum>
    <location href="repodata/b7faf596...-primary.xml.gz"/>
    <timestamp>...</timestamp>
    <size>717</size>
    <open-size>...</open-size>
  </data>
  <data type="filelists">...</data>
  <data type="other">...</data>
</repomd>
```

## Popularity (proxy)

This repo itself doesn't track download counts (it's a static
mirror with no analytics layer). For a proxy of adoption, see
the upstream `x-cmd` project:

| Channel | URL |
| --- | --- |
| GitHub stars | <https://github.com/x-cmd/x-cmd> |
| GitHub releases | <https://github.com/x-cmd/release> |
| Companion APT repo | <https://github.com/x-cmd/deb> |
| Main site | <https://www.x-cmd.com> |

## Update cadence

- The repo is published when a new x-cmd release ships
  upstream.
- The CDN mirror refreshes daily from this GitHub repo.
- No CI rebuild on a fixed schedule — there's nothing to
  rebuild unless a release lands.

## How to fetch the raw values

```sh
# Suite metadata
curl -fsSL https://www.x-cmd.com/rpm/repodata/repomd.xml

# Package metadata (gzipped)
curl -fsSL https://www.x-cmd.com/rpm/repodata/primary.xml.gz | gunzip

# The package itself
curl -fsSLO https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
```

All three are static files served from the same CDN as the
landing page.

## Difference from the Debian sibling

| | Debian (`x-cmd/deb`) | RPM (`x-cmd/rpm`) |
| --- | --- | --- |
| Layout | `pool/` + `dists/stable/{Release,main/binary-*/Packages*}` | flat `Packages/` + `repodata/` |
| Components / Suites | multi-Component Suite | single flat repo |
| Per-arch indices | yes (5 indices, identical) | no (`noarch` covers all) |
| Metadata generator | `dpkg-scanpackages` + `apt-ftparchive` | `createrepo_c` |
| Signing | `[trusted=yes]` today | `gpgcheck=0` + `repo_gpgcheck=0` today |
| Client | `apt` | `dnf` / `yum` / `zypper` |

## Next steps

- Want to know how those files get generated? See
  [Packaging](./1-packaging.en.md).
- Want to know what the security model promises today? See
  [Security](./2-security.en.md).

Source: [`repodata/repomd.xml`](../../repodata/repomd.xml),
[`repodata/primary.xml.gz`](../../repodata/primary.xml.gz),
and [`Packages/x-cmd_0.9.9_all.rpm`](../../Packages/x-cmd_0.9.9_all.rpm)
as of the commit pinned in `git log`.
