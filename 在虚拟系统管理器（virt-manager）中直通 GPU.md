# 在虚拟系统管理器（virt-manager）中直通 GPU

## 适用目标

- 笔记本，无外接显示器
- 有核显/第二块显卡给宿主机用  
  （本教程的特殊情况：Linux 使用 AMD 核显，虚拟机内使用 Nvidia 独显）
- 宿主机用不到独显  
  （本教程的特殊情况：宿主机直接没有安装 Nvidia 的驱动）
- 有方法实现独显直连  
  （本教程的特殊情况：在 Windows 内可以通过 Type-C 口输出 DP 信号实现激活独显输出）
- 高质量、高帧率、低延迟的虚拟机画面与声音  
  （为了打游戏呢）
- 一定程度的虚拟机内外联动  
  （如剪切板共享、文件系统共享）

## 缺陷

- 性能一定不如直接物理机。在我的机器上，FurMark 测试，1920x1080，物理机下 100 帧左右，虚拟机内 85 帧左右。
- 有些游戏的反作弊程序会检测虚拟机，因为宿主机可以直接操作虚拟机的内存做到作弊操作，而这种作弊方式反作弊程序无法检测。
  不同游戏的策略不同，轻则游戏闪退，重则封号警告。  
  可以通过一些特殊操作将虚拟机模拟成物理机，但是移动端显卡在物理机上只有在检测到电池时才正常工作。  
  也有一些教程提供了模拟电池的方法，但我始终没有尝试成功。由于我没有这方面的需求，因此放弃研究。

## 运行环境

- 华硕天选 FA506IV  
  （AMD R7-4800H，Nvidia RTX 2060 Mobile）  
  （以下教程均以此硬件为例，AMD 平台与 Intel 平台的操作不尽相同，请注意区分）
- Kubuntu 22.04
- 一个 Typc-C 接口的显卡诱骗器

## 具体步骤

### 确认硬件支持

- CPU 需要支持硬件虚拟化和 IOMMU 特性。
- 主板需要支持 IOMMU 特性。
- GPU 需要支持 UEFI 启动  
  （2012 年及之后发布的 GPU 都应该支持，因为微软规定所有宣称兼容 Windows8 的设备都必须支持 UEFI）

### 设置 IOMMU

#### 启用 IOMMU

- 在 BIOS 里找到 IOMMU 设置并启用（AMD 为 AMD-Vi）
- 在`/etc/default/grub`的`GRUB_CMDLINE_LINUX`中加入 iommu 参数，如以下示例：

```shell
...
GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt pcie_aspm=off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1"

# amd_iommu=on: 这个参数启用了AMD主板上的IOMMU（输入/输出内存管理单元）功能。IOMMU是一种硬件功能，它可以提供更好的设备虚拟化和安全性，尤其是在使用虚拟机或者通过VFIO框架将物理设备直通给虚拟机时。
# iommu=pt: 这个参数设置了IOMMU的工作模式为"passthrough"（直通模式）。在直通模式下，物理设备可以直接访问虚拟机，绕过主机操作系统的干预。
# pcie_aspm=off: 这个参数禁用了PCI Express的节能特性（Active State Power Management）。禁用ASPM可以避免某些硬件兼容性问题和性能问题。
# vfio_iommu_type1.allow_unsafe_interrupts=1: 这个参数允许VFIO在IOMMU上使用不安全的中断处理方式。通常，IOMMU会限制设备对中断的访问，以提高系统的安全性。但是，某些设备可能需要直接访问中断，这个参数允许VFIO绕过IOMMU的限制。
# kvm.ignore_msrs=1: 这个参数告诉KVM（内核虚拟机模块）忽略对模型特定寄存器（Model Specific Registers）的访问。这些寄存器可能与虚拟化有关，通过忽略对它们的访问，可以解决某些虚拟化环境中的性能问题。
...
```

- 更新 grub2

```shell
sudo update-grub2
```

- 重启系统，使用`dmesg`检查 IOMMU 是否已正常启用：

```shell
sudo dmesg | grep -i -e DMAR -e IOMMU
```

