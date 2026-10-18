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
