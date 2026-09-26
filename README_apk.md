# APK Packages & Repository for luci-app-xray

This project builds three OpenWrt 25.12.x packages as **Alpine `apk`** (`.apk`)
packages, installable via the standard `apk` tool that ships with recent OpenWrt
releases (25.12+):

| Package | Description |
|---|---|
| `luci-app-xray` | Main LuCI app + CLI (`xray`). |
| `luci-app-xray-geodata` | GeoIP/GeoSite data sets. |
| `luci-app-xray-status` | Live status dashboard (uses `xray` + `wget`). |

All three packages are **`arch:noarch`** - a single APK works on every target
architecture (x86/x86_64/mipsel/mips64/aarch64/arm/armv7/.../ramips/...). The
published repository is a single flat directory, so one repo URL serves every
architecture.

## The published repository

The build pipeline publishes a **flat apk repository** to the `gh-pages` branch
of this repo (served by GitHub Pages):

    https://kinddevops.github.io/luci-app-xray/
      +-- packages.adb            # repository index (binary APKINDEX / ADB)
      +-- luci-app-xray-3.7.1-r1.apk
      +-- luci-app-xray-geodata-3.7.1-r1.apk
      +-- luci-app-xray-status-3.7.1-r1.apk

The URL pattern follows the same layout OpenWrt's own release repositories use:
a directory whose **last path component is `packages.adb`**, with the `.apk`
files sitting next to it. That layout is what lets the device's `apk` derive the
per-package download URLs from the repo entry.

> **Signing**: these packages are published **unsigned** (the OpenWrt SDK's
> `mkpkg` is invoked without `--sign`). Add the repo with `--allow-untrusted`
> (see below). If you later want signatures, enable `CONFIG_SIGNED_PACKAGES`
> with a `BUILD_KEY_APK_SEC` in the SDK build; the device then needs the
> matching public key under `/etc/apk/keys/`.

## Adding the repository to an OpenWrt system

OpenWrt 25.12+ replaces `opkg` with `apk` by default. To add a custom feed,
append one line per repository to **`/etc/apk/repositories`**:

    # one-off
    echo 'https://kinddevops.github.io/luci-app-xray/packages.adb' >> /etc/apk/repositories

or persistently:

    cat >> /etc/apk/repositories <<'REPO'
    # luci-app-xray (this project) - unsigned, noarch
    https://kinddevops.github.io/luci-app-xray/packages.adb
    REPO

This is the same file/format the OpenWrt `apk` package ships as
`/etc/apk/customfeeds.list` (a comment plus one URL per line, the URL ending in
`packages.adb`).

> If you point the Pages site at a different hostname (e.g. a custom domain),
> use that URL instead.

## Installing the packages

Because the published packages are **unsigned**, every `apk` command that
touches this repo must use `--allow-untrusted`:

    # refresh the index
    apk update --allow-untrusted

    # install the app plus its companion packages
    apk add --allow-untrusted \
        luci-app-xray \
        luci-app-xray-geodata \
        luci-app-xray-status

Only the core package (if you do not want the extras):

    apk add --allow-untrusted luci-app-xray

### Runtime dependencies

`luci-app-xray` depends on `firewall4`, `kmod-nft-tproxy`, `luci-base`,
`xray-core`, `dnsmasq`, `ca-bundle` - **none of which are published in this
repository**. Provide them from a source that matches your firmware:

1. **Stock OpenWrt 25.12+ firmware** ships `apk` and already has the stock
   release repo (`https://downloads.openwrt.org/releases/.../packages/`)
   configured. Add only this repo for the luci-app-xray packages; the rest
   resolves from the stock repo.
2. **`xray-core`** is not in stock OpenWrt. Install a matching `xray-core`
   (from `openwrt/packages` or a community feed for your kernel/arch) **before**
   adding `luci-app-xray`, or the install will fail to resolve.
3. If you are on an **opkg-based** firmware (no `apk`), these `.apk` files do
   **not** work there - use the regular OpenWrt `.ipk` builds instead.

### Uninstall / clean

    apk del --allow-untrusted \
        luci-app-xray luci-app-xray-geodata luci-app-xray-status

## Building locally (SDK)

The pipeline mirrors the documented SDK flow:

    # 1. Get an OpenWrt 25.12.2 SDK for any target (the architecture only
    #    affects the host toolchain used for metadata; produced APKs are noarch).
    # 2. Add this repo as a feed; run ./scripts/feeds install -a.
    # 3. Build only the three packages (skip the kernel + dep closure):
    cd $SDK_HOME
    make NO_DEPS=1 -j"$(nproc)" \
        package/luci-app-xray/core/compile \
        package/luci-app-xray/geodata/compile \
        package/luci-app-xray/status/compile
    # 4. The three .apk files land in bin/packages/<arch>/.
    # 5. Generate the repository index (same invocation OpenWrt's
    #    package/Makefile uses for its own repo index):
    STAGING=$SDK_HOME/staging_dir/host
    $STAGING/bin/apk mkndx \
        --root $SDK_HOME --keys-dir $SDK_HOME --allow-untrusted \
        --output packages.adb \
        luci-app-xray-*.apk luci-app-xray-geodata-*.apk luci-app-xray-status-*.apk

## Troubleshooting

| Symptom | Fix |
|---|---|
| `apk: could not update repo` | Confirm the GitHub Pages site is enabled (Settings > Pages, source `gh-pages`, root `/`). Test with `curl -I https://kinddevops.github.io/luci-app-xray/packages.adb`. |
| `apk: signature verification` | This repo is unsigned; pass `--allow-untrusted` to `apk update`/`apk add`. |
| `apk: could not solve` for `luci-app-xray` | A runtime dep (usually `xray-core`) is not installable from your repos - see "Runtime dependencies". |
| Package installs but state resets on reboot | Not a repo issue; see the app's own docs for state persistence. |
