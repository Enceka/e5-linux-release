# Unisoc Linux releases

This repository owns build workflows and published artifacts for
[sprd-linux-devs/e5-linux](https://github.com/sprd-linux-devs/e5-linux).
Device configuration, kernel adaptations and infoscreen stay in their source repositories.

## Build and publish

Open Actions → **Build and publish OpenWrt** → **Run workflow**.
Choose the e5-linux source commit/ref, kernel ref and OpenWrt version.
Disable `publish` to upload build artifacts without creating a release.
The source checkout restores all device and screen commits pinned by its submodules.
The workflow resolves the selected source ref to one commit before starting any
platform build. Windows, Apple Silicon and firmware packaging all use that commit.

The Windows and macOS reusable workflows build their native installer artifacts.
The ARM64 firmware job downloads both, restores the Mac TAR archive to preserve
permissions and verifies their complete source/file fingerprints before packaging.
An unavailable or stale installer stops packaging; the release never omits a platform.

Tags matching `v*` in this repository also trigger a build. For tag builds,
`E5_SOURCE_REF` chooses the source revision (default `main`); `E5_KERNEL_REF`
chooses the kernel revision (default `e5-6.18`). The release tag belongs to
this repository, and `build.json` records the actual e5-linux/kernel/device/screen commits.
A workflow source hash is not used as the firmware source hash.

The shared kernel fixes are committed on the existing `e5-6.18` branch. The source
manifest `upstream/kernel-source.json` specifies the minimum revision, checked before
compilation. Generic builds no longer apply kernel patches at runtime. Kernel versions
keep the base version and device suffix without SCM/dirty suffixes; `build.json` still
records the exact source commits. No additional kernel branch is required.

## Bootstrap variables

Set repository Actions variables `E5_CI_INPUTS_URL` and `E5_CI_INPUTS_SHA256`
to the verified bootstrap archive described in the source project's
[release guide](https://github.com/sprd-linux-devs/e5-linux/blob/main/docs/RELEASE.md).
These public bootstrap variables are migrated from the source repository when available.
No device firmware, credentials or vendor runtime is placed in this repository.

The existing public bootstrap archive is hosted under
[`sprd-linux-devs/e5-linux` → `build-inputs`](https://github.com/sprd-linux-devs/e5-linux/releases/tag/build-inputs).
Use its current asset URL, not the retired `Enceka/e5-linux` owner path. The
verified archive SHA-256 remains `b9a67fecd5393da8fae58457d841a73105e664af19b5355f3c71d922400536d6`.

The workflow builds ARM64 OpenWrt flash ZIP/TAR.GZ bundles and the Android-to-Linux
Magisk ZIP, validates source hashes/modules/archive integrity and the OpenSSL executable,
and publishes through a draft release after checks pass. Published releases are not overwritten.
Source integration and workflow migration do not themselves trigger a firmware build.

## CI recovery (2026-10-18)

The previous run stopped before kernel compilation in the infoscreen USB settings
fixture: unknown IDs queried unavailable hardware instead of returning `no such setting`.
The firmware source now pins the USB guard and network fixtures that load the actual
shared hotspot module. USB settings, Wi-Fi startup, real API requests and diagnostic
redaction are checked in disposable OpenWrt containers.

The Linux runner installs the pinned desktop test dependencies in a virtual environment
and exercises the actual static BusyBox from the bootstrap inputs. Parent-owned
partition/GPT, update, TF, bundled-tool and packaging regressions use their current paths.
Existing APN, screen, voice, modem, browser and archive validation remain enabled.

For troubleshooting, dispatch a build with `publish` disabled. It runs the complete
build and retains validated artifacts without creating a GitHub Release.

## 发布入口

发布 workflow 已从 e5-linux 迁移到本仓库。手动构建可选择源码、内核和 OpenWrt 版本；
设备与信息屏固定版本随源码检出，构建结果发布在本仓库的 Releases。
历史发布仍保留在原源码仓库。先配置 bootstrap URL/SHA256，再执行发布工作流。

The ARM64 runner installs a digest-pinned QEMU 9.2.2 emulator for the x86 SDK
container. It builds that container before kernel compilation to verify APT
signatures and executable host tools early. The runner's distribution QEMU 8.2.2
produced false keyring errors in the Ubuntu 22.04 SDK image; signature validation
remains enabled.

The guarded block-mount APK is saved immediately after its build, so a failing later
step no longer discards that emulated SDK build.

ModemManager, BlueZ and hostapd are built in one step against the same
`e5-openwrt-src` tree, whose host tools belong to the runner's own architecture.
Their APKs and the Wi-Fi APKs share one cache entry: a cache hit skips the whole
step and the image assembly reuses the restored APKs instead of rebuilding hostapd
in a fresh tree.

## Landscape daemon

The Landscape image does not ship the release's `landscape-webserver`. 0.26.3 drops
every IPv6 packet on a WAN link that has no bound IPv6 address, which is this
device's modem uplink, so the image carries a daemon built from the pinned upstream
commit `e9f019b5` with the patches in the source repository's `openwrt/patches`.

`Build Landscape Router OpenWrt` builds it in its own `daemon` job
(`openwrt/build-landscape.sh`, which pins the commit and its tree hash and applies
every `openwrt/patches/landscape-*.patch`). The job uploads the binary with its
SHA-256, its build id and a provenance file; the image job restores it, checks that
the build id still matches this source tree, and assembles the bundle with
`E5_LANDSCAPE_DAEMON_REQUIRED=1`, so a missing or stale daemon fails the run
instead of falling back to the release binary. The bundle check then compares the
image's `/usr/sbin/landscape-webserver` with that artifact byte for byte.

### One entry point, any kernel line

`Build Landscape Router OpenWrt` is the only entry point, and `kernel_ref`
picks the kernel line it builds. The branch is named `e5-<line>-experiment`,
where `<line>` is the kernel version with dots (`e5-6.18-experiment` by default,
`e5-7.2-experiment` for the E5's 7.2 kernel); the `resolve` job strips an
optional `owner/` prefix, rejects any other shape and derives the rest of the
run from the line. `source_ref` is independent of it, because a kernel line may
be built from any source branch. The block-mount, daemon, kernel and image
steps, their verification and their failure modes exist once, so every kernel
line gets the same build.

What a kernel line builds is the manifest `upstream/kernel-source-<line>.json`,
which `resolve` derives and checks before anything is compiled — a line that has
no manifest fails the run with the path it expected, naming it, which is the
proof that the line was actually provisioned. The manifest pins the kernel
revision the build is allowed to compile, and it is what `upstream/build.sh` and
`upstream/check-kernel-source.py` read; it in turn names the shared Landscape
config fragment `upstream/e5-landscape.config`, so the release baseline
`upstream/kernel-source.json` and its `e5-mainline.config` stay untouched for
the published firmware. The derived line does not build anything by itself — it
only labels the run, which is why every artifact is named
`...-<kernel line>-<run id>-<attempt>` and a 7.2 artifact can never be mistaken
for a 6.18 one when a run is being inspected.

Adding a kernel line therefore means creating the kernel branch
`e5-<line>-experiment` and committing its manifest in the source repository —
not editing the workflow. The run starts from the Actions UI with no input at
all, and takes the OpenWrt and Landscape versions, `kernel_ref`,
`source_ref` and `keep_diagnostics`. The concurrency group carries the kernel
branch, so a 7.2 dispatch never cancels a 6.18 one.