```
# 示例输出
[    0.000000] Command line: BOOT_IMAGE=/vmlinuz-5.19.0-42-generic root=UUID=d8554c31-cdad-427f-87dd-5691f200ba60 ro quiet splash amd_iommu=on iommu=pt vfio-pci.ids=10de:1f15,10de:10f9,10de:1ada,10de:1adb pcie_aspm=off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 vt.handoff=7
[    0.060745] Kernel command line: BOOT_IMAGE=/vmlinuz-5.19.0-42-generic root=UUID=d8554c31-cdad-427f-87dd-5691f200ba60 ro quiet splash amd_iommu=on iommu=pt vfio-pci.ids=10de:1f15,10de:10f9,10de:1ada,10de:1adb pcie_aspm=off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 vt.handoff=7
[    0.355288] iommu: Default domain type: Passthrough (set via kernel command line)
[    0.379043] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.379100] pci 0000:00:01.0: Adding to iommu group 0
[    0.379108] pci 0000:00:01.1: Adding to iommu group 1
...
...
[    0.379399] pci 0000:07:00.0: Adding to iommu group 7
[    0.379402] pci 0000:07:00.1: Adding to iommu group 7
[    0.379958] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
[    0.383987] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
[    2.435032] AMD-Vi: AMD IOMMUv2 loaded and initialized

```

#### 确认 IOMMU 组有效

- 新建以下 shell 脚本并执行：

```shell
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

```
# 示例输出
IOMMU Group 0:
        00:01.0 Host bridge [0600]: Advanced Micro Devices, Inc. [AMD] Renoir PCIe Dummy Host Bridge [1022:1632]
...
...
IOMMU Group 10:
        01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106M [GeForce RTX 2060 Mobile] [10de:1f15] (rev a1)
        01:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:10f9] (rev a1)
        01:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ada] (rev a1)
        01:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1adb] (rev a1)
IOMMU Group 11:
        02:00.0 Ethernet controller [0200]: Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller [10ec:8168] (rev 15)
IOMMU Group 12:
        03:00.0 Network controller [0280]: Intel Corporation Wi-Fi 6 AX200 [8086:2723] (rev 1a)
...
...
IOMMU Group 14:
        05:00.0 Non-Volatile memory controller [0108]: Sandisk Corp WD Blue SN550 NVMe SSD [15b7:5009] (rev 01)

```

IOMMU 组是可以传递给虚拟机的最小一组物理设备。

例如，在上面的示例中，01:00.0 中的 GPU、01:00.1 中的音频控制器、01:00.2 中的 USB 控制器和 01:00.3 中的串行总线控制器都属于 IOMMU 组 10，只能一起直通（Passthrough）。  
而 AX200 无线网卡有自己的组（组 12），与有线网卡（组 11）之类的分开，这意味着它们中的任何一个都可以传递给虚拟机而不影响其他设备。

如果你的电脑不是笔记本且你的显卡所在的组里出现了非预期设备，可能是因为你的显卡插在了基于 CPU 的 PCI-E 插槽里，而你的 CPU 犯了些错误导致隔离乱了。给显卡换个插槽试试。

### 隔离 GPU

#### 通过设备 ID 绑定 VFIO 框架

- 根据之前的输出找到需要隔离的设备的设备 ID，比如以下输出中，`10de:1f15` `10de:10f9` `10de:1ada` `10de:1adb`这四个设备 ID。

```
...
IOMMU Group 10:
        01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106M [GeForce RTX 2060 Mobile] [10de:1f15] (rev a1)
        01:00.1 Audio device [0403]: NVIDIA Corporation TU106 High Definition Audio Controller [10de:10f9] (rev a1)
        01:00.2 USB controller [0c03]: NVIDIA Corporation TU106 USB 3.1 Host Controller [10de:1ada] (rev a1)
        01:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU106 USB Type-C UCSI Controller [10de:1adb] (rev a1)
...
```

- 在`/etc/default/grub`的`GRUB_CMDLINE_LINUX`中继续加入参数，如以下示例：

```shell
...
GRUB_CMDLINE_LINUX="amd_iommu=on iommu=pt pcie_aspm=off vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 vfio-pci.ids=10de:1f15,10de:10f9,10de:1ada,10de:1adb"

