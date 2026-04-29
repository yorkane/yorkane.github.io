# PVE 9.1 Intel 核显直通 Windows 11 Code 43 完整修复记录

## 环境

| 项目 | 详情 |
|------|------|
| PVE 版本 | 9.1.1 |
| 内核 | 6.17.2-1-pve |
| QEMU | 10.1.2-3（官方版）→ 10.2.1-1（修改版） |
| CPU | Intel Core i9-14900K |
| iGPU | Intel UHD Graphics 770 (Device ID: 8086:a780) |
| 独显 | NVIDIA RTX 4090 |
| 虚拟机 | Windows 11, Q35 机型, OVMF UEFI |
| 虚拟机配置 | 16C/64G, ide0 系统盘, hostpci0 iGPU, hostpci1 RTX 4090 |

## 问题现象

将 Intel i9-14900K 的核显 (UHD 770) 直通给 Windows 11 虚拟机后，设备管理器中显示：

> **Intel(R) UHD Graphics 770 — Windows 已停止此设备，因为它报告了问题 (Code 43)**

同时 QEMU 日志中有 ROM 错误：

```
kvm: Invalid PCI ROM header signature: expecting 0xaa55, got 0xbb34
```

## 排障过程

### 尝试 1：SR-IOV 虚拟化方案 ❌

尝试通过 i915 驱动 + SR-IOV 创建 VF 虚拟函数直通，但失败：

- **i915 无法加载**：内核 6.17 编译选项 `CONFIG_DRM_I915_FORCE_PROBE=""`（空字符串），导致 `force_probe=a780` 无法生效
- **SR-IOV 需要内核头文件编译 DKMS**：PVE 9.1 只有 6.11 和 6.14 头文件包，没有 6.17 的
- **`i915.max_vfs=7` 参数不支持**：主线内核不含 SR-IOV 补丁，dmesg 显示 "unknown parameter 'max_vfs' ignored"
- **结论**：SR-IOV 方案在当前内核版本下不可行

### 尝试 2：rombar=0 跳过物理 ROM ❌

设置 `hostpci0: 0000:00:02,pcie=1,rombar=0`，让 QEMU 不加载损坏的物理 ROM：

- VM 启动后黑屏，Windows 反复重启
- 原因：Windows Intel 驱动对 ROM 变化非常敏感

### 尝试 3：romfile 替换为 LongQT-sea ROM ❌

