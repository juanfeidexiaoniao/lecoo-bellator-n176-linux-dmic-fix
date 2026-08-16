# Lecoo Bellator N176：Ubuntu 22.04 / Linux 6.8 内置麦克风修复记录

> English summary: On the Lecoo Bellator N176, Linux 6.8 binds the AMD ACP `1022:15e2` rev `0x62` device to `snd_rpl_pci_acp6x`, which exposes no DMIC. This repository backports the upstream Linux 7.1 YC/DMIC support as two DKMS modules, blacklists the obsolete RPL driver, and sets `pdm_gain=0` to avoid severe clipping.

## 适用范围

本文记录一次已经实际验证的修复，目标环境为：

- 机器：Lecoo Bellator N176
- 系统：Ubuntu 22.04.5 LTS
- 内核：`6.8.0-124-generic`（同系列 6.8 内核可由 DKMS 自动重建）
- ACP 设备：AMD `1022:15e2`，revision `0x62`，subsystem `17aa:383d`
- HDA codec：Conexant SN6140
- 音频服务：PulseAudio 15.99.1

这不是通用麦克风“万能修复”。执行前先确认 DMI 与 PCI ID；其他机器不应直接套用 DMI quirk。

## 症状

应用经常读不到“默认麦克风”。系统层面的表现是：

```text
Default Source: alsa_output.pci-0000_07_00.6.analog-stereo.monitor
```

也就是默认输入落到了扬声器的 monitor，而不是真实麦克风。PulseAudio 最初只能看到：

```text
alsa_output.pci-0000_07_00.6.analog-stereo.monitor
```

`arecord -l` 虽能看到 SN6140 的模拟录音端，但它对应右侧外接麦克风插孔；插孔未连接时可用性为 `no`。内置数字麦克风完全没有出现。

## 根因定位

### 1. 固件和 HDA 路径只暴露了外接模拟麦克风

SN6140 codec 的 pin 信息中：

```text
Node 0x19: Mic at Ext Right
Mic Jack: off
```

因此 PulseAudio 恢复为“仅输出”profile 后，没有可用的真实输入源，只能回退到输出 monitor。

### 2. 内置 DMIC 属于 AMD ACP，但被旧驱动占用

真正的关键设备是：

```text
07:00.5 Multimedia controller [0480]: AMD Audio Processor [1022:15e2] (rev 62)
Subsystem: Lenovo [17aa:383d]
Kernel driver in use: snd_rpl_pci_acp6x
```

Linux 6.8 的 `snd_rpl_pci_acp6x` 会占用该设备，却不会为 Bellator N176 创建 PDM/DMIC 声卡。

Linux 上游随后加入了两部分修正：

1. 让 YC ACP 驱动接受 RPL revision `0x62`；
2. 为 `Lecoo / Bellator N176` 加入 DMI quirk，启用 ACP DMIC。

相关上游讨论和补丁：