# vfio-pci.ids=10de:1f15,10de:10f9,10de:1ada,10de:1adb: 这个参数指定了要使用VFIO框架进行直通的PCI设备的ID。VFIO是一种用于虚拟化的Linux内核模块，它允许将物理设备直接映射到虚拟机，以提供更高的性能和灵活性。这里列出了几个PCI设备的ID，以逗号分隔，表示这些设备将会被直通给虚拟机。
...
```

- 重启系统，使用`dmesg`检查配置是否正常：

```shell
sudo dmesg | grep -i vfio
```

```
# 示例输出
[    1.077426] VFIO - User Level meta-driver version: 0.3
[    1.077575] vfio-pci 0000:01:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
[    1.077730] vfio_pci: add [10de:1f15[ffffffff:ffffffff]] class 0x000000/00000000
[    1.115524] vfio_pci: add [10de:10f9[ffffffff:ffffffff]] class 0x000000/00000000
[    1.115653] vfio_pci: add [10de:1ada[ffffffff:ffffffff]] class 0x000000/00000000
[    1.115721] vfio_pci: add [10de:1adb[ffffffff:ffffffff]] class 0x000000/00000000
[    2.573878] vfio-pci 0000:01:00.0: optimus capabilities: enabled, status dynamic power, hda bios codec supported
[   16.949713] vfio-pci 0000:01:00.0: vgaarb: changed VGA decodes: olddecodes=io+mem,decodes=io+mem:owns=none
```

貌似这里没有显示出全部设备也没关系，甚至预期设备（比如 GPU）没显示都没事（至少 arch wiki 是这么说的）

你可以使用`lspci`检查一下几个设备的驱动是不是`vfio-pci`，比如：

```shell
sudo lspci -nnk -d 10de:1f15
```

```
# 示例输出
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU106M [GeForce RTX 2060 Mobile] [10de:1f15] (rev a1)
        Subsystem: ASUSTeK Computer Inc. TU106M [GeForce RTX 2060 Mobile] [1043:1e21]
        Kernel driver in use: vfio-pci
        Kernel modules: nvidiafb, nouveau
```

如果这里驱动不对，你可以尝试禁用冲突的驱动。通过编辑`/etc/modprobe.d/blacklist.conf`向其中加入条目即可禁用驱动。例如：

```
...
# block Nvidia driver
blacklist nouveau
...
```

### 创建虚拟机

- 安装`virt-manager` `qemu-kvm`

```shell
sudo apt install virt-manager qemu-kvm
```

- 打开`虚拟系统管理器`-`编辑`-`Preferences`-`常规`-`启用 XML 编辑`，为之后直接编辑 XML 配置做准备。

- 正常创建一个虚拟机，安装系统，安装`virtio-win`驱动，关闭虚拟机。
  - 注意事项：
    1. 至少留俩核心给宿主机
    2. 勾选`在安装前自定义配置`，以进一步配置
    3. 芯片组`Q35`，固件为`UEFI x86_64: /usr/share/OVMF/OVMF_CODE_4M.fd`，不需要另外那俩。
    4. CPU 手动设置拓扑，套接字 1，线程 2，核心越多越好。`复制主机 CPU 配置`对于 AMD CPU 来说可以不修改，对于 Intel CPU 则建议改成 host-model
    5. 如果你要装 Windows 11，记得添加一个`TPM`，不然过不了要求检查。
    6. 磁盘总线建议使用`virtio`，性能更好。  
       不过需要额外驱动，先前往 [virtio-win direct-downloads full archive](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/ "virtio-win direct-downloads full archive") 下载一份`virtio-win ISO`，与 Windows 安装介质一起挂载入虚拟机，在安装系统过程中选择磁盘时加载驱动即可。
    7. 网卡也可以用`virtio`，参考上一条。
    8. 进入系统之后，再完整安装一遍`virtio-win`驱动。

### 添加 PCI-E 直通设备，并安装驱动

可以直接通过`虚拟系统管理器`添加直通设备，但最好做一些调整。

- 在`添加新虚拟硬件`对话框中选择`PCI 主机设备`，将上面已隔离的设备添加进虚拟机中。
- 在虚拟机详情中选中刚添加的 PCI 主机设备，其名称应类似于`PCI 0000:01:00.0`。检查每个新增设备的 XML，修改其 address 为对应值，修改的规则为，若设备间的`source`的`bus`值相等，其虚拟机内的`bus`也应相同。`slot`值同理。`function`值则应与其`source`值相同。如下例：

```xml
<!-- 修改前 -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x2"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x09" slot="0x00" function="0x0"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x0a" function="0x3"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x0a" slot="0x00" function="0x0"/>
</hostdev>
```

```xml
<!-- 修改后 -->
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0" multifunction="on"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x1"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x2"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x2"/>
</hostdev>
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x3"/>
  </source>
  <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x3"/>
