# Contributing & maintaining x-cmd/rpm

> Runbook for whoever is cutting the next release, debugging
> a sync issue, or adding GPG signing. Read this end-to-end
> once before your first release; for the next one it should
> be skimmable.

## Repo roles

| Role | Person / bot | Does what |
| --- | --- | --- |
| Maintainer | Li Junhao <l@x-cmd.com> | Reviews releases, holds the GPG key, cuts final tags |
| Release pipeline | `x-cmd/release` upstream | Builds the `.rpm` artifact with pinned commit SHAs |
| This repo's sync | Daily from `main` branch | Mirrors `Packages/` + `repodata/` + `x-cmd.repo` + `docs/` to the CDN |

## Cutting a release

1. **Build the new `.rpm`** in `x-cmd/release` upstream. The
   pipeline emits a single
   `x-cmd_<version>-<release>.<arch>.rpm` (typically `noarch`).

2. **Drop the `.rpm` into `Packages/`**:
   ```sh
   cp x-cmd_<version>-<release>.noarch.rpm Packages/
   ```
   Decide whether to **keep** the previous version's `.rpm`
   (rollback path) or **remove** it. The default is to keep
   both — dnf can downgrade via `dnf downgrade x-cmd` as long
   as the repo still references both.

3. **Regenerate the metadata** in a Fedora container (macOS
   doesn't ship `createrepo_c`):
   ```sh
   docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
     dnf install -y createrepo_c >/dev/null &&
     createrepo_c --general-compress-type=gz --basedir /repo /repo'
   ```
   The script regenerates the whole `repodata/` directory.
   `--general-compress-type=gz` keeps metadata gzip-compressed
   for RHEL 7 `yum` compatibility.

4. **Update `docs/3-stats/` (en.md, cn.md, llms.md, faq.yml)** with the
   new version, size, and checksums.

5. **Sanity-check the diff**:
   ```sh
   git diff --stat
   git diff x-cmd.repo
   ```
   Confirm:
   - One new `.rpm` in `Packages/`.
   - Updated `repomd.xml` and three `<sha>-*.xml.gz` files
     in `repodata/`.
   - `docs/3-stats/` reflects the new version.

6. **Commit + push**:
   ```sh
   git add Packages/x-cmd_<version>-<release>.noarch.rpm \
          repodata \
          docs/3-stats
   git commit -m "release: x-cmd <version>"
   git push
   ```

7. **Wait for the daily sync** (or trigger it manually if the
   pipeline supports it). Verify at
   `https://www.x-cmd.com/rpm/` that the new file is present
   in the `Packages/` listing.

8. **Sanity-check from a clean container**:
   ```sh
   docker run --rm -it fedora:latest bash -lc '
     dnf install -y dnf-plugins-core
     dnf config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo
     dnf install -y x-cmd
     x --version'
   ```
   Should report `<version>`.

## Adding a new architecture

For `noarch` packages this section is mostly a no-op. For native
binaries:

1. Build per-arch `.rpm` in CI.
2. Drop into `Packages/` — `createrepo_c` auto-detects and
   indexes by arch.
3. Update `x-cmd.repo` / docs to reflect new arch in the
   supported list.
4. Commit + push + verify from a machine of that arch.

## Adding GPG signing (the `gpgcheck=0` removal)

Pre-requisites: a GPG key generated and stored somewhere safe
(Hardware token preferred — YubiKey, Nitrokey, etc.). The
sub-key used for RPM signing should be a separate signing-only
sub-key with an expiry.

1. **Export the public key** to a stable URL:
   ```sh
   gpg --export --armor <KEY-ID> > RPM-GPG-KEY-x-cmd
   ```
   Host at `https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd` and
   announce on the x-cmd blog + a GitHub release.

2. **Sign every `.rpm`** in CI:
   ```sh
   rpm --addsign Packages/*.rpm
   ```
   (uses the default RPM signing key from `~/.rpmmacros`).

3. **Sign `repomd.xml`** for repo-level integrity:
   ```sh
   gpg --default-key <KEY-ID> --armor \
       --detach-sign --output repodata/repomd.xml.asc \
       repodata/repomd.xml
   ```
   Note: signing `repomd.xml` is the yum-style; modern
   createrepo_c can also embed the signature inline (since
   v0.17+). Check `createrepo_c --help` for current options.

4. **Update `x-cmd.repo`** to flip both flags on:
   ```ini
   [x-cmd]
   name=x-cmd - One-line description
   baseurl=https://www.x-cmd.com/rpm/
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```

5. **Document the key install**:
   ```sh
   rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```

6. **Update `docs/2-security/` (all four files)**
   — flip all rows to ✅, remove the `gpgcheck=0` rationale.

7. **Announce** on the x-cmd blog and a GitHub release.

## Incident playbook

### "Users say `dnf update` fails with checksum error"

- Check that `Packages/*.rpm` and `repodata/*` are byte-aligned
  with what's on the CDN. Most "checksum mismatch" reports
  come from a CDN serving stale metadata against a freshly
  re-uploaded `.rpm`. Force a CDN purge, or wait for the next
  sync cycle.

### "GitHub repo push triggered a sync, but CDN is stale"

- The daily sync is event-driven on push; the cron is a
  backstop. Wait one cron tick, or trigger the sync workflow
  manually.

### "I uploaded a `.rpm` and forgot to regenerate `repodata/`"

- dnf won't see it. Run step 3 of "Cutting a release" again
  and push. The `.rpm` itself doesn't move, so existing
  mirrors don't re-download it.

### "I need to yank a release"

- Move the offending `.rpm` out of `Packages/` (or rename it
  to e.g. `.disabled`), regenerate `repodata/`, push. `dnf
  downgrade` will take users off the bad version. Choose this
  destructive option only for security emergencies.

### "I broke `repomd.xml`"

- Run `createrepo_c` again. The script regenerates the whole
  file. Diff before pushing.

### "Mirror operator reports missing files"

- Confirm the file exists in `Packages/` in this repo (the
  source of truth). If yes, the mirror is out of sync — point
  them at the daily-sync trigger. If no, someone accidentally
  deleted a tracked file; `git restore` it and re-run
  `createrepo_c`.

## Release cadence

- **Event-driven**: a release ships when upstream
  `x-cmd/release` cuts one.
- **Daily sync**: the CDN mirror pulls this repo once every
  24 hours.
- **No fixed CI rebuild** on a schedule — there's nothing to
  rebuild unless a release lands or `repodata/` needs
  regeneration.

## Branch / tag policy

- All releases land on `main`. No long-lived branches.
- A `git tag x-cmd/<version>` may be added for traceability,
  but the canonical "what version is out" is the `Version:` /
  `Release:` fields in `repodata/primary.xml.gz`.