从 [LongQT-sea/intel-igpu-passthru](https://github.com/LongQT-sea/intel-igpu-passthru) 下载 RKL_TGL_ADL_RPL GOP ROM（166KB）：

- `hostpci0: 0000:00:02,pcie=1,rombar=1,romfile=igd.rom`
- Windows 不再黑屏，能正常启动，但 **Code 43 仍然存在**
- 原因：官方 QEMU 10.1 的地址映射存在 Bug，ROM 本身没问题但 QEMU 层面有问题

### 尝试 4：i440fx 机型切换 ❌

Intel 核显直通的成熟方案大多要求 i440fx 机型：

- 切换到 `pc-i440fx-10.1` + `legacy-igd=1`
- RTX 4090 在传统 PCI 总线上无法正常输出，**物理显示器黑屏**
- 4090 必须依赖 Q35 的 PCIe 架构

### 最终方案：修改版 QEMU + QEMU10 专用 ROM ✅

根据 [途纳下载教程](https://www.2na.com/tutorial/54.html) 和 [AICodo/pve-emu-realpc](https://github.com/AICodo/pve-emu-realpc) 项目的方案：

- **根本原因**：PVE 9.1 官方 QEMU 10.1 存在 Windows VM iGPU 直通地址映射 Bug（[PVE 论坛已确认](https://forum.proxmox.com/threads/pve-9-0-4-igpu-passthrough-error-43.169951/)）
- **修复方法**：安装社区维护的修改版 QEMU + 修改版 OVMF + QEMU10 专用核显 ROM

## 最终修复步骤

### 第一步：下载文件

在 PVE 宿主机上下载三个文件：

```bash
cd /root

# 1. 修改版 QEMU 10.2.1
curl -L -o pve-qemu-kvm_10.2.1-1_amd64.deb \
  https://github.com/AICodo/pve-emu-realpc/releases/download/v20260329-214123-pve9/pve-qemu-kvm_10.2.1-1_amd64.deb

# 2. 修改版 OVMF 固件
curl -L -o pve-edk2-firmware-ovmf_4.2025.05-2_all.deb \
  https://github.com/AICodo/pve-emu-realpc/releases/download/v20260329-214123-pve9/pve-edk2-firmware-ovmf_4.2025.05-2_all.deb

# 3. QEMU10 专用核显 ROM（6-14 代台式机）
curl -L -o /usr/share/kvm/pc-12-13-14-q10.rom \
  https://github.com/AICodo/intel6-14rom/releases/download/v2.0-20260306-092336/pc-12-13-14-q10.rom
```

### 第二步：在 Windows 中卸载 Intel 驱动

**这一步非常重要，否则换 ROM 后 Windows 会绿屏/黑屏！**

1. 设备管理器 → 显示适配器
2. 右键 Intel UHD Graphics 770 → 卸载设备
3. **勾选"尝试删除此设备的驱动程序软件"**
4. 关闭 Windows

### 第三步：安装修改版组件

```bash
# 关闭所有虚拟机（QEMU 替换需要重启 QEMU 服务）
# 注意：会短暂影响所有运行中的 VM

# 安装修改版 QEMU
dpkg -i pve-qemu-kvm_10.2.1-1_amd64.deb

# 安装修改版 OVMF
dpkg -i pve-edk2-firmware-ovmf_4.2025.05-2_all.deb

# 锁定版本，防止 apt update 覆盖
apt-mark hold pve-qemu-kvm pve-edk2-firmware-ovmf
```

### 第四步：更新 VM 配置

```bash
# 设置 iGPU 直通参数
qm set <VMID> --hostpci0 '0000:00:02.0,romfile=pc-12-13-14-q10.rom'

# 设置 QEMU args
qm set <VMID> --args \
  '-set device.hostpci0.bus=pcie.0 \
   -set device.hostpci0.addr=0x02.0 \
   -set device.hostpci0.x-igd-gms=0x2 \
   -set device.hostpci0.x-igd-opregion=on \
   -set device.hostpci0.x-igd-lpc=on'

# 移除虚拟 VGA（如果不需要 noVNC 控制台）
qm set <VMID> --vga none
```

### 第五步：启动验证

```bash
qm start <VMID>
```

Windows 启动后，设备管理器中 Intel UHD Graphics 770 应显示正常工作，无 Code 43。

## 最终 VM 配置参考

```
args: -set device.hostpci0.bus=pcie.0 -set device.hostpci0.addr=0x02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on -set device.hostpci0.x-igd-lpc=on
bios: ovmf
boot: order=ide2;ide0
cores: 16
cpu: host
hostpci0: 0000:00:02.0,romfile=pc-12-13-14-q10.rom
hostpci1: 0000:01:00,pcie=1,x-vga=1
machine: pc-q35-10.1
memory: 65536
vga: none
```

## 回退方法

如需恢复官方版本：

```bash
apt-mark unhold pve-qemu-kvm pve-edk2-firmware-ovmf
apt reinstall pve-qemu-kvm pve-edk2-firmware-ovmf
```

## 注意事项

1. **PVE 更新时注意**：`apt-mark hold` 会阻止 QEMU/OVMF 自动更新，PVE 大版本升级时需要手动评估
2. **VM 类型**：修改版 QEMU 项目建议将 VM 的 `ostype` 设为 `l26`（Linux），声称可解决闪屏/黑屏问题
3. **xe 驱动**：PVE 9.1+ 内核引入了新的 `xe` 显卡驱动，可能会抢占 iGPU 设备，确保 `vfio-pci` 优先绑定
4. **物理 ROM 损坏**：如果 dmesg 中出现 `Invalid PCI ROM header signature`，说明物理 VBIOS ROM 可能损坏，必须使用 romfile 方案
5. **ROM 选择**：`pc-12-13-14-q10.rom` 是通用版，如果有问题可尝试提取本机的 `IntelGopDriver.efi` 编译专用 ROM

## 参考资源

- [AICodo/pve-emu-realpc](https://github.com/AICodo/pve-emu-realpc) — 修改版 QEMU（解决 Code 43）
- [AICodo/intel6-14rom](https://github.com/AICodo/intel6-14rom) — QEMU10 专用核显 ROM
- [途纳下载 - PVE9 核显直通教程](https://www.2na.com/tutorial/54.html)
- [LongQT-sea/intel-igpu-passthru](https://github.com/LongQT-sea/intel-igpu-passthru) — iGPU 直通 ROM（i440fx 方案）
- [PVE Forum - Error 43 讨论](https://forum.proxmox.com/threads/pve-9-0-4-igpu-passthrough-error-43.169951/)
- [lurenJBD/intel-rom](https://github.com/lurenJBD/intel-rom) — OVMF 补丁方案