</hostdev>
```

- 开启虚拟机，正常安装最新的显卡驱动，检查设备是否正常运行。
- 将虚拟显卡的型号更改为`virtio`，将`查看`-`缩放显示`-`自动调整虚拟系统窗口大小`关闭。  
  因为在安装完成驱动后，默认的`QXL`显卡将不能操控虚拟机内。

### 添加低延迟画面

#### 打开虚拟机详情，选择`概览`的`XML`，在`<devices>...</devices>`里添加类似以下内容，以添加用于画面传输的共享内存：

```xml
<shmem name="looking-glass">
  <model type="ivshmem-plain"/>
  <!-- 共享内存大小，其大小参考下面的计算方法 -->
  <size unit="M">32</size>
</shmem>
```

共享内存大小计算：

```
宽 x 高 x 像素大小 x 2 = 帧字节
帧字节 / 1024 / 1024 = 帧MiB
帧MiB + 10MiB = 总MiB
```

其中，像素大小在 SDR 时为 4，在 HDR 时为 8。  
但不建议使用 HDR，因为貌似目前 Xorg 与 Wayland 都没有对 HDR 进行支持，HDR 内容仍旧会被转换成 SDR 显示。

比如，屏幕分辨率为 1920x1080（1080p）时：

```
1920 x 1080 x 4 x 2 = 16,588,800 bytes
16,588,800 / 1024 / 1024 = 15.82 MiB
15.82 MiB + 10 MiB = 25.82 MiB
```

最终的共享内存大小必须为大于计算值的最小的 2 的 n 次方，比如上面的计算结果为`25.82 MiB`，则最终结果为`32 MiB`。

#### 预先创建临时文件，以保证权限正确

因为需要宿主机的 client 端能正常访问共享内存，所以需要提前创建其对应的临时文件。共享内存的路径在`/dev/shm/looking-glass`，其中`looking-glass`即为上面配置中的`name`值。  
你可以使用`systemd-tmpfiles`来在虚拟机运行前创建此文件，来使其权限符合要求。

- 创建一个新文件`/etc/tmpfiles.d/10-looking-glass.conf`，在文件中填入以下内容：

```conf
# Type Path              Mode UID  GID Age Argument
f /dev/shm/looking-glass 0660 user kvm -
```

记得把上面的`user`替换成你的用户名。

- 然后使用以下命令创建文件：

```shell
sudo systemd-tmpfiles --create /etc/tmpfiles.d/10-looking-glass.conf
```

#### 虚拟机内安装 host 端

- 虚拟机内前往此网站下载 host 端：[Download Looking Glass](https://looking-glass.io/downloads/ "Download Looking Glass")
- 在虚拟机内正常安装 host 端，其应会自动同时安装 IVSHMEM 驱动。

#### 移除`EvTouch USB 图形数位板`设备，仅保留一套 PS2 键鼠设备即可

#### 宿主机安装 client 端

虽然`apt`能装`looking-glass-client`，但你先别急，有坑，别着急跳。那貌似是老版本，盲目安装会寄掉的。所以我们要按照官方推荐的方法本地编译一个最新的。

- 首先，`git clone`

```shell
git clone --recursive https://github.com/gnif/LookingGlass.git
cd LookingGlass
```

- 安装依赖

```shell
sudo apt install cmake gcc g++ libegl-dev libgl-dev libgles-dev libfontconfig-dev libgmp-dev libspice-protocol-dev make nettle-dev pkg-config libwayland-dev libxkbcommon-dev libxi-dev libxfixes-dev libxinerama-dev libxcursor-dev libxpresent-dev libxss-dev libpipewire-0.3-dev libpulse-dev libsamplerate0-dev binutils-dev
```

- 开始编译并安装

```shell
mkdir client/build
cd client/build
cmake ../
make
sudo make install
```

#### 插入 Type-C 接口的显卡诱骗器

这一步是为了让虚拟机里的显卡有画面输出，强制其使用显卡驱动。其原理与在 Windows 内通过 Type-C 口输出 DP 信号实现激活独显输出同理。否则 Windows 会使用`Microsoft 基本显示适配器`进行画面输出，其不被`Looking Glass`兼容。

#### 检查画面输出是否正常

启动虚拟机，启动宿主机 client 端，测试画面是否正常输出：

```shell
looking-glass-client
```

正常情况下，在进入 Windows 系统前，可能只有画面输出而无法直接通过`looking glass`操作虚拟机，右上角有一个裂开的图标。进入系统后，虚拟机内 host 端正常启动，右上角裂开图标消失，可以正常直接通过`looking glass`操控虚拟机。此时剪切板共享功能也应正常工作。

`Looking Glass`有许多快捷键。你可以按住`Scroll Lock`键检查快捷键。

#### 启用 NVFBC 画面捕捉

`Looking Glass`默认使用`Microsoft DXGI Desktop Duplication`来捕获屏幕画面，这样可用，但有一些`Overlay`并不能被捕捉，比如`Xbox Game Bar`等。此时可以启用`NVFBC`来进行画面捕捉。而众所周知，Nvidia 还尚未对消费级显卡开放`NVFBC`（So Nvidia, Fxxk You），所以我们要先对此功能进行解锁。

- 虚拟机内前往此网站获取`nvfbcwrp64.dll` `nvfbcwrp32.dll`：[nvfbcwrp](https://github.com/keylase/nvidia-patch/tree/master/win/nvfbcwrp/ "nvfbcwrp")
- 备份一份`%WINDIR%\system32\NvFBC64.dll`和`%WINDIR%\SysWOW64\NvFBC.dll`
- 将`%WINDIR%\system32\NvFBC64.dll`重命名为`%WINDIR%\system32\NvFBC64_.dll`
- 将`nvfbcwrp64.dll`重命名并放到`%WINDIR%\system32\NvFBC64.dll`
- 将`%WINDIR%\SysWOW64\NvFBC.dll`重命名为`%WINDIR%\SysWOW64\NvFBC_.dll`
- 将`nvfbcwrp32.dll`重命名并放到`%WINDIR%\SysWOW64\NvFBC.dll`
- 前往`Looking Glass`的 host 端安装目录，一般在`C:\Program Files\Looking Glass (host)`。在此处新建一个`looking-glass-host.ini`，并填入以下内容：  
  （可能会碰到权限问题，也可以在别处新建填好之后再挪进去）

```ini
[app]
capture=nvfbc
```

- 重启，检查 Overlay 程序是否正常显示。

### 添加低延迟输入

#### 打开虚拟机详情，选择`概览`的`XML`，在`<devices>...</devices>`里添加类似以下内容，以添加低延迟输入：

```xml
<!-- 鼠标，dev为鼠标事件的路径，一般在/dev/input/by-id/目录下，找不到可以去/dev/input/by-path/目录下找找 -->
<input type="evdev">
  <source dev="/dev/input/by-id/usb-Logitech_USB_Receiver-if02-event-mouse"/>
