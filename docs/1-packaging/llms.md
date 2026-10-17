---
name: 1-packaging
description: End-to-end recipe for building a noarch RPM package and hosting a yum/dnf/zypper repository in the canonical Packages/ + repodata/ layout. Includes rpmbuild, createrepo_c, and Fedora container command.
type: tutorial
---

# Core Content

core_steps:

  - name: write-spec
    description: |
      Write a .spec file with Name / Version / Release / Summary /
      License / URL / BuildArch=noarch / Requires / %install / %files /
      %changelog. Run `man rpm.spec` for the full schema.

  - name: build-rpm
    description: |
      Build with rpmbuild. Need `rpm-build` package and a _topdir.
    command: |
      rpmbuild -ba myapp.spec \
          --define "_topdir $PWD/rpmbuild" \
          --define "_sourcedir $PWD" \
          --define "_rpmdir $PWD/RPMS"

  - name: place-in-packages
    description: |
      Move the .rpm straight into `Packages/`. No `pool/`, no
      per-source subdirectory (yum/dnf layout is flat).

  - name: regenerate-repodata
    description: |
      Inside a Fedora container, run `createrepo_c` against
      the repo root. `--general-compress-type=gz` keeps
      metadata gzip-compressed for RHEL 7 `yum` compatibility.
    command: |
      docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
        dnf install -y createrepo_c >/dev/null
        cd /repo
        createrepo_c --general-compress-type=gz --basedir /repo /repo
      '

  - name: serve-static
    description: |
      `Packages/` + `repodata/` are pure static files. Serve
      from any HTTP server (nginx, GitHub Pages, CDN).

  - name: add-repo-snippet
    description: |
      Add a `x-cmd.repo` snippet at the repo root so humans
      can copy-paste it into `/etc/yum.repos.d/`.

## Key Information

rpm_format:

  type: cpio-in-cpio wrapper
  contents:
    - lead               # magic + arch
    - signature          # (optional) RPM signing
    - header             # Name, Version, Requires, ...
    - payload (cpio)     # actual files
    - payload-size trailer

required_tools:

  - rpm-build            # provides rpmbuild
  - createrepo_c         # generates repodata
  note: |
    None of these ship on macOS by default. Run them inside a
    `fedora:latest` container, mounted with the repo as `/repo`.

repo_layout:

  packages_dir: Packages/
  repodata_dir: repodata/
  metadata_format: repomd.xml + gzip-compressed sqlite
  metadata_compression: gz (not zstd, for RHEL 7 yum compat)

## Use Cases

use_cases:

  - Ship a shell-only CLI to Fedora / RHEL / openSUSE users
    without going through the official archive
  - Internal corporate package distribution (Linux fleet)
  - Pre-release / beta channels for a project
  - Hosting multiple versions side-by-side for rollback

## Anti-patterns

avoid_when:

  - You want the package IN Fedora / RHEL / openSUSE official —
    submit upstream or find a sponsor.
  - You ship native code for many architectures — use Koji /
    OBS instead.
  - You need reproducibility / SBOMs — add rpmbuild in a CI
    matrix with pinned sources.

## Related Resources

related:

  man_rpm_spec: "man rpm.spec"
  man_createrepo_c: "man createrepo_c"
  fedora_packaging_guide: "https://docs.fedoraproject.org/en-US/package-guidelines/"
  fedora_koji: "https://koji.fedoraproject.org/"
  opensuse_obs: "https://openbuildservice.org/"
  reference_repo: "https://github.com/x-cmd/rpm"

## Summary

End-to-end recipe: build a noarch `.rpm` with `rpmbuild -ba`,
drop it in `Packages/`, run `createrepo_c --general-compress-type=gz`
to regenerate `repodata/`, serve the tree from any static HTTP
server. The same playbook this repo (`x-cmd/rpm`) uses to
publish itself. Metadata is gzip-compressed (not zstd) so
classic `yum` on RHEL 7 can read it.
