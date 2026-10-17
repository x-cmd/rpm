---
x-title: Install x-cmd via dnf / yum / zypper
x-desc: Add the official x-cmd RPM repository on Fedora / RHEL / openSUSE, install the x-cmd package, and verify with one command.
x-sidebar: Install
x-keywords: x-cmd, dnf, yum, zypper, Fedora, RHEL, CentOS, Rocky, Alma, openSUSE, install, .repo, noarch
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Install x-cmd via dnf / yum / zypper'
      inLanguage: 'en'
      about: 'RPM installation of x-cmd on Fedora / RHEL / openSUSE'
---

# Install x-cmd via dnf / yum / zypper

> The official x-cmd RPM repository, served at
> <https://www.x-cmd.com/rpm/>. Works on Fedora, RHEL, CentOS,
> Rocky, Alma, openSUSE — `dnf`, `yum`, or `zypper` consume the
> same repo via the standard `Packages/` + `repodata/` layout.

The package is `noarch` (architecture-independent), so one
artifact serves every RPM-supported architecture.

## Quick install

### Fedora / RHEL / CentOS / Rocky / Alma (dnf)

```sh
# 1. Enable this repository (downloads x-cmd.repo into /etc/yum.repos.d/)
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo

# 2. Install x-cmd
sudo dnf install -y x-cmd
```

Verify:

```sh
x --version
```

You should see `0.9.9` (or newer).

### Older RHEL / CentOS 7 (yum)

```sh
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo
sudo yum install -y x-cmd
```

### openSUSE (zypper)

```sh
sudo zypper ar -f https://www.x-cmd.com/rpm/ x-cmd
sudo zypper refresh
sudo zypper in x-cmd
```

### Manual repo drop-in (no plugins required)

```sh
# Any dnf / yum / zypper distro
sudo curl -fsSL https://www.x-cmd.com/rpm/x-cmd.repo \
    -o /etc/yum.repos.d/x-cmd.repo
sudo dnf install -y x-cmd
```

`x-cmd.repo` is the canonical repo file that this repo
publishes alongside the packages. It sets `baseurl`,
`enabled=1`, and (currently) `gpgcheck=0` / `repo_gpgcheck=0`.

## What gets installed

| Item | Value |
| --- | --- |
| Package | `x-cmd` |
| Version | `0.9.9` (or newer) |
| Architecture | `noarch` (architecture-independent shell code) |
| Maintainer | Li Junhao <l@x-cmd.com> |
| Group | (unset) |
| Build host | (see RPM header) |
| Source RPM | (none — pre-built binary release) |
| License | Apache-2.0 |

The package is `noarch`, so one `.rpm` serves every RPM
client. dnf will install the same artifact on x86_64,
aarch64, etc.

## Why gpgcheck is off (today)

The repository is currently **unsigned** — `x-cmd.repo`
declares `gpgcheck=0` and `repo_gpgcheck=0`. There is no
release GPG key published yet. This is a known gap; see
[Security](../2-security/en.md) for the plan to enable signed
metadata once the key is announced.

`gpgcheck=0` is opt-in acceptance of unsigned packages. **Drop
it once a key is published.**

## Verify the package manually

```sh
rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
# Today: just shows digest + payload verification, no signature.
# After signing: also shows "V3 RSA/SHA256 Signature, key ID ..."

# Or download + verify digest
curl -fsSLO https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
rpm -Kv x-cmd_0.9.9_all.rpm
```

For maximum rigor (after signing is live):

```sh
rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
sudo dnf config-manager --save --setopt=x-cmd.gpgcheck=1
sudo dnf config-manager --save --setopt=x-cmd.repo_gpgcheck=1
sudo dnf update x-cmd
```

## Next steps

- Want to package your own `.rpm` for distribution via dnf /
  yum / zypper? See [Packaging](../1-packaging/en.md).
- Want to understand the signing plan and what changes once
  we publish a GPG key? See [Security](../2-security/en.md).
- Curious about what the repo currently publishes (versions,
  checksums, architectures)? See [Stats](../3-stats/en.md).

Source: [`README.md`](../../README.md),
[`x-cmd.repo`](../../x-cmd.repo),
and `Packages/x-cmd_0.9.9_all.rpm` as of the commit pinned in
`git log`.