</input>
<!-- 键盘，dev同上，grabToggle为将输入在虚拟机内外之间切换的快捷键 -->
<input type="evdev">
  <source dev="/dev/input/by-id/usb-RONGYUAN_2.4G_Wireless_Device-if02-event-kbd" grab="all" grabToggle="ctrl-scrolllock" repeat="on"/>
</input>
```

- 十分建议把 Windows 的鼠标加速关闭。

### 添加低延迟音频

低延迟音频使用 Scream 虚拟声卡通过本地网络进行传输。

#### 虚拟机内添加虚拟声卡

- 移除虚拟机默认虚拟声卡，一般是`ich9`
- 虚拟机内前往此网站获取`Scream`：[Scream Releases](https://github.com/duncanthrax/scream/releases/ "Scream Releases")
- 虚拟机内安装（记得管理员权限安装）

#### 宿主机安装 Receiver 端

Receiver 也需要自己编译，诶嘿（

- 首先，`git clone`

```shell
git clone --recursive https://github.com/duncanthrax/scream.git
cd scream/Receivers/unix
```

- 安装 Pulseaudio 依赖

```shell
sudo apt install libpulse-dev
```

- 开始编译并安装

```shell
mkdir build && cd build
cmake ..
make
sudo make install
```

- 测试是否正常

```shell
# virbr0 为虚拟机网卡接口，一般没改过网络的话都是 virbr0
scream -i virbr0
# 然后在虚拟机里弄点声音看是否正常输出声音
```

### 虚拟机声音与画面同时启停

#### 创建 Scream 服务

- 新建文件`~/.config/systemd/user/scream.service`，填入以下内容（记得按需修改网卡接口）：

```service
[Unit]
Description=A Scream audio receiver using Pulseaudio, ALSA or stdout as audio output
After=network-online.target
After=pulseaudio.service