- [ASoC: Add DMIC support for the AMD RPL platform (v6 1/3)](https://patchew.org/linux/20260213055904.110256-1-qby140326%40gmail.com/20260213055904.110256-2-qby140326%40gmail.com/)
- [ASoC: Add quirk for Lecoo Bellator N176 (v6 3/3)](https://patchew.org/linux/20260213055904.110256-1-qby140326%40gmail.com/20260213055904.110256-4-qby140326%40gmail.com/)

该支持已进入 Linux 7.1。本文针对仍使用 Ubuntu 22.04 / Linux 6.8 的系统做最小回移。

### 3. 默认 PDM 增益会严重削波

DMIC 出现后，`snd_acp6x_pdm_dma` 的默认参数为：

```text
pdm_gain=3
```

实测录音出现大量 `+1.0 / -1.0` 满幅样本和明显直流偏置。设置 `pdm_gain=0` 后，削波消失；最终 5 秒环境录音约为：

```text
Peak:    -25.44 dB
RMS:     -29.43 dB
Clipped: 0
```

## 为什么没有直接安装 7.1 主线内核

Ubuntu 官方确实提供了 7.1 主线包，但其 generic headers 依赖较新的 `libc6 (>= 2.38)`、`libdw1t64`、`libelf1t64` 和 `libssl3t64`。Ubuntu 22.04 不满足这些依赖，NVIDIA DKMS 也就无法为新内核重建。

因此本次选择只回移两个小型声卡模块，继续使用发行版的 6.8 内核，保留 NVIDIA 驱动兼容性和现有回退内核。

## 修复方法

### 0. 核对机器

```bash
cat /sys/class/dmi/id/board_vendor
cat /sys/class/dmi/id/product_name
lspci -nnk -s 07:00.5
```

预期包含：

```text
Lecoo
Bellator N176
1022:15e2 (rev 62)
17aa:383d
```

### 1. 准备构建环境

```bash
sudo apt install build-essential dkms curl linux-headers-"$(uname -r)"
```

### 2. 准备 DKMS 源码

```bash
sudo install -d -m 0755 /usr/src/bellator-n176-dmic-1.0
cd /usr/src/bellator-n176-dmic-1.0

base=https://raw.githubusercontent.com/torvalds/linux/v6.8/sound/soc/amd/yc
sudo curl -fLO "$base/pci-acp6x.c"
sudo curl -fLO "$base/acp6x-mach.c"
sudo curl -fLO "$base/acp6x.h"
sudo curl -fLO "$base/acp6x_chip_offset_byte.h"
```

把本仓库中的文件复制到对应位置：

```bash
sudo cp dkms/Makefile dkms/dkms.conf /usr/src/bellator-n176-dmic-1.0/
sudo patch -d /usr/src/bellator-n176-dmic-1.0 -p1 < patches/0001-asoc-amd-enable-bellator-n176-dmic.patch
```

### 3. 构建并安装 DKMS 模块

```bash
sudo dkms add -m bellator-n176-dmic -v 1.0
sudo dkms build -m bellator-n176-dmic -v 1.0 -k "$(uname -r)"
sudo dkms install -m bellator-n176-dmic -v 1.0 -k "$(uname -r)"
```

确认模块解析到 DKMS 版本：

```bash
modinfo -n snd_pci_acp6x
modinfo -n snd_soc_acp6x_mach
```

预期路径位于：

```text
/lib/modules/<kernel>/updates/dkms/
```

### 4. 配置驱动选择与增益

```bash
sudo cp config/bellator-n176-dmic.conf /etc/modprobe.d/
sudo cp config/bellator-n176-dmic.modules /etc/modules-load.d/bellator-n176-dmic.conf
sudo update-initramfs -u -k "$(uname -r)"
```

配置的作用是：

- 禁止旧 `snd_rpl_pci_acp6x` 自动占用 ACP；
- 让 DMIC 依赖模块先于 YC PCI 驱动加载；
- 将 `pdm_gain` 固定为 0，避免削波；
- 开机自动加载 `snd_pci_acp6x`。

### 5. 重启，或在线切换驱动

最稳妥的方式是重启：

```bash
sudo reboot
```

也可以在线切换，但会短暂中断当前音频会话：

```bash
systemctl --user stop pulseaudio.service pulseaudio.socket

sudo modprobe -r snd_pci_acp6x 2>/dev/null || true
sudo modprobe -r snd_rpl_pci_acp6x
sudo modprobe -r snd_soc_acp6x_mach 2>/dev/null || true
sudo modprobe -r snd_acp6x_pdm_dma 2>/dev/null || true

sudo modprobe snd_pci_acp6x

systemctl --user start pulseaudio.socket pulseaudio.service
```

### 6. 设置默认麦克风

驱动正确加载后，PulseAudio 会出现名称中含 `hw_acp6x` 的输入源：

```bash
pactl list short sources
```

设置它为默认输入：

```bash
dmic_source="$(pactl list short sources | awk '$2 ~ /hw_acp6x/ {print $2; exit}')"
pactl set-default-source "$dmic_source"
pactl set-source-mute "$dmic_source" 0
pactl set-source-volume "$dmic_source" 100%
```

## 验证

### 内核绑定

```bash
lspci -nnk -s 07:00.5
```

预期：

```text
Kernel driver in use: snd_pci_acp6x
```

### ALSA 设备

```bash
arecord -l
```

预期新增：

```text
card: acp6x
device 0: DMIC capture dmic-hifi-0
```

### 默认输入源

```bash
pactl get-default-source
pactl get-source-mute "$(pactl get-default-source)"
pactl get-source-volume "$(pactl get-default-source)"
```

默认源应包含 `hw_acp6x`，且为未静音状态。

### 录音

```bash
timeout 5 parecord \
  --device="$(pactl get-default-source)" \
  --file-format=wav \
  --rate=48000 \
  --channels=2 \
  /tmp/bellator-dmic-test.wav

sox /tmp/bellator-dmic-test.wav -n stats
```

重点检查是否仍出现 0 dB 满幅削波。

## 回滚

如果补丁在你的系统上不适用：

```bash
sudo dkms remove -m bellator-n176-dmic -v 1.0 --all
sudo rm -f /etc/modprobe.d/bellator-n176-dmic.conf
sudo rm -f /etc/modules-load.d/bellator-n176-dmic.conf
sudo update-initramfs -u -k "$(uname -r)"
sudo reboot
```

DKMS 会恢复原始内核模块。也可以在 GRUB 的“高级选项”中启动未安装该 DKMS 模块的旧内核。

## 最终结果

- ACP 从 `snd_rpl_pci_acp6x` 切换为 `snd_pci_acp6x`；
- ALSA 出现独立 `acp6x / DMIC capture`；
- PulseAudio 默认输入变为 `hw_acp6x`；
- 麦克风未静音，软件音量 100%；
- `pdm_gain=0`，录音无满幅削波；
- NVIDIA 580 驱动保持正常；
- DKMS 会为后续 6.8 小版本内核自动重建。

## 说明

本文中的用户名、主机名和其他个人信息均已移除。内核代码遵循其原有的 GPL-2.0-or-later / GPL-2.0+ 许可标识；补丁仅包含对上游改动的最小回移。
