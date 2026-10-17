---
name: 2-security
description: RPM repository security model for x-cmd/rpm. Status of GPG signing, what `gpgcheck=0` and `repo_gpgcheck=0` each opt into, manual digest verification recipe, and the GPG signing roadmap.
type: explanation
---

# Core Content

security_status:

  tls_at_cdn: true
  per_package_sha256_in_primary: true
  repomd_with_checksums: true
  repomd_xml_asc_detached: false
  repomd_clear_signed: false
  rpm_header_per_package_signature: false
  gpg_key_published: false
  key_rotation_policy: false
  gpgcheck: 0                     # disabled today
  repo_gpgcheck: 0                # disabled today
  trusted_required: true           # operator trust on reputation alone

what_the_flags_disable:

  - id: gpgcheck-skip
    description: |
      `gpgcheck=0` skips the per-package GPG/PGP signature that
      rpm embeds inside each `.rpm`'s header (over lead +
      headers + payload).
    what_it_does:
      - Without: dnf refuses to install unsigned `.rpm`
      - With: dnf installs without checking the signature

  - id: repo-gpgcheck-skip
    description: |
      `repo_gpgcheck=0` skips the signature on `repomd.xml`
      (the suite-level manifest pointing at every metadata file).
    what_it_does:
      - Without: dnf refuses to use a repo whose repomd.xml is
        unsigned or has an invalid signature
      - With: dnf trusts repomd.xml from this baseurl

verification_recipes:

  - id: rpm-verify
    description: |
      Inspect a package's signature + digest in one command.
      Same call works today (digest only) and after signing
      (additionally shows key ID).
    command: rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm

  - id: manual-digest
    description: |
      Cross-check the package's SHA-256 against the digest
      listed in `repodata/<sha>-primary.xml.gz`. Compare the
      expected hash against a second channel (GitHub release
      page) for full confidence.
    command: |
      curl -fsSLO https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
      curl -fsSL https://www.x-cmd.com/rpm/repodata/repomd.xml
      # Follow data references, fetch each metadata file, then
      # compare the package's digest to the value in
      # primary.xml.gz.

signing_roadmap:

  - step: 1
    title: Publish GPG key
    description: |
      Publish the release GPG key at
      `https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd`.
      Long-term key + signing-only sub-key with expiry.

  - step: 2
    title: Sign packages + repomd.xml
    description: |
      CI runs `rpm --addsign` on every `.rpm` and
      `gpg --detach-sign --output repomd.xml.asc repomd.xml`
      so the suite-level manifest is also signed.

  - step: 3
    title: Update x-cmd.repo
    description: |
      Flip `gpgcheck=0` → `gpgcheck=1` and `repo_gpgcheck=0`
      → `repo_gpgcheck=1`. Add `gpgkey=https://...` so
      dnf / yum / zypper can fetch the public key.

  - step: 4
    title: Document the key install
    description: |
      One command for users: `rpm --import https://...`.
      Documented in install + security pages.

  - step: 5
    title: Enforce
    description: |
      With gpgcheck + repo_gpgcheck both 1, dnf refuses
      unsigned / invalid-signature sources — same protection
      as any signed RPM repo.

## Key Information

supply_chain_risks_not_mitigated_today:

  - id: account-takeover
    description: |
      A compromised GitHub token with push access to
      `x-cmd/rpm` could publish a malicious `.rpm`.
    mitigation_after_signing: true

  - id: mirror-compromise
    description: |
      `www.x-cmd.com/rpm/` is a static mirror; if swapped
      before TLS, you'd get attacker's bytes.
    mitigation_after_signing: true

  - id: replay-downgrade
    description: |
      TLS-MITM attacker serves an older, vulnerable `.rpm`.
    mitigation_after_signing: true

practical_safety_net_today:

  - repo_on_github_for_audit
  - verbatim_mirror_via_readonly_pipeline
  - deterministic_build_from_same_source_as_public_release_artifacts
  - cross_channel_sha256_verification_via_github_release

## Use Cases

use_cases:

  - Dev-machine install: fine as-is, `gpgcheck=0` is documented
    and the operator trust model is acceptable
  - Production fleet with audit: wait for InRelease-equivalent
    GPG signing OR pin the version AND verify SHA-256 from a
    second channel on every deploy
  - CI ephemeral container: skip the dnf repo, fetch + verify
    the `.rpm` directly

## Related Resources

related:

  repo: https://github.com/x-cmd/rpm
  upstream: https://github.com/x-cmd/release
  rpm_signing_docs: "https://docs.fedoraproject.org/en-US/package-guidelines/PackageSigningGuidelines/"
  rpm_man: "man rpm"

## Summary

This RPM repo is currently unsigned. The integrity layer you
can rely on today is TLS at the CDN, per-package SHA-256 in
`primary.xml`, and suite-level `repomd.xml` with checksums of
every metadata file. `gpgcheck=0` and `repo_gpgcheck=0`
together disable APT's signature check for this specific source;
they do not disable TLS. The supply-chain risks not mitigated
today are account takeover, mirror compromise, and
replay/downgrade — all caught once GPG signing goes live.
Until then, pin the version and verify SHA-256 from a second
channel (e.g., GitHub release page).