[Service]
Type=simple
ExecStart=/usr/local/bin/scream -i virbr0
```

- 重新加载当前用户的 systemd 用户会话管理器的配置文件。

```shell
systemctl --user daemon-reload
```

#### 创建 Looking Glass 启动快捷方式

- 下载图标文件并放在`~/.local/share/icons/looking-glass.png`：[icon-128x128.png](https://raw.githubusercontent.com/gnif/LookingGlass/master/resources/icon-128x128.png "icon-128x128.png")
- 创建文件`~/.local/share/applications/LookingGlass.desktop`，并填入类似以下内容：

```desktop
[Desktop Entry]
Name=Looking Glass
Comment=A client application for accessing the LookingGlass IVSHMEM device of a VM
Categories=Network;RemoteAccess;
Exec=sh -c "systemctl --user restart scream.service;looking-glass-client;wait;systemctl --user stop scream.service"
Icon=looking-glass
Terminal=false
Type=Application
```

此时你应该可以在应用列表里找到`Looking Glass`了，测试一下画面与声音是否可以正常使用。

### 添加共享文件系统

有时需要在宿主机和虚拟机内共享文件，可以给虚拟机添加共享文件系统达成此目的。（但实测并不太能直接拿来打游戏，运行程序请先复制到虚拟机内再运行）

- 给虚拟机添加新虚拟硬件`文件系统`，驱动选择`virtiofs`，`源路径`即源路径，`目标路径`即将被映射到虚拟机内的硬盘名称。
- 启动虚拟机，假设你之前已经正常完整安装过一遍`virtio-win`驱动。
- 打开`计算机管理`-`服务和应用程序`-`服务`，将`VirtIO-FS Service`的`启动类型`修改为`自动(延迟启动)`，并手动启动一次此服务。
- 检查`此电脑`中是否已出现从`Z:`盘符开始向下排序的新磁盘，即共享文件系统。  
  注意：有一定几率出现在重启虚拟机后共享文件系统未生效的情况，此时手动再去启动一下`VirtIO-FS Service`即可。

### 至此，教程结束，完结撒花（

## 参考资料

- [PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF "PCI passthrough via OVMF")
- [Looking Glass B6 documentation](https://looking-glass.io/docs/B6/ "Looking Glass B6 documentation")
- [duncanthrax/scream](https://github.com/duncanthrax/scream "duncanthrax/scream")
- [KVM显卡直通进阶配置（looking-glass、scream等）](https://zhuanlan.zhihu.com/p/428845384 "KVM显卡直通进阶配置（looking-glass、scream等）")
- [Optimus MUXed 笔记本上的 NVIDIA 虚拟机显卡直通](https://lantian.pub/article/modify-computer/laptop-muxed-nvidia-passthrough.lantian/ "Optimus MUXed 笔记本上的 NVIDIA 虚拟机显卡直通")
- [笔记本 Optimus MUXless 下的 Intel 和 NVIDIA 虚拟机显卡直通](https://lantian.pub/article/modify-computer/laptop-intel-nvidia-optimus-passthrough.lantian/ "笔记本 Optimus MUXless 下的 Intel 和 NVIDIA 虚拟机显卡直通")
