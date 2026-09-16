---
x-title: Security, signing, and the gpgcheck=0 question
x-desc: Why this RPM repo is unsigned today, what `gpgcheck=0` and `repo_gpgcheck=0` actually opt into, how to verify package digests manually, and the GPG signing plan to enable both checks.
x-sidebar: Security
x-keywords: RPM, signing, GPG, gpgcheck, repo_gpgcheck, sha256, rpm -Kv, RPM-GPG-KEY, supply chain, dnf config-manager
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: 'Security, signing, and the gpgcheck=0 question'
      inLanguage: 'en'
      about: 'RPM repository security, signing, and verification'
---

# Security, signing, and the `gpgcheck=0` question

> The honest, complete picture of what this repo promises today
> — and what it does not. If you're a security reviewer or a
> cautious sysadmin, this page is for you.

This repo is currently **unsigned**. `x-cmd.repo` declares
`gpgcheck=0` and `repo_gpgcheck=0` — opt-in acceptance that
nothing is signed. We have a plan to publish a release GPG key
and turn both checks on; the section at the bottom tracks it.

## What you get today

| Property | Status |
| --- | --- |
| TLS at the CDN | ✅ (https://www.x-cmd.com/rpm/) |
| Per-package SHA-256 digest in `primary.xml` | ✅ |
| Suite-level `repomd.xml` with checksums of every metadata file | ✅ |
| `repomd.xml.asc` detached signature | ❌ |
| `repodata/repomd.xml` clear-signed inline | ❌ |
| RPM header signature (per-package `.rpm.sig`) | ❌ |
| GPG key published at `RPM-GPG-KEY-x-cmd` | ❌ |
| Time-based key expiry + rotation policy | ❌ (no key yet) |

The first three rows are the integrity layer you can already
rely on. The bottom four rows are what `gpgcheck=0` and
`repo_gpgcheck=0` are silently disabling.

## What `gpgcheck=0` and `repo_gpgcheck=0` actually opt into

The two flags are separate:

| Flag | What it skips |
| --- | --- |
| **`gpgcheck=0`** | The per-package signature inside each `.rpm`'s header (rpm's built-in GPG/PGP sig on the lead+headers+payload). |
| **`repo_gpgcheck=0`** | The signature on `repomd.xml` (the suite-level manifest). |

Both together protect against two things:

1. **Network MITM**: a TLS-stripping attacker rewriting the
   `.rpm` you fetch.
2. **Repository compromise**: the operator (or someone who
   takes over their account) pushing a `.rpm` you didn't
   intend to install.

`gpgcheck=0` + `repo_gpgcheck=0` disables **both** for this
specific repo. The `https://` URL still gives you TLS, so the
MITM vector is covered. The operator-trust vector is the one
you are now trusting on operator reputation alone.

For x-cmd specifically: the repo is on GitHub (you can audit
every commit), mirrored to `www.x-cmd.com` via the standard
read-only pipeline, and the `.rpm` is built deterministically
from the same source as the public x-cmd release artifacts.
You can verify the SHA-256 against the GitHub release's
checksum file. That's the practical safety net today.

## How to verify without adding the repo

```sh
URL="https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm"
EXPECTED="<sha256-from-packages>"

curl -fsSLO "$URL"
echo "$EXPECTED  x-cmd_0.9.9_all.rpm" | sha256sum -c -
```

`OK` means the bytes you fetched match what the repo claimed
in `repodata/<sha>-primary.xml.gz`. Compare the expected hash
against a second channel (the GitHub release page, a Twitter
announcement, a friend) for full confidence.

For maximum rigor, fetch `repodata/repomd.xml` directly and
follow the `data` references to each metadata file's SHA-256;
then verify the package file's digest against
`repodata/<sha>-primary.xml.gz`.

Or just use `rpm`:

```sh
rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
# Today: just digest + payload verification, no signature.
# After signing: also shows "V3 RSA/SHA256 Signature, key ID ..."
```

## The signing plan

When the operator publishes a GPG key, this is what changes:

1. The key is published at a stable URL
   (`https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd`).
2. The key is **imported into the user's RPM database**
   (one command):
   ```sh
   rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```
3. CI signs every `.rpm` with `rpm --addsign` and signs
   `repomd.xml` with `gpg --detach-sign` (produces
   `repomd.xml.asc`).
4. The `x-cmd.repo` snippet flips both flags on:
   ```ini
   [x-cmd]
   name=x-cmd - One-line description
   baseurl=https://www.x-cmd.com/rpm/
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```
5. dnf / yum / zypper refuse to install from this source if
   signatures are missing, invalid, or made by an unexpected
   key — the same protection as any other signed RPM repo.

Until that work is published, **do not** blindly trust this
repo on production fleets you can't easily audit. Use it on
dev machines and pin the version.

## Supply-chain risks we **don't** mitigate today

- **Account takeover of `x-cmd/rpm`**: a compromised GitHub
  token could publish a malicious `.rpm`. Mitigation: GPG
  signing makes any such push require the operator's signing
  key, not just any commit access.
- **Mirror compromise**: `www.x-cmd.com/rpm/` is a static
  mirror; if it's swapped before the TLS layer, you'd get the
  attacker's bytes. Mitigation: same GPG signature catches
  it.
- **Replay / downgrade**: an attacker with TLS-MITM could
  serve you an older, vulnerable `.rpm`. Mitigation: dnf's
  `repomd.xml` date check plus the GPG signature catches this
  once signing is live.

## Source provenance

The `.rpm` is built from the upstream x-cmd release pipeline
([`x-cmd/release`](https://github.com/x-cmd/release)) and
mirrored verbatim into this repo. The build is reproducible
to the extent that the upstream pipeline is — for shell-only
content with pinned commit SHAs that is generally a yes.

## Next steps

- Want to know what's currently being published? See
  [Stats](./3-stats.md).
- Want to audit the package metadata yourself? See
  [Packaging](./1-packaging.md) for the index layout.

Source: this repo's [`README.md`](../../README.md) and the
upstream RPM signing documentation.
