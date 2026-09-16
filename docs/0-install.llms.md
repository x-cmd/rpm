---
name: 0-install
description: Install x-cmd via the official RPM repository on Fedora / RHEL / CentOS / Rocky / Alma / openSUSE. dnf / yum / zypper install paths, gpgcheck rationale, and post-signing verification.
type: how-to
---

# Core Content

core_steps:

  - name: enable-repo-dnf
    description: |
      On Fedora / RHEL / CentOS / Rocky / Alma via dnf. Downloads
      x-cmd.repo into /etc/yum.repos.d/.
    command: |
      sudo dnf install -y dnf-plugins-core
      sudo dnf config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo

  - name: enable-repo-yum
    description: |
      Older RHEL / CentOS 7 via yum-utils.
    command: |
      sudo yum install -y yum-utils
      sudo yum-config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo

  - name: enable-repo-zypper
    description: openSUSE via zypper.
    command: |
      sudo zypper ar -f https://www.x-cmd.com/rpm/ x-cmd
      sudo zypper refresh

  - name: manual-drop-in
    description: |
      Skip plugins entirely — curl the .repo file directly.
      Works on any dnf / yum / zypper distro.
    command: |
      sudo curl -fsSL https://www.x-cmd.com/rpm/x-cmd.repo \
          -o /etc/yum.repos.d/x-cmd.repo

  - name: install-package
    description: Install the x-cmd package after enabling the repo.
    command: sudo dnf install -y x-cmd

  - name: verify
    description: Confirm x-cmd is on PATH and reports the expected version.
    command: x --version
    expected_output: "0.9.9 or newer"

## Key Information

package_metadata:

  name: x-cmd
  version: 0.9.9
  architecture: noarch
  maintainer: Li Junhao <l@x-cmd.com>
  license: Apache-2.0
  source_rpm: null                    # pre-built binary release
  install_path_examples:
    - /usr/bin/x
    - /usr/libexec/x-cmd/
    - (see `rpm -ql x-cmd`)

repository_metadata:

  origin: x-cmd
  label: x-cmd
  baseurl: https://www.x-cmd.com/rpm/
  enabled: 1
  gpgcheck: 0                        # unsigned today
  repo_gpgcheck: 0                   # unsigned today
  metadata_format: repomd.xml + gzip-compressed sqlite
  metadata_generator: createrepo_c

## Use Cases

use_cases:

  - One-line install on Fedora / RHEL / openSUSE
  - Air-gapped install — drop .rpm manually + `rpm -Uvh`
  - Manual repo drop-in for locked-down CI runners
  - Multi-distro fleet — same RPM works on x86_64, aarch64, etc.

## Common Gotchas

gotchas:

  - id: gpgcheck-disabled
    description: |
      Today x-cmd.repo declares `gpgcheck=0` and `repo_gpgcheck=0`
      — packages and metadata are not signed. dnf will not warn,
      but you're trusting the operator. Drop these flags once
      a release GPG key is published.

  - id: noarch-vs-arch
    description: |
      Package is `noarch` so one RPM serves every architecture.
      Don't try to fetch per-arch variants — there is only one
      `.rpm`.

  - id: metadata-compression
    description: |
      Metadata is gzip-compressed (`--general-compress-type=gz`)
      rather than zstd, for maximum client compatibility —
      classic `yum` on RHEL 7 cannot read zstd metadata.

  - id: opensuse-basurl
    description: |
      openSUSE's `zypper ar` takes a baseurl, not a .repo URL.
      Point it directly at the repository root.

## Verification Recipes

verification:

  - id: package-only
    description: Inspect the package signature and digest.
    command: rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm

  - id: after-signing
    description: After GPG signing goes live, enable both checks.
    command: |
      rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
      sudo dnf config-manager --save --setopt=x-cmd.gpgcheck=1
      sudo dnf config-manager --save --setopt=x-cmd.repo_gpgcheck=1
      sudo dnf update x-cmd

## Related Resources

related:

  repo: https://github.com/x-cmd/rpm
  cdn: https://www.x-cmd.com/rpm/
  manual_rpm: https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
  upstream: https://github.com/x-cmd/release

## Summary

One-line RPM install on Fedora / RHEL / openSUSE via the
official x-cmd repo at `https://www.x-cmd.com/rpm/`. The
repo is currently unsigned (`gpgcheck=0`, `repo_gpgcheck=0`),
so drop those flags once a release GPG key is published. The
package is `noarch` so one RPM serves every architecture.
Metadata is gzip-compressed (not zstd) for RHEL 7 `yum`
compatibility.
