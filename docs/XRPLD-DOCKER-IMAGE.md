# xrpld in Docker: Build Your Own Image from the Signed Package

*A companion to [The Rippled Field Guide](RIPPLED-FIELD-GUIDE.md). Written 2026-09-26, from a production validator upgrade (3.3.0 to 3.4.1).*

## Bottom Line

- Since 3.4.0, official xrpld binaries ship only as signed DEB and RPM packages at `packages.xrplf.org`. The official install guide has no Docker section.
- Third-party images lag. `xrpllabsofficial/xrpld` stops at 3.3.0, and waiting for it can run you into an amendment-block deadline.
- Build the image yourself, on the validator box, from the signed DEB. It takes about a minute.
- Verify three things before you trust it: the signing key fingerprint, the .deb SHA-256 from the release announcement, and the commit hash the binary reports.
- **Test the new image with your exact production mounts and user before you swap.** A wrong runtime user doesn't crash the node. It starts xrpld on the package's stock config, with no validator token, and your healthcheck still passes.
- Give the old container time to stop cleanly (`--timeout 300`), and keep the old image for rollback. The rollback expires when the new release's amendment enables.

## Why Build Your Own

| Option | Trust chain | Lag |
|---|---|---|
| Official DEB/RPM | XRPLF signing key, SHA-256 in the announcement | None |
| Third-party image | Whoever built it | Days to never |
| Your own image from the DEB | The same as the DEB, verified by you | None |

As of 2026-09-26, `xrpllabsofficial/xrpld` on Docker Hub still tops out at 3.3.0. `rippleci/xrpld` does have a 3.4.1 tag, but it's a CI account with no description, and neither the install guide nor the release announcement mentions it. I can't tell you how it's built, so I don't run it.

**Example: xrpld 3.4.1.** It shipped as an emergency release on 2026-09-25. Its new amendment, `fixBatchV1_2`, already had supermajority support on release day and was expected to enable on 2026-10-09. Any server not upgraded by then becomes amendment blocked. The signed packages were out the same day, but a day later `xrpllabsofficial/xrpld` still had no 3.4.x tag. With your own build, the deadline depends on nobody but you.

## What You Need

