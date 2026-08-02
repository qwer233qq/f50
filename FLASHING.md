# ZTE UMS9620 (MiFi F50) 内核打包与刷机指南

本文档说明如何把 GitHub Actions 编译出的内核产物（`Image`）打包成可刷写的 `boot.img`，并刷入 ZTE UMS9620 (MiFi F50) 设备。

---

## 1. 准备

### 1.1 获取编译产物
- 打开 GitHub Actions 页面 → 找到成功的运行 → **Artifacts** → 下载 `new-kernel`
- 解压后得到内核本体：`Image`（约 26MB，ARM64 未压缩内核）

### 1.2 获取原厂 boot.img
从设备官方固件包中提取 `boot.img`（原厂包内通常叫 `boot.img`，或需从 `boot.img.lz4`/分区镜像中提取）。

> **重要**：boot.img 里包含设备原厂的 ramdisk（initrd），**ramdisk 必须保留原厂的**，我们只替换其中的内核 `Image`。

### 1.3 工具
- **AIK-Linux**（本仓库已自带 `AIK-Linux/`）：解包/打包 boot.img，需在 Linux x86_64 环境运行
  - 也可在 GitHub Actions 里跑，或在本机 WSL / Linux PC 上执行
- **刷机工具**（按设备情况二选一，见第 4 节）
  - `fastboot`（如果设备支持 fastboot 模式）
  - **SPD Flash Tool (ResearchDownload)**：展锐（Unisoc/SPRD）平台专用，MiFi F50 大概率用这个

---

## 2. 解包原厂 boot.img

```bash
cd AIK-Linux
./unpackimg.sh /path/to/original/boot.img
```

解包完成后：
- `boot.img-ramdisk/` — 原厂 ramdisk（**不要改**）
- `boot.img-kernel/` — 原厂内核文件（`Image` / `Image.gz` / `zImage` 等）

## 3. 替换内核并重新打包

把编译产物 `Image` 复制覆盖内核文件：

```bash
cp /path/to/new/Image AIK-Linux/boot.img-kernel/
# 如果原厂内核是 Image.gz 等压缩格式，AIK 会按 split_img 里的配置自动处理；
# 直接替换为未压缩 Image 即可，AIK 打包时会按 config 重新压缩
```

重新打包：

```bash
cd AIK-Linux
./repackimg.sh
# 生成新的 boot.img 文件（原 boot.img 会被备份为 image-new.img）
```

> AIK 的 `repackimg.sh` 会自动处理内核压缩格式与 ramdisk。如果打包后体积异常或无法启动，检查 `AIK-Linux/split_img/` 下的 `boot.img-cmdline`、`boot.img-base` 等参数是否与原厂一致。

## 4. 刷入设备

### 方式 A：fastboot（如果设备有 fastboot 模式）
```bash
adb reboot bootloader      # 或按厂商按键组合进 fastboot
fastboot flash boot new-boot.img
fastboot reboot
```

### 方式 B：SPD Flash Tool / ResearchDownload（展锐平台）
1. 打开 ResearchDownload 或厂商刷机工具
2. 加载该机型对应的 **分散加载文件**（scatter 文件，如 `*.xml`）
3. 只勾选 `boot` 分区，指定为新的 `boot.img`
4. 连接设备（关机后短接/按组合键进下载模式），点 **Download**

> 不同厂商工具界面不同，请以你实际下载到的工具为准。**刷入前务必保留原固件与 boot.img 备份**。

---

## 5. 注意事项

- **备份**：刷机前务必备份原厂 `boot.img`，变砖后可恢复
- **解锁/Secure Boot**：部分设备需要先解锁 bootloader 或关闭签名校验，否则可能无法启动或回滚被拒
- **DroidSpaces**：内核刷入后，还需在设备上安装 DroidSpaces 相关 App/容器镜像才能使用
- **验证内核生效**：
  ```bash
  adb shell
  cat /proc/version
  # 应看到新内核版本与编译时间
  ```
- **回滚**：用原厂 boot.img 重新打包刷回即可恢复官方状态

---

## 6. 故障排查

| 现象 | 可能原因 | 处理 |
|------|---------|------|
| 开机卡 logo / 无限重启 | 内核与 ramdisk 不匹配、缺少驱动 | 确认只替换内核、ramdisk 用原厂；确认 `f50_defconfig` 覆盖设备所需驱动 |
| fastboot 无设备 | 未装驱动 / 设备不支持 fastboot | 换 SPD 工具刷机；安装厂商 USB 驱动 |
| 刷完无信号 | 基带驱动/固件未匹配 | 内核不应影响基带；确认没动 modem 分区 |
| 内核版本没变 | 刷错分区 / 未生效 | 确认刷的是 boot 分区，`cat /proc/version` 验证 |
