---
x-title: Build your own .rpm and host it in a yum/dnf repo
x-desc: How to assemble a noarch RPM package from a shell project, regenerate the repodata with createrepo_c, and host the result as a yum/dnf/zypper repository in the canonical Packages/ + repodata/ layout.
x-sidebar: Packaging
x-keywords: rpm, rpmbuild, createrepo_c, noarch, Fedora, RHEL, SUSE, yum repository, spec file, dnf config-manager, repodata, repomd.xml
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Build your own .rpm and host it in a yum/dnf repo'
      inLanguage: 'en'
      about: 'Building an RPM package and hosting a yum/dnf repository'
---

# Build your own .rpm and host it in a yum/dnf repo

> A minimal end-to-end recipe: turn a directory of files into
> a `.rpm` (or `noarch` RPM), regenerate the repo metadata with
> `createrepo_c`, and serve the result from any static CDN.
> Same playbook this repo uses.

If you want to ship shell scripts, completions, and a manpage
to Fedora / RHEL / openSUSE users without going through the
official archive, this is the shortest path.

## What an RPM actually is

An RPM file is a **cpio archive inside a cpio header**
(metainfo), which is itself wrapped in a **lead + signature +
headers + payload** structure:

```
x-cmd_0.9.9_all.rpm =
  lead                       # magic + arch
  signature                  # (optional) RPM signing
  header                     # Name, Version, Requires, ...
  payload (cpio)             # the actual files (noarch → /opt/x-cmd/...)
  payload-size trailer
```

For `noarch` packages, the payload contains shell / scripts
and metadata — not native binaries — so one RPM serves every
architecture.

---

## Why noarch — and why this RPM is just the source tree

x-cmd ships as a `noarch` RPM for two compounding reasons that,
together, make the package *literally* a copy of the source code
wrapped in a cpio payload.

**1. The runtime is the shell itself.** x-cmd is a pure POSIX
shell library, written at the source level to run on `bash`,
`dash`, `ash`, and `zsh` without rewriting. There is no C, no Go,
no Python, no bytecode — just `.sh` files, a few `.1` manpages,
and the assets they read. Whatever shell the target system
already has is the only interpreter the package needs.

**2. There is no build step.** The "build" of an x-cmd RPM is
just `install -m 0644` of the source tree into `%{buildroot}`.
No `make`, no `configure`, no transpiler, no cross-compilation
matrix. The same `.spec`'s `%install` section produces the
correct payload for `x86_64`, `aarch64`, `ppc64le`, `s390x`,
and every other RPM-supported architecture.

The result: the payload of `x-cmd_<ver>_all.rpm` is the source
tree itself, byte-for-byte. A user who `rpm2cpio`s the package
and untars the result sees the same `.sh` files a contributor
sees in the upstream repo. One artifact, every architecture,
no hidden binary.

## 1. Lay out the source tree

RPMs build from a `BUILDROOT` — a flat directory tree
mirroring what should land on the target system:

```
rpmbuild/BUILDROOT/
└── opt/x-cmd/
    └── pkg/
        └── myapp-1.2.3/
            ├── bin/myapp
            ├── share/man/man1/myapp.1
            └── share/myapp/...
```

For noarch packages you can use any path; `/usr/local/x-cmd/pkg/`
or `/opt/x-cmd/pkg/` both work.

## 2. Write the `.spec` file

The single most important file. Minimum viable version:

```spec
Name:           myapp
Version:        1.2.3
Release:        1%{?dist}
Summary:        One-line short description
License:        Apache-2.0
URL:            https://example.com/myapp
Source0:        https://example.com/myapp-%{version}.tar.gz

BuildArch:      noarch
Requires:       bash

%description
One-line short description.
Longer description (any number of lines).

%install
mkdir -p %{buildroot}/opt/x-cmd/pkg/myapp-%{version}/bin
mkdir -p %{buildroot}/opt/x-cmd/pkg/myapp-%{version}/share/man/man1
install -m 0755 myapp %{buildroot}/opt/x-cmd/pkg/myapp-%{version}/bin/
install -m 0644 myapp.1 %{buildroot}/opt/x-cmd/pkg/myapp-%{version}/share/man/man1/

%files
/opt/x-cmd/pkg/myapp-%{version}/

%changelog
* Mon Jan 01 2026 Maintainer <you@example.com> - 1.2.3-1
- Initial package.
```

