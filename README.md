# sochinorth OpenWrt feed

apk feed for [routebot](https://github.com/sochinorth/routebot) and
[omnidns](https://github.com/sochinorth/omnidns), rebuilt on every commit to
their default branches and served at <https://sochinorth.cc/openwrt-pkgs/>.

## Using it on a router (OpenWrt 25.12)

```sh
wget -O /etc/apk/keys/sochinorth.pem https://sochinorth.cc/openwrt-pkgs/sochinorth.pem
echo "https://sochinorth.cc/openwrt-pkgs/25.12/$(cat /etc/apk/arch)/packages.adb" >> /etc/apk/repositories.d/customfeeds.list
apk update
apk add routebot omnidns
```

Architectures: `mipsel_24kc` (ramips/mt7621), `mips_24kc` (ath79),
`aarch64_cortex-a53` (filogic, ipq807x, ...), `aarch64_cortex-a72` (RPi 4),
`aarch64_generic` (armsr, rockchip), `arm_cortex-a7_neon-vfpv4` (ipq40xx),
`x86_64`.

## How it works

```
routebot / omnidns push
  └─ .github/workflows/openwrt.yml ── calls ──> build-package.yml (this repo)
       ├─ build: SDK $OPENWRT_RELEASE, one job per arch, PKG_RELEASE = commit count
       └─ publish (default branch only): replace the repo's "snapshot" release,
          repository_dispatch "package-updated" here
publish-feed.yml
  └─ download every snapshot .apk, sign each .apk and a per-arch packages.adb
     with APK_SIGNING_KEY, deploy to Pages
```

- Target list and OpenWrt release: `build-package.yml` (`matrix`, `OPENWRT_RELEASE`).
  When moving to a new release series, bump `SERIES` in `publish-feed.yml` too.
- Adding another package repo: give it the same caller workflow and add it to
  `PACKAGE_REPOS` in `publish-feed.yml`.

## Secrets

| Repo | Secret | What |
|------|--------|------|
| openwrt-pkgs | `APK_SIGNING_KEY` | ECDSA P-256 private key (PEM) |
| routebot, omnidns | `FEED_TOKEN` | fine-grained PAT, repo `sochinorth/openwrt-pkgs`, Contents: read and write |

The public half is committed as `sochinorth.pem` and published next to the
feed; the workflow refuses to sign if the secret doesn't match it. To rotate,
generate a new pair, replace `sochinorth.pem` and the secret together
(keep `private-key.pem` offline, never commit it):

```sh
openssl ecparam -name prime256v1 -genkey -noout -out private-key.pem
openssl ec -in private-key.pem -pubout -out sochinorth.pem
```
