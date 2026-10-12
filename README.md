# Unisoc Linux releases

This repository owns build workflows and published artifacts for
[sprd-linux-devs/e5-linux](https://github.com/sprd-linux-devs/e5-linux).
Device configuration, kernel adaptations and infoscreen stay in their source repositories.

## Build and publish

Open Actions → **Build and publish OpenWrt** → **Run workflow**.
Choose the e5-linux source commit/ref, kernel ref and OpenWrt version.
Disable `publish` to upload build artifacts without creating a release.
The source checkout restores all device and screen commits pinned by its submodules.

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

The workflow builds ARM64 OpenWrt flash ZIP/TAR.GZ bundles and the Android-to-Linux
Magisk ZIP, validates source hashes/modules/archive integrity and the OpenSSL executable,
and publishes through a draft release after checks pass. Published releases are not overwritten.
Source integration and workflow migration do not themselves trigger a firmware build.

## 发布入口

发布 workflow 已从 e5-linux 迁移到本仓库。手动构建可选择源码、内核和 OpenWrt 版本；
设备与信息屏固定版本随源码检出，构建结果发布在本仓库的 Releases。
历史发布仍保留在原源码仓库。先配置 bootstrap URL/SHA256，再执行发布工作流。