Add fields as needed: `BuildRequires`, `Recommends`,
`Suggests`, `Requires(post)` etc. Run `man rpm.spec` for the
full schema.

## 3. Build the `.rpm`

```sh
# Use rpmbuild's directory layout
rpmbuild -ba myapp.spec \
    --define "_topdir $PWD/rpmbuild" \
    --define "_sourcedir $PWD" \
    --define "_rpmdir $PWD/RPMS"

# Resulting RPM:
ls RPMS/noarch/myapp-1.2.3-1.noarch.rpm
```

`rpmbuild` needs:
- `rpm-build` package (provides `rpmbuild`)
- A `_topdir` to write `BUILD/`, `BUILDROOT/`, `RPMS/`,
  `SOURCES/`, `SPECS/`, `SRPMS/`

**macOS users** — there is no `rpmbuild` on macOS by default.
Run it inside a Fedora container:

```sh
docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
  dnf install -y rpm-build >/dev/null
  cd /repo
  rpmbuild -ba myapp.spec \
      --define "_topdir $PWD/rpmbuild" \
      --define "_sourcedir $PWD" \
      --define "_rpmdir $PWD/RPMS"
'
```

## 4. Drop it into `Packages/`

The yum/dnf canonical layout is flat:

```
repo/
├── Packages/
│   └── x-cmd_0.9.9_all.rpm         # the hosted RPM
└── repodata/
    └── (createrepo_c output)
```

Drop the `.rpm` straight into `Packages/` — no `pool/`,
no per-source subdirectory (that's a Debian convention).

## 5. Regenerate the repo metadata

`createrepo_c` walks `Packages/` and produces the
metadata that `dnf` / `yum` / `zypper` actually read:

```sh
# Inside a Fedora container (createrepo_c isn't on macOS)
docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
  dnf install -y createrepo_c >/dev/null
  cd /repo
  createrepo_c --general-compress-type=gz --basedir /repo /repo
'

# Output:
repodata/
├── repomd.xml
├── <sha>-primary.xml.gz     # package metadata
├── <sha>-filelists.xml.gz   # file lists
└── <sha>-other.xml.gz       # changelogs, deps
```

**Important**: `--general-compress-type=gz` keeps metadata
gzip-compressed. The classic `yum` on RHEL 7 cannot read zstd
metadata, so this is the safest choice.

## 6. Serve it

`Packages/` + `repodata/` are pure static files. Any HTTP
server will do — `python3 -m http.server`, GitHub Pages,
nginx, a CDN, etc. Users add it via `x-cmd.repo`:

```ini
# /etc/yum.repos.d/x-cmd.repo
[x-cmd]
name=x-cmd - One-line description
baseurl=https://example.com/rpm/
enabled=1
gpgcheck=0
```

Or via `dnf config-manager --add-repo`.

## 7. Add a `.repo` snippet

A ready-to-drop `.repo` in the repo root is friendly to humans
landing on the page:

```ini
[x-cmd]
name=x-cmd - Bootstrap 1000+ command line tools in seconds
baseurl=https://example.com/rpm/
enabled=1
# Metadata and packages are currently unsigned; enable gpgcheck /
# repo_gpgcheck once a GPG key is published.
gpgcheck=0
repo_gpgcheck=0
```

This is the exact pattern this repo uses — see
[`x-cmd.repo`](../../x-cmd.repo).

## When NOT to do this

- **You want the package in Fedora / RHEL / openSUSE proper.**
  Submit to `bugzilla.redhat.com` or upstream maintainers;
  the self-hosted route is for projects that don't want that
  overhead.
- **You have native code for many architectures.** Use
  Fedora's Koji / openSUSE's OBS instead — `rpmbuild` for
  multi-arch still works but you'd want the build matrix,
  not the manual recipe above.
- **You need reproducibility / SBOMs.** Add `rpmbuild`
  in a CI matrix with pinned sources; the recipe here gives
  you a binary, not a reproducible source-to-binary path.

## Next steps

- Want to add GPG signing to the resulting repo? See
  [Security](../2-security/en.md).
- Want to publish a one-liner mirror alongside the repo?
  See [Stats](../3-stats/en.md) for the canonical layout this
  repo exposes.

Source: this repo's [`README.md`](../../README.md) and the
upstream Fedora packaging guide.
