# DroidspacesKernelBuild

编译支持 DroidSpaces 的 ZTE UMS9620 (MiFi F50) 内核

## 说明

本项目基于 [kumalum/DroidspacesKernelBuild](https://github.com/kumalum/DroidspacesKernelBuild) 的 GitHub Actions 构建流程改造，
目标内核从 Sony SM6375 替换为 ZTE UMS9620 (MiFi F50)：

- 内核源码：https://github.com/Enceka/android_kernel_zte_ums9620_mifi_f50 （ZTE 开源网站下载，Linux 5.4.210，展锐平台）
- 基础配置：`arch/arm64/configs/f50_defconfig`
- DroidSpaces 所需内核特性：`CONFIG_SYSVIPC` / `CONFIG_POSIX_MQUEUE` / `CONFIG_IPC_NS` / `CONFIG_PID_NS`
- KABI 补丁：将 SYSVIPC / POSIX_MQUEUE 字段放入 `ANDROID_KABI_*` 预留槽位，避免破坏内核 ABI

## 使用

1. 将该仓库推送到你自己的 GitHub 仓库
2. 手动触发 Actions（`workflow_dispatch`）或 push 到 `main` 分支
3. 编译产物（`arch/arm64/boot/Image`）会上传到 Actions 的 Artifacts

## 注意

- 展锐内核通常依赖特定 clang/gcc 版本，若纯 LLVM 编译报错，可调整 `.github/clang` 中的工具链选择（`CLANG: google|lineage|llvm`）
- 若内核编译通过但无法启动，请核对 `f50_defconfig` 与该设备实际固件的差异
