# CentOS 7 安装问题与解决记录（VMware Workstation）

> 记录时间：2026-09-10
> 环境：Windows 11 + VMware Workstation 16.2.4 + CentOS 7.9.2009 Minimal（`CentOS-7-x86_64-Minimal-2009.iso`）
> 目标：在虚拟机里装好一台可用于实验的 CentOS 7，并完成基础配置（yum 源、静态 IP、SSH、共享目录）
> 结果：CentOS 7.9.2009 安装成功，静态 IP `192.168.105.10`，SSH 正常，与宿主机共享目录可用

姊妹篇见 [《OpenTenBase 部署问题与解决》](./02-OpenTenBase-部署问题与解决.md) —— 系统装好之后，在它上面部署 OpenTenBase 分布式集群踩的坑。

---

## 目录

| 章节 | 内容 |
|---|---|
| [零、先看这三条](#零先看这三条最省时间的三个坑) | 最容易卡死的三个坑，一眼定位 |
| [一、环境与最终配置](#一环境与最终配置) | 宿主机、虚拟机规格、软件路径 |
| [二、虚拟机怎么建](#二虚拟机怎么建) | vmx 关键参数、磁盘创建命令、分片说明 |
| [三、安装阶段的坑](#三安装阶段的坑按出现顺序) | 6 个坑，含现象 / 根因 / 解决 |
| [四、一条流传很广的错误解法](#四一条流传很广的错误解法) | `/dev/root does not exist` 不等于 SCSI 驱动问题 |
| [五、装完必做的基础配置](#五装完必做的基础配置) | 换源、静态 IP、SSH 提速、共享目录 |
| [六、排障手法](#六排障手法可复用) | 安装器 SSH、日志位置、vmrun |
| [七、附录：宿主机 Windows 侧的两个坑](#七附录宿主机-windows-侧的两个坑) | 写 .bat 时的编码与换行 |
| [八、安装检查清单](#八安装检查清单) | 装完后逐项打勾 |

---

## 零、先看这三条（最省时间的三个坑）

安装过程中真正让人卡住的只有三个，其余都是小问题。遇到卡死先对照这张表：

| 现象 | 真正的原因 | 一句话解法 |
|---|---|---|
| 安装到一半掉进 dracut 紧急 shell，刷屏 `dracut-initqueue timeout`，最后 `Warning: Could not boot. /dev/root does not exist` | 运行中点了 VMware 菜单的「安装 VMware Tools」，虚拟光驱被换成 VMware 自带的 `linux.iso`，安装介质半路消失。**和 SCSI 驱动无关** | 关机重开（光驱路径会自动还原），之后不要再点「安装 VMware Tools」 |
| 装到一半重启后直接进 PXE，屏幕显示 `Network boot from Intel E1000 ... DHCP`，VMware 提示「此虚拟机中未安装 CentOS 7 64 位」 | 点错了安装界面右下角挨在一起的「退出」按钮。anaconda 退出时会弹出光盘并硬重启 | 关机重开、重装，注意按钮别点错（见 [坑 2](#坑-2安装程序误点退出磁盘零写入bios-跑去-pxe)） |
| 自动安装卡在 `Starting automated install` 后面那排省略号，磁盘 0 写入，几分钟不推进 | 重打包 ISO 时 Windows 把 repodata 的超长文件名截断了，anaconda 永远找不到元数据 | 用 Windows 自带的 `tar` 解包 ISO（支持 Rock Ridge），别用 `robocopy`（见 [坑 4](#坑-4核心重打包-iso-时文件名被截断anaconda-读不到-repodata)） |

---

## 一、环境与最终配置

**宿主机**

| 项目 | 值 |
|---|---|
| CPU / 内存 | Intel Core i7-14650HX（16 核 / 24 线程）/ 31.6 GB |
| 磁盘 | KIOXIA SSD 954 GB（C / D / E 同一块盘，虚拟机放 E 盘） |
| 虚拟化 | 已开启 VBS / Hyper-V（VMware 走 WHP 兼容模式，可用但略慢） |

**虚拟机**

| 项目 | 值 |
|---|---|
| 名称 | `OTB-CentOS7` |
| 规格 | 4 vCPU / 8 GB 内存 / 60 GB 磁盘 |
| 系统 | CentOS Linux 7.9.2009 (Core)，内核 `3.10.0-1160.el7.x86_64` |
| 网络 | VMware NAT，静态 IP `192.168.105.10`，主机名 `otb-c7` |
| 安装方式 | 自建 kickstart 自动安装盘（在原版 ISO 上加 `ks.cfg` 重新打包） |
| 实测耗时 | 自动安装约 10 分钟 |

**软件路径（宿主机）**

| 项目 | 路径 |
|---|---|
| VMware 本体 | `E:\VMware\`（含 `vmrun.exe`、`vmware-vdiskmanager.exe`、`mkisofs.exe`、`7za.exe`） |
| 虚拟机目录 | `E:\VMware\OTB-CentOS7\` |
| 原版 ISO | `E:\VMware\ISO\CentOS-7-x86_64-Minimal-2009.iso` |
| 自建自动安装盘 | `E:\VMware\ISO\CentOS7-OTB-auto.iso` |
| kickstart 源文件 | `E:\VMware\OTB-CentOS7\scripts\ks.cfg` |
| 引导配置备份 | `E:\VMware\OTB-CentOS7\scripts\isolinux.cfg` |
| 共享文件夹 | `E:\VMware\OTB-CentOS7\share\` ↔ 虚拟机 `/mnt/hgfs/` |

---

## 二、虚拟机怎么建

### 1. 关键配置项

`OTB-CentOS7.vmx` 里真正影响安装的几个参数（手工写的，不是向导生成的）：

```
guestOS = "centos7-64"
firmware = "bios"
memsize = "8192"
numvcpus = "4"
cpuid.coresPerSocket = "2"
bios.bootOrder = "hdd,cdrom"        # 硬盘优先，装完自动从硬盘启动
scsi0.virtualDev = "lsilogic"
ide1:0.deviceType = "cdrom-image"
ide1:0.startConnected = "TRUE"      # 开机就挂上光驱
ethernet0.connectionType = "nat"
ethernet0.virtualDev = "e1000"
isolation.tools.hgfs.disable = "FALSE"   # 开启共享文件夹
```

几个参数为什么这么设：

- `bios.bootOrder = "hdd,cdrom"`：硬盘优先。装完之后重启不会再从光盘引导，避免反复进入安装界面
- `ide1:0.deviceType = "cdrom-image"` + `startConnected = "TRUE"`：光驱固定挂在 IDE 上并开机自动连接 —— 这也是「误点退出」和「VMware Tools 抢光驱」之后能靠关机重开自愈的原因
- `ethernet0.virtualDev = "e1000"`：CentOS 7 自带 e1000 驱动，免装额外驱动

### 2. 创建磁盘

```powershell
E:\VMware\vmware-vdiskmanager.exe -c -s 60GB -a lsilogic -t 1 "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmdk"
```

> **注意磁盘类型**：`-t 1` 生成的是 `twoGbMaxExtentSparse`，会被拆成 `OTB-CentOS7-s001.vmdk` ~ `s016.vmdk` 共 16 个分片。
> 资源管理器里看到的那个 `.vmdk` 只是描述文件，虚拟机里的 `/data/...` 全都在这 16 个分片里，**宿主机上无法直接浏览**（想读数据只能在虚拟机里操作，或者挂载整盘）。

### 3. 光驱指向安装盘

虚拟机设置 → CD/DVD → 使用 ISO 映像文件 → 选 `CentOS7-OTB-auto.iso`（或原版 ISO），并勾选「启动时连接」。

---

## 三、安装阶段的坑（按出现顺序）

### 坑 1：安装 VMware Tools 抢走光驱 → `/dev/root does not exist`

**现象**

安装程序走到一半，画面下切到 dracut 紧急 shell，刷屏：

```
dracut-initqueue[745]: Warning: dracut-initqueue timeout
dracut-initqueue[745]: Warning: dracut-initqueue timeout - starting timeout scripts
dracut-initqueue[745]: Warning: Could not boot.
Warning: /dev/root does not exist

Generating "/run/initramfs/rdsosreport.txt"
Entering emergency mode. Exit the shell to continue.
dracut:/# _
```

**根因**

虚拟机运行期间点了 VMware 菜单的「虚拟机 → 安装 VMware Tools」。VMware 会把虚拟光驱从 CentOS 安装盘换成自带的 `linux.iso` 并断开连接 —— 安装介质半路消失，安装器找不到根文件系统。

`vmware.log` 里的铁证：

```
ide1:0.fileName = "E:\VMware\linux.iso"
ide1:0.startConnected = "FALSE"
toolsInstall.origType = "cdrom-image"
toolsInstallManager.lastInstallError = "21004"
```

**解决**

1. 虚拟机**关机**再重新开机（VMware 会在关机时把光驱路径还原成原 ISO）
2. 重新走安装流程
3. **关键：之后不要再点「安装 VMware Tools」**。CentOS 7 里执行 `yum install open-vm-tools` 就够了，功能完全够用

> 判断技巧：只要 `vmware.log` 里出现 `toolsInstall.*` 或者 `ide1:0.fileName` 指向 `linux.iso`，就一定是这个原因。

---

### 坑 2：安装程序误点「退出」→ 磁盘零写入，BIOS 跑去 PXE

**现象**

安装界面操作一下之后虚拟机重启，最终停在这一屏：

```
Network boot from Intel E1000
VMware, Inc.
Intel Corporation

CLIENT MAC ADDR: 00 0C 29 B6 62 AB GUID: 564D0508-171D-8AE4-4164-A6296DB662AB
DHCP
```

VMware 同时弹窗：「此虚拟机中未安装 CentOS 7 64 位。请插入安装程序光盘，然后单击『重新启动虚拟机』」。

**根因**

CentOS 7 安装界面**右下角**的「退出」和「开始安装」两个按钮挨在一起。点错点到「退出」时，anaconda 会**弹出光盘并硬重启**虚拟机。此时磁盘上什么也没写，重启后硬盘无引导记录，BIOS 按顺序找到网卡做 PXE 启动。

`vmware.log`：

```
CDROM: Guest eject on 'ide1:0'. Disconnecting disc image.
Chipset: The guest has requested that the virtual machine be hard reset.
```

**磁盘真的一个字节都没写**（16 个分片文件每个都还是 512 KB）—— 说明安装从来没开始过，不是「装了一半坏了」。

**解决**

关机重开，重新安装。安装流程的正确顺序是：

1. 先配「安装位置」（磁盘分区）
2. 再配「网络和主机名」
3. 上面两项配好后，右下角的「开始安装」才会从灰色变成可点
4. 中途需要离开界面，用 `Ctrl+Alt+F2` 切 TTY，**不要点「退出」**

> 顺带一提：`bios.bootOrder = "hdd,cdrom"` 这种情况下，硬盘没系统就会回落到光驱 → 再没有才走 PXE。看到 PXE 就说明**前面两个都没成**。

---

### 坑 3：串口日志文件已存在，VMware 弹窗阻塞启动

**现象**

虚拟机开机后什么都不发生，卡在一个对话框上：

> `serial.log 已存在，要替换还是附加？`

**根因**

为了排障给 vmx 加过串口输出到文件：

```
serial0.present = "TRUE"
serial0.fileType = "file"
serial0.fileName = "E:\VMware\OTB-CentOS7\logs\serial.log"
```

当目标文件**已存在**时，VMware 会弹窗询问「替换 / 附加」。而 `msg.autoAnswer = "TRUE"`（自动应答对话框）被删掉之后，没人回答这个弹窗，虚拟机就一直等着。

**解决**

- 开机前先删掉 `serial.log`，或者
- 在弹出的对话框里点「附加」（日志会往后追加，不影响使用），或者
- vmx 里加上 `msg.autoAnswer = "TRUE"` 让它自动选默认项

---

### 坑 4（核心）：重打包 ISO 时文件名被截断，anaconda 读不到 repodata

**现象**

自动安装卡在 `Starting automated install` 后面那排省略号，磁盘 0 写入，几分钟不推进，看起来像「卡死」。

**根因**

用 Windows 的 `robocopy` 把 ISO 里的文件复制出来时，**Windows 只认 Joliet 目录树**。CentOS 7 的 repodata 文件名长达 76 个字符（`<sha256>-primary.xml.gz`），而 Joliet 的文件名上限是 64 字符 —— 后缀 `-primary.xml.gz` 被截掉了。

重新打包后 Rock Ridge 里存的名字也变成截断版，而 `repodata/repomd.xml` 里引用的仍是完整名字 → anaconda 永远找不到元数据，无限重试。

安装器日志（通过 `inst.sshd` 进安装环境看到的 `/tmp/packaging.log`）：

```
ERR packaging: failed to grab repo metadata for anaconda:
  repodata/136912ae...-primary.xml.gz: [Errno 14] curl#37 - "Couldn't open file"
retrying metadata download for repo anaconda, retrying (8/10)
```

**解决**

改用 Windows 自带的 `tar`（bsdtar / libarchive，支持 Rock Ridge，不会截断文件名）解包：

```powershell
tar -xf "E:\VMware\ISO\CentOS-7-x86_64-Minimal-2009.iso" -C "E:\VMware\OTB-CentOS7\iso-stage2"
```

解出来的 repodata 文件名就带完整后缀了。

**重新打包命令**：

```powershell
E:\VMware\mkisofs.exe -o E:\VMware\ISO\CentOS7-OTB-auto.iso `
  -b isolinux/isolinux.bin -c isolinux/boot.cat `
  -no-emul-boot -boot-load-size 4 -boot-info-table `
  -R -V "CentOS 7 x86_64" "E:\VMware\OTB-CentOS7\iso-stage2"
```

两个坑点：

- 打包前**必须先删掉 `isolinux/boot.cat`**，否则 mkisofs 报 `same Rock Ridge name 'boot.cat'`
- 卷标 `-V "CentOS 7 x86_64"` **必须和原盘一致**，因为引导参数写的是 `inst.stage2=hd:LABEL=CentOS\x207\x20x86_64`，卷标不对就找不到安装源

---

### 坑 5：`%post` 在 chroot 里看不到安装器的光驱挂载点

**现象**

kickstart 的 `%post` 脚本里想从安装光盘复制源码包，条件判断永远不成立，源码没解出来 —— 但脚本还是「跑完了」，状态标记正常写入，**很容易误判成功**。

**根因**

`%post` 是在新系统的 chroot（`/mnt/sysimage`）里执行的，而 `/run/install/repo` 是**安装器环境**的挂载点，chroot 里根本看不到。

**解决**

两条路，任选：

1. 走共享文件夹（`/mnt/hgfs/`）中转 —— 前提是 vmx 里已开 `isolation.tools.hgfs.disable = "FALSE"`，且装 open-vm-tools 时 hgfs 已挂上
2. 在 `%post` 里显式挂载光驱：`mount /dev/sr0 /mnt/cdrom`

> 教训：`%post` 里凡是有「复制文件」「装外部包」的步骤，**都要验证结果**，不能只看脚本退出码。

---

### 坑 6：CentOS 7 已 EOL，yum 源全部失效

**现象**

装完系统第一次 `yum install` 就报：

```
Could not resolve host: mirrorlist.centos.org; Unknown error
```

**根因**

CentOS 7 已于 2024-06 停止维护，官方镜像站下架。`/etc/yum.repos.d/CentOS-Base.repo` 里的 `mirrorlist` 全部指向失效域名。

**解决**

切到清华 vault 镜像，并**写死版本号 `7.9.2009`** —— 因为 `$releasever` 只有 `7`，对不上 vault 下的目录名：

```bash
mkdir -p /etc/yum.repos.d/bak
mv /etc/yum.repos.d/CentOS-*.repo /etc/yum.repos.d/bak/

cat > /etc/yum.repos.d/CentOS-Vault.repo <<'EOF'
[base]
name=CentOS-7.9.2009 - Base
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos-vault/7.9.2009/os/x86_64/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
enabled=1

[updates]
name=CentOS-7.9.2009 - Updates
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos-vault/7.9.2009/updates/x86_64/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
enabled=1

[extras]
name=CentOS-7.9.2009 - Extras
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos-vault/7.9.2009/extras/x86_64/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
enabled=1
EOF

yum clean all && yum makecache
```

> 现在（2026 年）新装 CentOS 7 的话，这一步是**必做**的，不是可选优化。
> 另外 CentOS 7 已于 2024-06-30 彻底 EOL，不再有安全更新 —— 只适合做实验，不要放到公网上。

---

## 四、一条流传很广的错误解法

搜索 `dracut-initqueue timeout` / `/dev/root does not exist` 时，网上最常见的答案是：

> 「把硬盘从 SCSI 改成 SATA，就能根治。」

**这条在本次事故里是错的。**

CentOS 7 的安装盘带 `mptspi` / `megaraid` / `vmw_pvscsi` 等一堆驱动，SCSI（LSI Logic）开箱即用。本次真正的原因是「**光驱被 VMware Tools 抢走**」（见坑 1）—— 把硬盘从 SCSI 换到 SATA 不但没用，还白折腾一次虚拟机配置。

**正确的判断顺序**：

| 检查项 | 怎么查 | 结论 |
|---|---|---|
| 光驱现在挂的是什么？ | `vmware.log` 里的 `ide1:0.fileName` | 指向 `linux.iso` 就是被 Tools 抢了 |
| 有没有 `toolsInstall.*` 记录？ | `vmware.log` 搜索 `toolsInstall` | 有就是同一原因 |
| 磁盘上有没有写进东西？ | 看 `*-s001.vmdk` 等分片文件大小 | 还是 512 KB → 安装从未开始 |
| 引导参数里的 `inst.stage2=` 卷标对不对？ | `isolinux.cfg` + `mkisofs -V` | 卷标不一致 → 找不到安装源 |

排查顺序建议：**先看光驱、再看磁盘写入量、最后才怀疑存储控制器**。

---

## 五、装完必做的基础配置

### 1. 换 yum 源

见 [坑 6](#坑-6centos-7-已-eolyum-源全部失效)。

### 2. 装 open-vm-tools + 共享目录

```bash
yum -y install open-vm-tools
systemctl enable --now vmtoolsd

# 确认共享目录挂上了（vmx 里 isolation.tools.hgfs.disable = "FALSE" 时）
vmware-hgfsclient          # 应该列出宿主机配置的共享名
mount | grep hgfs          # 应该能看到挂载点
```

> **别点 VMware 菜单里的「安装 VMware Tools」**，它会把光驱抢走（坑 1）。

### 3. 配静态 IP（NAT）

虚拟机用 NAT 模式，宿主机在 `VMnet8` 上的地址是 `192.168.105.1`，虚拟机取 `192.168.105.10`。

```bash
# 先确认网卡名（CentOS 7 一般是 ens33）
ip -br link

vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

```ini
TYPE=Ethernet
BOOTPROTO=static
ONBOOT=yes
NAME=ens33
DEVICE=ens33
IPADDR=192.168.105.10
NETMASK=255.255.255.0
GATEWAY=192.168.105.2
DNS1=192.168.105.2
```

```bash
systemctl restart network
ip addr show ens33          # 确认地址生效
ping -c 2 192.168.105.1     # 确认能通宿主机
```

> **网段怎么确定**：宿主机执行 `ipconfig`，看「VMware Network Adapter VMnet8」的 IPv4 地址（本机是 `192.168.105.1`），虚拟机就配同一个网段。
> **网关**：VMware NAT 的网关默认是网段的 `.2`（本机 `192.168.105.2`）。以「虚拟网络编辑器 → VMnet8 → NAT 设置」里显示的为准，不是 `.1` —— `.1` 是宿主机自己。
>
> 配这个的目的：以后 `ssh root@192.168.105.10` 可以直接连，不用每次去 VMware 窗口里敲命令。
>
> 静态 IP 只在你连这个网络的时候有效 —— 它跟宿主机在哪个 WiFi 上无关（NAT 网段是 VMware 自己虚拟出来的），换 WiFi 不影响。

### 4. SSH 提速（84 秒 → 0.3 秒）

新装的 CentOS 7 通过 SSH 连接可能要等 **80 多秒**才出密码提示：

```
real  1m23.801s
```

两个叠加的原因：

1. sshd 默认 `UseDNS yes`，会对客户端 IP 做反向解析，内网没有反向 DNS → 每次都等到超时
2. `GSSAPIAuthentication` 默认开着，也会拖时间

```bash
sed -i 's/^#\?UseDNS.*/UseDNS no/' /etc/ssh/sshd_config
sed -i 's/^#\?GSSAPIAuthentication.*/GSSAPIAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

修完后实测降到 **0.34 秒**。

顺带把虚拟机内互连（比如集群内部的 ssh 调用）也做掉：

```bash
mkdir -p /root/.ssh
cat > /root/.ssh/config <<'EOF'
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    GSSAPIAuthentication no
    ConnectTimeout 10
    LogLevel ERROR
EOF
chmod 600 /root/.ssh/config
```

> `UseDNS no` 这一条尤其重要：**很多「安装/部署脚本卡住不动」最终都是这个原因**（脚本里的 ssh 调用每次要等 84 秒）。

### 5. 关于「控制台不能粘贴」

VMware 控制台是纯文本模式（tty），**没法粘贴** —— 这不是配置问题，而是机制限制：VMware 的剪贴板共享要靠客户机**图形会话**里的 `vmusr` 插件接收，纯文本模式下没有图形会话，也就没有东西接住粘贴的内容。装 `open-vm-tools-desktop` 也一样，除非真的跑起 X。

三种绕过办法：

1. **用 SSH（推荐）**：`ssh root@192.168.105.10`，Windows 终端里 `Ctrl+V` 直接粘贴
2. **共享文件夹中转**：命令写到 `E:\VMware\OTB-CentOS7\share\r.sh`，控制台敲 `bash /mnt/hgfs/r.sh`
3. **做短别名**：把长命令写成 `alias`，控制台敲几个字符就够

---

## 六、排障手法（可复用）

### 1. 安装器阶段就用 SSH 进去看日志

在引导参数里加 `inst.sshd`（配合 `ip=dhcp`），安装器会起 sshd，直接 `ssh root@<安装器IP>` 进安装环境 —— CentOS 7 安装器的 root 默认无密码，可免密登录。

进去之后重点看这两个文件：

| 文件 | 看什么 |
|---|---|
| `/tmp/anaconda.log` | 安装流程走到哪一步、卡在哪 |
| `/tmp/packaging.log` | 元数据下载失败、包依赖冲突 |

> 坑 4 就是靠这条路径拿到 `Couldn't open file ...-primary.xml.gz` 这条关键报错的。

### 2. 看 VMware 侧真相：`vmware.log`

虚拟机目录下的 `vmware.log` 记录光驱挂载、SCSI 命令、引导顺序、Tools 安装状态，**比猜快得多**：

```powershell
Select-String -Path "E:\VMware\OTB-CentOS7\vmware.log" -Pattern "ide1:0|toolsInstall|CDROM|reset" |
  Select-Object -Last 40
```

### 3. 串口日志：等于控制台画面的「文字录像」

vmx 里配好串口输出到文件后：

```
serial0.present = "TRUE"
serial0.fileType = "file"
serial0.fileName = "E:\VMware\OTB-CentOS7\logs\serial.log"
```

虚拟机的内核和安装程序输出会全量写到宿主机这个文件里，**虚拟机卡住、黑屏、起不来的时候用记事本就能看**（比截屏清楚）。

代价：多出一条「虚拟机 → 宿主机文件」的写通道，文件会慢慢变大（实测 110 KB 量级）。

> ⚠️ 在虚拟机里跑不可信程序、或者要把虚拟机当干净环境交付之前，记得关掉它（虚拟机设置 → 选项 → 串行端口，需先关机）。

### 4. 用 `vmrun` 在虚拟机里执行命令（不需要图形界面）

前提：客户机里 open-vm-tools 在运行。

```powershell
# 开机 / 关机
E:\VMware\vmrun.exe -T ws start "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmx" gui
E:\VMware\vmrun.exe -T ws stop  "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmx" soft

# 在客户机里跑命令
E:\VMware\vmrun.exe -T ws -gu root -gp <密码> runScriptInGuest `
  "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmx" "/bin/bash" "<命令>"

# 从客户机取文件
E:\VMware\vmrun.exe -T ws -gu root -gp <密码> copyFileFromGuestToHost `
  "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmx" "/tmp/x.log" "E:\VMware\OTB-CentOS7\x.log"

# 看正在运行的虚拟机
E:\VMware\vmrun.exe -T ws list
```

### 5. 共享文件夹是最省事的传输通道

宿主机把脚本写进 `E:\VMware\OTB-CentOS7\share\`，虚拟机里直接执行：

```bash
bash /mnt/hgfs/xxx.sh                # 执行宿主机放进去的脚本
echo hello > /mnt/hgfs/out.txt       # 把结果写回宿主机
```

**在纯文本控制台上，这是比复制粘贴更靠谱的办法。**

---

## 七、附录：宿主机 Windows 侧的两个坑

给虚拟机做个「双击就连接」的 `.bat` 快捷方式时，连续踩了两个坑。

### 坑 A：.bat 里的中文不能存成 UTF-8

**现象**：双击后满屏报错

```
'10' is not recognized as an internal or external command
'用' is not recognized as an internal or external command
'-o' is not recognized as an internal or external command
```

**根因**：cmd.exe 按**当前系统 ANSI 代码页**逐字节解析批处理文件。文件存成 UTF-8 时，中文字符的多字节序列会让解析器错位，把一行的后半截当成新命令执行。

**解决**：`.bat` 必须按系统 ANSI 代码页保存。本机 ACP = `936`（GBK）：

```powershell
# 确认本机 ACP
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage').ACP   # 936

# 按 GBK 写入
$gbk = [System.Text.Encoding]::GetEncoding(936)
[System.IO.File]::WriteAllText("C:\path\to\x.bat", $content, $gbk)
```

同时在批处理开头写 `chcp 936 >nul`（**不要**写 `chcp 65001`，那和 GBK 文件内容冲突）。

**同类坑**：往 .bat 里插入中文**文本行**时，每行前面必须加 `echo `，否则 cmd 会把它当命令执行：

```
[提醒] 串行端口调试通道还开着            ← 报错：'[提醒]' 不是内部或外部命令
echo [提醒] 串行端口调试通道还开着        ← 正确
```

### 坑 B：.bat 必须用 CRLF 换行

**现象**：编码改对之后仍然报错

```
'.105.10)' is not recognized as an internal or external command
The system cannot find the drive specified.
'indows' is not recognized as an internal or external command
```

**根因**：文件是**纯 LF 换行**（Unix 风格）。cmd.exe 解析带 `goto` / `:label` 的多行批处理时需要 CRLF，纯 LF 会导致行边界错位、跳转错乱。

**排查**：

```powershell
$b = [System.IO.File]::ReadAllBytes($f)
$cr = ($b | Where-Object { $_ -eq 13 }).Count
$lf = ($b | Where-Object { $_ -eq 10 }).Count
"CR=$cr LF=$lf"    # CR=0 就是纯 LF，需要转换
```

**解决**：

```powershell
$gbk = [System.Text.Encoding]::GetEncoding(936)
$t = $gbk.GetString([System.IO.File]::ReadAllBytes($f))
$t = $t -replace "`r`n", "`n" -replace "`n", "`r`n"
[System.IO.File]::WriteAllText($f, $t, $gbk)
```

> **结论：给 Windows 写 .bat，记住「GBK 编码 + CRLF 换行」两条，缺一个都会出幺蛾子。**

---

## 八、安装检查清单

装完之后逐项打勾（这台是这么做的）：

- [ ] `bios.bootOrder = "hdd,cdrom"`，重启后从硬盘启动，不再进安装界面
- [ ] 光驱 `ide1:0.fileName` 指向安装 ISO，**没有**指向 `linux.iso`
- [ ] `cat /etc/redhat-release` → `CentOS Linux release 7.9.2009 (Core)`
- [ ] yum 源指向 vault，`yum makecache` 无报错
- [ ] `open-vm-tools` 已装且 `vmtoolsd` 在跑
- [ ] 共享目录可用：`/mnt/hgfs/` 下能看到宿主机的文件
- [ ] 静态 IP 配好，`ping 192.168.105.1` 能通
- [ ] SSH 秒连（不是 80 多秒）：`UseDNS no` 已生效
- [ ] 免密 ssh config 写好（`/root/.ssh/config`，权限 600）
- [ ] 串口调试通道的状态自己心里有数（要不要关）
- [ ] 显存/内存/磁盘够用：`free -h`、`df -h /`

---

## 参考资料

- CentOS 7 EOL 公告与 vault 源：[清华 TUNA 镜像 · centos-vault](https://mirrors.tuna.tsinghua.edu.cn/centos-vault/)
- 本文档所有日志片段均来自实机 `vmware.log` / `/tmp/packaging.log` / `journalctl` 原始输出