- An x86_64 Linux host with Docker (the package is amd64 only)
- Outbound HTTPS to `packages.xrplf.org` and the Ubuntu archive
- Two files: a `Dockerfile` and an `entrypoint.sh` ([Appendix](#appendix-the-two-files))
- Your current `docker-compose.yml`, and 15 minutes

## Step 1: Verify the Release

Do this before you build anything.

**1a. Read the announcement** at `https://xrpl.org/blog/<year>/xrpld-<version>`. Write down the DEB SHA-256 and the commit hash. Also read every announcement you're skipping over, since config changes hide there.

**1b. Check the package index matches the announcement:**

```bash
curl -fsSL https://packages.xrplf.org/repository/deb-stable/dists/any/main/binary-amd64/Packages \
  | awk '/^Package: xrpld$/{p=1} /^$/{p=0} p' | grep -E '^(Version|SHA256)'
```

The `SHA256` for your version must match the announcement exactly.

**1c. Check the signing key:**

```bash
curl -fsSL https://packages.xrplf.org/xrplf.asc | gpg --show-keys
```

```
pub   rsa4096 2026-08-18 [SC]
      B655416741221F780FBCFBC9AA84D41A11D29FA9
uid                      XRPLF Packages <distribution@xrplf.org>
```

The fingerprint must match what the [install guide](https://github.com/XRPLF/rippled/blob/develop/docs/install.md) and [xrpl.org](https://xrpl.org/docs/infrastructure/installation/install-rippled-on-ubuntu) publish. If it doesn't, stop. The key has rotated before, so confirm any new one from both sources.

**1d. Note your deadline.** Ask your node which amendments have majority:

```bash
docker exec <container> xrpld --conf /etc/xrpld/xrpld.cfg -q feature \
  | jq -r '.result.features[] | select(.enabled==false and .majority!=null) | "\(.name // "?") supported=\(.supported) majority=\(.majority)"'
```

`majority` is in Ripple epoch seconds. Convert it with `date -u -d @$((majority + 946684800))`, then add 14 days. An amendment your version doesn't support (`supported=false`, often `name=?`) blocks you on that date.

## Step 2: Build the Image

Put the two files from the appendix in one directory and build. For another release, swap in its version and its announced SHA-256:

```bash
docker build \
  --build-arg XRPLD_VERSION=3.4.1 \
  --build-arg XRPLD_PKG_VERSION=3.4.1-1 \
  --build-arg XRPLD_DEB_SHA256=<sha256 from the announcement> \
  --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --tag localhost/xrpld:3.4.1 .
```

What the Dockerfile does:

| Step | Why |
|---|---|
| Pins `ubuntu:24.04` by digest | The base can't change under you between builds |
| Fetches the key and fails unless it's exactly one key with the published fingerprint | apt's `signed-by` trusts every key in the file, so the whole file gets checked |
| Adds the repo line from the install guide, character for character | Signature checking stays on. No `trusted=yes` |
| Installs one exact version, after checking the .deb's SHA-256 | A second, independent check on the same bytes |
| Checks `dpkg` and `xrpld --version` report what you asked for | What went in is what you asked for |
| Removes curl and gnupg, keeps ca-certificates | xrpld fetches validator lists over HTTPS |
| Runs as uid 997 by default, entrypoint `/entrypoint.sh` | The same contract as the old `xrpllabsofficial` image |

The `localhost/` prefix matters. That name can never resolve to Docker Hub, so a typo or a `pull` fails loudly instead of fetching someone else's image.

The build log must show both of these lines:

```
signing key fingerprint verified: B655416741221F780FBCFBC9AA84D41A11D29FA9
package SHA-256 matches the announcement: cae8ce3b...6f26
```

Then confirm the commit:

```bash
docker run --rm --network none --entrypoint /usr/bin/xrpld localhost/xrpld:3.4.1 --version
```

`Git commit hash:` must match the announcement.

The Dockerfile pins two values that can change between releases: the signing key fingerprint and the Ubuntu base digest. If the key rotates, the build refuses (good). Confirm the new fingerprint from both official sources before you update it. Bump the base digest on purpose to pick up Ubuntu security patches.

## Step 3: Test It the Way Production Runs It

This is the step that saves you. Check who your container runs as today:

```bash
docker inspect <container> --format 'User={{json .Config.User}}'
docker exec <container> id
```

The old `xrpllabsofficial` image ran as **root** by default. The new image defaults to **uid 997**. If your compose file has no `user:` line, you've been running as root, and the switch changes that silently.

Why it matters: the entrypoint copies `/config/rippled.cfg` to `/etc/xrpld/xrpld.cfg`. As uid 997 that copy fails, and the entrypoint deliberately doesn't stop on a failed copy. xrpld then starts on the package's default config: a stock node with a new identity and no validator token. It syncs, it reports `full`, and a healthcheck that accepts `full` goes green.

I reproduced exactly that:

```
/entrypoint.sh: line 35: /etc/xrpld/xrpld.cfg: Permission denied
entrypoint: could not copy /config/rippled.cfg to /etc/xrpld/xrpld.cfg; xrpld reads /etc/xrpld/xrpld.cfg as it is
```

**Test with your real mounts and security options, no network, and no data volume:**

```bash
docker run --rm --network none --user 0:0 --cap-drop ALL --security-opt no-new-privileges:true \
  -v /path/to/rippled.cfg:/config/rippled.cfg:ro \
  -v /path/to/validators.txt:/config/validators.txt:ro \
  --entrypoint /bin/bash localhost/xrpld:3.4.1 -c \
  '/entrypoint.sh --version; cmp /config/rippled.cfg /etc/xrpld/xrpld.cfg && echo CFG_OK; cmp /config/validators.txt /etc/xrpld/validators.txt && echo VL_OK'
```

You want `copied` twice, `CFG_OK` and `VL_OK`. Use the same `--user` your compose will use.

| Your situation | Do this |
|---|---|
| Old container ran as root (empty `User`) | Add `user: "0:0"` for the swap. Move data ownership to a service account later, as a separate change |
| Running as a non-root uid (or you want to) | `/etc/xrpld/` in the image is root-owned, so the copy fails for any non-root uid. Mount the config read-only at both `/config/rippled.cfg` and `/etc/xrpld/xrpld.cfg` (same for `validators.txt`) so the entrypoint finds identical files and skips the copy. That uid must be able to read both files and write your data directory |

> **Never** run the new image beside the live one with the real config and network access. Two instances signing with one validator key is the one thing you must never do. `--network none` plus `--version` is safe.

## Step 4: Swap

**4a. Pick the moment.** Don't stop the node during an online-delete rotation. The log shows `SHAMapStore:WRN rotating` at the start and `finished rotation` at the end. With `advisory_delete=1`, your own `can_delete` schedule decides when the next one starts.

**4b. Back up** the compose file, the config and `validators.txt` to a dated directory, mode 0700, on the same box. The config holds your `[validator_token]`, so it never leaves the machine. Leave the data volume alone: it carries over, and so does your node identity (`wallet.db`). Keep the old image. Don't prune.

**4c. Edit the compose file:**

```yaml
    image: localhost/xrpld:3.4.1
    pull_policy: never        # local image, never try a registry
    user: "0:0"               # only if the old container ran as root (Step 3)
    stop_grace_period: 5m     # matches the package's systemd unit (TimeoutStopSec=5min)
```

**4d. Recreate only the validator service:**

```bash
docker compose -p <project> -f /path/to/docker-compose.yml up -d --timeout 300 <service>
```

`--timeout 300` gives the old container 5 minutes to close NuDB cleanly. Don't use `down`, which takes the whole project with it.

**4e. Read the start log:**

```bash
docker logs <container> 2>&1 | grep -E 'entrypoint:|Validator identity|Process starting'
```

```
entrypoint: copied /config/rippled.cfg to /etc/xrpld/xrpld.cfg
LedgerConsensus:NFO Validator identity: <your validator public key>
Application:NFO Process starting: xrpld-3.4.1
```

No `Validator identity` line means it's running on the wrong config. Roll back now.

## Step 5: Verify

| Check | Command | Pass |
|---|---|---|
| Version and state | `server_info` | `build_version` is new, `server_state` is `proposing` |
| Identity | `server_info` | `pubkey_validator` and `pubkey_node` unchanged |
| Quorum | `server_info` | `validation_quorum` back to the old value (it can read one higher until the Negative UNL reapplies) |
| Not blocked | `server_info` | no `amendment_blocked` |
| Config in use | `docker exec <c> cmp /config/rippled.cfg /etc/xrpld/xrpld.cfg` | exit 0 |
| New amendment | `feature` | `supported: true` |
| UNL | `validators` | both publisher lists `available: true` |
| Health | `docker inspect` | `healthy` |
| Network sees you | `validations` stream on a public server | your key shows up about once per ledger |

The last check matters. `proposing` only means you're sending validations. Subscribe to `validations` on a public server (`wss://xrplcluster.com` or `wss://s2.ripple.com`) for a minute, and look for your master key or your current signing key (`validators` RPC, `signing_keys`). I saw 15 per minute on both servers, one per ledger.

## Rollback

Put the old compose file back and recreate the same way:

```bash
cp /path/to/backup/docker-compose.yml /path/to/docker-compose.yml
docker compose -p <project> -f /path/to/docker-compose.yml up -d --timeout 300 <service>
```

**The rollback expires.** Once the new amendment enables on the network, the old version is amendment blocked. After that, roll back only to another image that supports it. Keep your previous `localhost/xrpld` tag around for that reason.

## Common Mistakes

| Mistake | What happens |
|---|---|
| Skipping the Step 3 test | The node starts on a stock config with a new identity. The healthcheck stays green |
| Rebuilding the same tag | Every build is a different image (the build date is in it). The tag moves, and the next recreate changes your node's image without anyone asking |
| Tagging `latest` or no `localhost/` prefix | A pull can fetch someone else's image |
| Default 90 s stop grace | Long-running nodes can hit SIGKILL mid-close. Use 5 minutes |
| `docker compose down` | Stops every service in the project |
| Testing the new image with the real config and network | Two nodes signing with one validator key |
| Pruning the old image right after | No fast rollback |
| A healthcheck that passes on `full` | It can't tell your validator from a stock node running the wrong config. Check `proposing` and `pubkey_validator` yourself after every swap |

## My Run (3.3.0 to 3.4.1, 2026-09-26)

| Item | Result |
|---|---|
| Build time | ~20 s (BuildKit, Docker 28.3) |
| Image size | 236 MB, against 364 MB for `xrpllabsofficial/xrpld:3.3.0` (overlay2 store) |
| Old container stop | SIGTERM to exit 0 in 27.5 s after 20 days up. No SIGKILL |
| Without RPC | 28 s |
| Container start to `proposing` | 3 min 15 s (3 min 43 s from SIGTERM) |
| Quorum | back to 28 (read 29 for 13 s until the Negative UNL reapplied) |
| Identity | `pubkey_validator` and `pubkey_node` unchanged |
| Network | my validations seen on xrplcluster.com and s2.ripple.com, 15 per minute |
| `fixBatchV1_2` | `supported: true`, voting yes (it was `name=? supported=false` on 3.3.0) |

## Sources

- [xrpl.org: Introducing XRP Ledger version 3.4.1](https://xrpl.org/blog/2026/xrpld-3.4.1): the emergency release, the DEB SHA-256, the commit hash, and the 2026-10-09 expectation
- [xrpl.org: Introducing XRP Ledger version 3.4.0](https://xrpl.org/blog/2026/xrpld-3.4.0): the packages moved to packages.xrplf.org with the XRPLF key
- [XRPLF/rippled install.md](https://github.com/XRPLF/rippled/blob/develop/docs/install.md): the APT steps and the key fingerprint
- [xrpl.org: Install on Ubuntu or Debian Linux](https://xrpl.org/docs/infrastructure/installation/install-rippled-on-ubuntu): the fingerprint, and the supported Ubuntu versions
- [xrpl.org: Amendments](https://xrpl.org/docs/concepts/networks-and-servers/amendments): the two-week, 80% rule

## Appendix: The Two Files

These are the exact files I built and ran for 3.4.1. Treat them as a working example, not a drop-in for every setup. Read both before you build, and adjust them to your deployment. Both go in one directory.

| What may differ for you | Where to change it |
|---|---|
| xrpld version and .deb SHA-256 | Build args (`XRPLD_VERSION`, `XRPLD_PKG_VERSION`, `XRPLD_DEB_SHA256`), no file edit |
| Signing key fingerprint (if XRPLF rotates the key) | `expected_fpr` and the fingerprint `LABEL` in the Dockerfile, only after confirming the new key from both official sources |
| Ubuntu base image | The `FROM` digest, plus `base.digest` in the `LABEL` and the `base=` provenance line |
| Runtime uid/gid (997) | The `groupadd`/`useradd` lines and `USER`. A compose `user:` line overrides it without a rebuild |
| Where you mount your config | `entrypoint.sh` reads `/config/xrpld.cfg`, then `/config/rippled.cfg`, then `/config/validators.txt`. Match your mounts, or change those paths |

After any edit, rerun the Step 3 test before you swap.

### Dockerfile

```dockerfile
# xrpld in a container, built from the XRPL Foundation's signed DEB package.
#
# This directory holds two files, this one and entrypoint.sh. Build from it:
#
#   docker build \
#     --build-arg XRPLD_VERSION=3.4.1 \
#     --build-arg XRPLD_PKG_VERSION=3.4.1-1 \
#     --build-arg XRPLD_DEB_SHA256=cae8ce3b9bc9451b19975c890714ba789d2004987cdfff9cbd55522c612c6f26 \
#     --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
#     --tag localhost/xrpld:3.4.1 .
#
# The SHA-256 is the one the release announcement publishes for that .deb
# (https://xrpl.org/blog/2026/xrpld-3.4.1). The localhost/ prefix means the
# name can never be resolved against Docker Hub: a pull of it fails instead of
# fetching someone else's image.
#
# Why this exists: xrpld 3.4.x is published as signed DEB and RPM packages at
# packages.xrplf.org, and the install guide has no Docker section. This file
# follows that guide's APT steps exactly, inside an image:
#   https://github.com/XRPLF/rippled/blob/4a4fded2eba11427c48ce3f24d9c1aea5e7a9d17/docs/install.md#L34-L83
#
# It is written as a drop-in replacement for xrpllabsofficial/xrpld, which
# stops at 3.3.0. "The previous image" below means that one.
#
# What it holds to:
#
#   - The base image is pinned by digest. ubuntu:24.04 because the xrpl.org
#     Ubuntu install page names 22.04 and 24.04 on x86_64 as the most
#     supported and tested, and because the previous image is 24.04 too.
#     A digest does not move, so base-layer patches arrive when the digest in
#     this line is bumped on purpose, never behind anyone's back.
#   - The signing key is fetched and its fingerprint checked against the
#     published value before apt is allowed to trust it. The key file must
#     hold exactly one primary key, no subkeys, with that fingerprint: apt's
#     signed-by trusts every key in the file, so the file's whole shape is
#     checked, not one line of it. Any mismatch fails the build.
#   - The repository line is the install guide's, character for character.
#   - apt's signature checking is left on. Nothing here passes trusted=yes,
#     --allow-unauthenticated or an insecure-repository option.
#   - The package is installed at one exact version (xrpld=<version>-<rev>),
#     and, when a SHA-256 is passed, the downloaded .deb must match it. The
#     release announcement publishes that hash.
#   - The runtime account is xrpld, uid 997 gid 997, the same account and
#     numbers the previous image ships. A `--user` or compose `user:` still
#     wins, as it did before.
#   - Same entrypoint path and argument contract as the previous image (see
#     entrypoint.sh for why it is kept), and no HEALTHCHECK, because that
#     image has none and deployments carry their own.
#   - Nothing goes in but xrpld, its package's own files, ca-certificates and
#     the base image. Labels and the provenance file record only facts about
#     the package and the base.

FROM ubuntu:24.04@sha256:008173c23f95b170204355c12626cb5a965d779a7e1283b09e9cffbb1bf33ca3

# xrpld's upstream version (3.4.1) and the package version apt knows
# (3.4.1-1). Both are required.
ARG XRPLD_VERSION=
ARG XRPLD_PKG_VERSION=
# SHA-256 of the .deb as the release announcement publishes it. Optional here,
# because the package is already verified through the signed repository
# metadata; when set, it is a second, independent check on the same bytes.
ARG XRPLD_DEB_SHA256=
# RFC 3339 UTC, passed in so the label is the build's own time.
ARG BUILD_DATE=

RUN set -eu; \
    export DEBIAN_FRONTEND=noninteractive; \
    fail() { echo "BUILD REFUSED: $*" >&2; exit 1; }; \
    case "$XRPLD_VERSION" in ''|*[!0-9.]*) fail "XRPLD_VERSION must be digits and dots, got '$XRPLD_VERSION'";; esac; \
    case "$XRPLD_PKG_VERSION" in "${XRPLD_VERSION}"-[0-9]*) ;; *) fail "XRPLD_PKG_VERSION '$XRPLD_PKG_VERSION' is not a revision of $XRPLD_VERSION";; esac; \
    case "$XRPLD_PKG_VERSION" in *[!0-9.-]*) fail "XRPLD_PKG_VERSION must be digits, dots and one dash";; esac; \
    case "$XRPLD_DEB_SHA256" in '') ;; *[!0-9a-f]*) fail "XRPLD_DEB_SHA256 must be lowercase hex";; *) [ "${#XRPLD_DEB_SHA256}" -eq 64 ] || fail "XRPLD_DEB_SHA256 must be 64 hex characters";; esac; \
    [ -n "$BUILD_DATE" ] || fail "BUILD_DATE is required"; \
    \
    # Tools for fetching and reading the key. Removed again below.
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl gnupg; \
    \
    # Install guide step 2: the XRPL Foundation package-signing key.
    install -d -m 0755 /etc/apt/keyrings; \
    curl -fsS --proto '=https' --tlsv1.2 https://packages.xrplf.org/xrplf.asc -o /etc/apt/keyrings/xrplf.asc; \
    \
    # Install guide step 3: check the fingerprint, and fail if it differs.
    # Published in the install guide (L50-L64) and on xrpl.org's Ubuntu page.
    expected_fpr=B655416741221F780FBCFBC9AA84D41A11D29FA9; \
    GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
    gpg --batch --show-keys --with-colons /etc/apt/keyrings/xrplf.asc > "$GNUPGHOME/keyinfo"; \
    pubs=$(grep -c '^pub:' "$GNUPGHOME/keyinfo" || true); \
    subs=$(grep -c '^sub:' "$GNUPGHOME/keyinfo" || true); \
    fpr=$(awk -F: '$1 == "pub" { p = 1; next } p && $1 == "fpr" { print $10; exit }' "$GNUPGHOME/keyinfo"); \
    [ "$pubs" = 1 ] || fail "the key file holds $pubs primary keys, expected exactly 1"; \
    [ "$subs" = 0 ] || fail "the key file holds $subs subkeys, expected none"; \
    [ "$fpr" = "$expected_fpr" ] || fail "key fingerprint is '$fpr', expected $expected_fpr"; \
    echo "signing key fingerprint verified: $fpr"; \
    rm -rf "$GNUPGHOME"; unset GNUPGHOME; \
    \
    # Install guide step 4: the stable channel, exactly as written there.
    echo "deb [signed-by=/etc/apt/keyrings/xrplf.asc] https://packages.xrplf.org/repository/deb-stable any main" > /etc/apt/sources.list.d/xrplf.list; \
    \
    # The runtime account, before anything can allocate 997. The package's
    # sysusers entry ("u xrpld -") then finds it and leaves it alone.
    groupadd --system --gid 997 xrpld; \
    useradd --system --uid 997 --gid 997 --home-dir /var/lib/xrpld --no-create-home \
            --shell /sbin/nologin --comment "XRP Ledger daemon" xrpld; \
    install -d -o xrpld -g xrpld -m 0750 /var/lib/xrpld /var/log/xrpld; \
    \
    # Install guide steps 5 and 6, at one exact version. The package depends
    # on "systemd | systemd-standalone-sysusers | systemd-sysusers"; installed
    # first, the standalone one satisfies it without pulling in systemd.
    apt-get update; \
    apt-get install -y --no-install-recommends systemd-standalone-sysusers; \
    apt-get install -y --no-install-recommends --download-only "xrpld=${XRPLD_PKG_VERSION}"; \
    deb="/var/cache/apt/archives/xrpld_${XRPLD_PKG_VERSION}_amd64.deb"; \
    [ -f "$deb" ] || fail "apt did not leave $deb in its cache"; \
    deb_sha=$(sha256sum "$deb" | cut -d' ' -f1); \
    if [ -n "$XRPLD_DEB_SHA256" ]; then \
      [ "$deb_sha" = "$XRPLD_DEB_SHA256" ] || fail "the .deb's SHA-256 is $deb_sha, the announcement says $XRPLD_DEB_SHA256"; \
      echo "package SHA-256 matches the announcement: $deb_sha"; \
    fi; \
    apt-get install -y --no-install-recommends "xrpld=${XRPLD_PKG_VERSION}"; \
    \
    # What was installed is what was asked for, by the package database and
    # by the binary itself.
    got_pkg=$(dpkg-query -W -f='${Version}' xrpld); \
    [ "$got_pkg" = "$XRPLD_PKG_VERSION" ] || fail "dpkg reports xrpld $got_pkg, asked for $XRPLD_PKG_VERSION"; \
    got_bin=$(/usr/bin/xrpld --version | head -1); \
    [ "$got_bin" = "xrpld version ${XRPLD_VERSION}" ] || fail "the binary reports '$got_bin', asked for $XRPLD_VERSION"; \
    [ "$(id -u xrpld):$(id -g xrpld)" = "997:997" ] || fail "the xrpld account is $(id -u xrpld):$(id -g xrpld), expected 997:997"; \
    \
    # A record of what this image is, readable with `docker run --rm
    # --entrypoint cat <image> /usr/share/doc/xrpld-image/provenance`.
    install -d -m 0755 /usr/share/doc/xrpld-image; \
    { echo "xrpld_version=${XRPLD_VERSION}"; \
      echo "package=xrpld=${got_pkg}"; \
      echo "package_sha256=${deb_sha}"; \
      echo "apt_source=deb [signed-by=/etc/apt/keyrings/xrplf.asc] https://packages.xrplf.org/repository/deb-stable any main"; \
      echo "signing_key_fingerprint=${expected_fpr}"; \
      echo "binary=$(/usr/bin/xrpld --version | tr '\n' ' ')"; \
      echo "base=ubuntu:24.04@sha256:008173c23f95b170204355c12626cb5a965d779a7e1283b09e9cffbb1bf33ca3"; \
      echo "build_date=${BUILD_DATE}"; \
    } > /usr/share/doc/xrpld-image/provenance; \
    \
    # Nothing the node needs at runtime is removed: ca-certificates stays,
    # because xrpld fetches its validator list over HTTPS. curl and gnupg
    # were only for the key.
    apt-get purge -y --auto-remove curl gnupg; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*.deb

COPY entrypoint.sh /entrypoint.sh
RUN chmod 0755 /entrypoint.sh

# Keep these after the RUN above so a label change does not rebuild the
# package layer. Only facts about xrpld, its package and the base image.
LABEL org.opencontainers.image.title="xrpld" \
      org.opencontainers.image.version="${XRPLD_VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://packages.xrplf.org/repository/deb-stable" \
      org.opencontainers.image.documentation="https://github.com/XRPLF/rippled/blob/release/3.4.x/docs/install.md" \
      org.opencontainers.image.licenses="ISC" \
      org.opencontainers.image.base.name="docker.io/library/ubuntu:24.04" \
      org.opencontainers.image.base.digest="sha256:008173c23f95b170204355c12626cb5a965d779a7e1283b09e9cffbb1bf33ca3" \
      xrpld.package.version="${XRPLD_PKG_VERSION}" \
      xrpld.package.sha256="${XRPLD_DEB_SHA256}" \
      xrpld.package.signing-key-fingerprint="B655416741221F780FBCFBC9AA84D41A11D29FA9"

# Documentation only, EXPOSE publishes nothing: the package's default config
# binds peer 2459, admin RPC 5005 and admin WebSocket 6006, and 51235 is the
# conventional public peer port a deployment maps onto 2459.
EXPOSE 2459 5005 6006 51235

USER 997:997
ENTRYPOINT ["/entrypoint.sh"]
```

### entrypoint.sh

```bash
#!/bin/bash
# Entrypoint for an xrpld image built from the signed DEB package.
#
# Why it exists at all: it keeps the contract of the previous image,
# xrpllabsofficial/xrpld (up to 3.3.0), whose deployments mount their config
# at /config and rely on the entrypoint to copy it where xrpld reads it. A
# deployment that mounts only /config/rippled.cfg needs this file; without it
# xrpld would start on the package's default config. The contract:
#
#   /config/xrpld.cfg, else /config/rippled.cfg   copied to /etc/xrpld/xrpld.cfg
#   /config/validators.txt                        copied to /etc/xrpld/validators.txt
#   then: exec /usr/bin/xrpld --conf /etc/xrpld/xrpld.cfg <args> $ENV_ARGS
#
# Two differences from the previous entrypoint, both deliberate:
#
#   - No environment dump. The old one ran `printenv` at every start, which
#     puts whatever a deployment passes in the environment into the log.
#   - A copy onto a file that already holds the same bytes is skipped. A
#     non-root node with its config mounted read-only at both paths used to
#     print one "Read-only file system" line per file at every start.
#
# A copy that is needed and fails does NOT stop the node, same as before:
# xrpld then reads whatever /etc/xrpld/xrpld.cfg holds, and says so below.
set -u

CFG=/etc/xrpld/xrpld.cfg
VALIDATORS=/etc/xrpld/validators.txt

copy_in() {
  local src=$1 dst=$2
  if cmp -s "$src" "$dst"; then
    echo "entrypoint: $dst already matches $src"
    return 0
  fi
  if cat "$src" > "$dst"; then
    echo "entrypoint: copied $src to $dst"
  else
    echo "entrypoint: could not copy $src to $dst; xrpld reads $dst as it is" >&2
  fi
}

if [ -s /config/xrpld.cfg ]; then
  copy_in /config/xrpld.cfg "$CFG"
elif [ -s /config/rippled.cfg ]; then
  copy_in /config/rippled.cfg "$CFG"
fi

if [ -s /config/validators.txt ]; then
  copy_in /config/validators.txt "$VALIDATORS"
fi

# ENV_ARGS is split into words on purpose, as the image this replaces did.
# shellcheck disable=SC2086
exec /usr/bin/xrpld --conf "$CFG" "$@" ${ENV_ARGS:-}
```
