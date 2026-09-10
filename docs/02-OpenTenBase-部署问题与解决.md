# OpenTenBase 部署问题与解决记录（CentOS 7 + VMware）

> 记录时间：2026-09-10
> 环境：VMware Workstation 16.2.4 + CentOS 7.9.2009 Minimal + OpenTenBase v5.0（内核 PostgreSQL 10.0 / V5.21.8）
> 结果：单机分布式集群（1 GTM + 1 CN + 2 DN）部署成功，建库 / 建分布表 / 写入 / 聚合查询全部验证通过
>
> 前置篇见 [《CentOS 7 安装问题与解决》](./01-CentOS7-安装问题与解决.md) —— 装系统、配网络、换 yum 源那一段踩的坑。

## 目录

| 章节 | 内容 |
|---|---|
| [一、最终结果](#一最终结果) | 集群拓扑、IP、验证结论 |
| [二、环境信息](#二环境信息) | 宿主机配置、软件路径 |
| [三、虚拟机配置要点](#三虚拟机配置要点) | vmx 关键配置、磁盘创建命令 |
| [四、踩坑清单](#四踩坑清单18-个按出现顺序) | 19 个坑，每个含现象 / 根因 / 解决 |
| [五、隐藏依赖](#五官方文档里没写的隐藏依赖重点) | 官方文档漏写的 7 个依赖 |
| [六、关键路径速查](#六关键路径速查) | 目录、端口分配 |
| [七、如果要重做一遍](#七如果要重做一遍) | 8 步复现清单 |
| [八、调试技巧](#八调试技巧可复用) | vmrun、共享目录、安装器排障等方法 |

**19 个坑速查：**

1. [CentOS 7 已 EOL，yum 源全部失效](#坑-1centos-7-已-eolyum-源全部失效)
2. [「安装 VMware Tools」抢走光驱 → `/dev/root does not exist`](#坑-2安装-vmware-tools抢走光驱--devroot-does-not-exist)
3. [安装程序误点「退出」→ 磁盘零写入，BIOS 跑去 PXE](#坑-3安装程序误点退出-磁盘零写入bios-跑去-pxe)
4. [串口日志文件已存在，VMware 弹窗阻塞启动](#坑-4串口日志文件已存在vmware-弹窗阻塞启动)
5. [**重打包 ISO 文件名被截断，anaconda 读不到 repodata**](#坑-5核心重新打包-iso-时文件名被截断anaconda-读不到-repodata)
6. [`%post` 在 chroot 里看不到安装器的光驱挂载点](#坑-6post-在-chroot-里看不到安装器的光驱挂载点)
7. [zstd 缺失且路径写死](#坑-7zstd-库缺失且-configure-写死了-usrlocalliblibzstda)
8. [lz4 同样缺失、同样写死](#坑-8lz4-同样缺失同样写死-usrlocallibliblz4a)
9. [gcc 4.8 默认 C89，pgvector 编译失败](#坑-9gcc-48-默认-c89pgvector-编译失败)
10. [opentenbase_ctl 是 C++17，系统没有 g++](#坑-10opentenbase_ctl-是-c17-程序系统没有-g)
11. [缺 CLI11 头文件](#坑-11缺-cli11-头文件)
12. [缺 indicators 头文件](#坑-12缺-indicators-头文件)
13. [缺 libpqxx（构建还要 python3）](#坑-13缺-libpqxx而且它自己构建需要-python3)
14. [ssh 每次连接 84 秒 + `ss` 找不到](#坑-14ssh-每次连接要-84-秒--ss-命令找不到)
15. [两次部署重叠，节点状态变脏](#坑-15两次部署重叠节点状态变脏)
16. [**opentenbase_ctl 不写 forward 端口（最终 Boss）**](#坑-16最隐蔽最终-bossopentenbase_ctl-安装时不写-forward-端口)
17. [.bat 里的中文不能存成 UTF-8](#坑-17写-windows-快捷方式时bat-里的中文不能存成-utf-8)
18. [.bat 必须用 CRLF 换行](#坑-18bat-必须用-crlf-换行lf-会让-goto标签错乱)
19. [systemd unit 里内联 `bash -c` 引号被吃掉](#坑-19systemd-unit-里内联-bash--c-引号被吃掉)

> 目录里的跳转链接在 GitHub / VS Code / Typora 里都能用；如果某个链接点不动，说明当前编辑器生成的锚点规则不同，按章节号找即可。

---

## 一、最终结果

| 组件 | 值 |
|---|---|
| 虚拟机 | `OTB-CentOS7`，4 vCPU / 8 GB 内存 / 60 GB 磁盘 |
| 操作系统 | CentOS Linux 7.9.2009 (Core)，内核 3.10.0-1160.el7.x86_64 |
| IP | `192.168.105.10`（VMware NAT，静态） |
| 实例名 | `otb01` |
| 集群拓扑 | gtm0001 + cn0001 + dn0001 + dn0002（4/4 Running） |
| 数据验证 | `foo` 表两条数据可查，聚合查询 `count/min/max` 正常 |

---

## 二、环境信息

**宿主机**

- CPU：Intel Core i7-14650HX（16 核 / 24 线程）
- 内存：31.6 GB
- 磁盘：KIOXIA SSD 954 GB（C/D/E 同一块盘，E 盘放虚拟机）
- 已开启 VBS / Hyper-V（VMware 走 WHP 兼容模式，可用但略慢）

**软件路径**

| 项目 | 路径 |
|---|---|
| VMware 本体 | `E:\VMware\`（含 `vmrun.exe`、`vmware-vdiskmanager.exe`、`mkisofs.exe`、`7za.exe`） |
| 虚拟机目录 | `E:\VMware\OTB-CentOS7\` |
| 原版 ISO | `E:\VMware\ISO\CentOS-7-x86_64-Minimal-2009.iso` |
| 自动安装盘 | `E:\VMware\ISO\CentOS7-OTB-auto.iso`（自建，含 kickstart） |
| kickstart 源文件 | `E:\VMware\OTB-CentOS7\scripts\ks.cfg`（从打包目录里备份出来的） |
| 引导配置备份 | `E:\VMware\OTB-CentOS7\scripts\isolinux.cfg` |
| 共享文件夹 | `E:\VMware\OTB-CentOS7\share\` ↔ 虚拟机 `/mnt/hgfs/` |
| 部署脚本 | `E:\VMware\OTB-CentOS7\scripts\` |

---

## 三、虚拟机配置要点

`OTB-CentOS7.vmx` 里几个关键配置（手工写的，不是向导生成的）：

```
guestOS = "centos7-64"
firmware = "bios"
memsize = "8192"
numvcpus = "4"
cpuid.coresPerSocket = "2"
bios.bootOrder = "hdd,cdrom"        # 硬盘优先，装完自动从硬盘启动
scsi0.virtualDev = "lsilogic"
ide1:0.deviceType = "cdrom-image"
ethernet0.connectionType = "nat"
ethernet0.virtualDev = "e1000"
isolation.tools.hgfs.disable = "FALSE"   # 开启共享文件夹
```

磁盘用 `vmware-vdiskmanager.exe` 创建：

```powershell
E:\VMware\vmware-vdiskmanager.exe -c -s 60GB -a lsilogic -t 1 "E:\VMware\OTB-CentOS7\OTB-CentOS7.vmdk"
```

> 注意：生成的是 `twoGbMaxExtentSparse` 类型，会拆成 `OTB-CentOS7-s001.vmdk` ~ `s016.vmdk`。Windows 资源管理器里看到的 `.vmdk` 只是描述文件，虚拟机里的 `/data/...` 目录都在这 16 个分片里，宿主机上无法直接浏览。

---

## 四、踩坑清单（19 个，按出现顺序）

### 坑 1：CentOS 7 已 EOL，yum 源全部失效

**现象**：`yum install` 报 `Could not resolve host: mirrorlist.centos.org`。

**根因**：CentOS 7 于 2024-06 停止维护，官方镜像站下架，`/etc/yum.repos.d/CentOS-Base.repo` 里的 `mirrorlist` 全指向已失效域名。

**解决**：切到清华 vault 镜像，写死版本号 `7.9.2009`（`$releasever` 只有 `7`，对不上 vault 目录）：

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

---

### 坑 2：「安装 VMware Tools」抢走光驱 → `/dev/root does not exist`

**现象**：安装程序走到一半掉进 dracut 紧急 shell，刷屏 `dracut-initqueue timeout`，最后 `Warning: Could not boot. /dev/root does not exist`。

**根因**：虚拟机运行时点了「虚拟机 → 安装 VMware Tools」。VMware 会把虚拟光驱从 CentOS 安装盘换成自带的 `linux.iso` 并断开连接，安装介质半路消失。`vmware.log` 里的铁证：

```
ide1:0.fileName = "E:\VMware\linux.iso"
ide1:0.startConnected = "FALSE"
toolsInstall.origType = "cdrom-image"
toolsInstallManager.lastInstallError = "21004"
```

**解决**：关机重开即可（VMware 会在关机时还原光驱路径），**关键是不再点「安装 VMware Tools」**。CentOS 7 用 `yum install open-vm-tools` 就够。

> 这个报错和 SCSI 驱动无关，把硬盘从 SCSI 换成 SATA 解决不了。
>
> 完整的排查思路和判断顺序见前置篇 [《CentOS 7 安装问题与解决》· 四、一条流传很广的错误解法](./01-CentOS7-安装问题与解决.md#四一条流传很广的错误解法)。

---

### 坑 3：安装程序误点「退出」→ 磁盘零写入，BIOS 跑去 PXE

**现象**：安装界面操作后虚拟机重启，最终停在 `Network boot from Intel E1000 ... DHCP`，VMware 弹「此虚拟机中未安装 CentOS 7 64 位」。

**根因**：CentOS 7 安装界面右下角「**退出**」和「**开始安装**」两个按钮挨在一起，点错会退出安装程序 —— anaconda 退出时会**弹出光盘并硬重启**。日志：

```
CDROM: Guest eject on 'ide1:0'. Disconnecting disc image.
Chipset: The guest has requested that the virtual machine be hard reset.
```

此时磁盘分片文件仍然每个 512 KB（一个字节都没写）= 安装从未真正开始。

**解决**：关机重开（光驱会自动挂回，因为 `ide1:0.startConnected = "TRUE"`），重新走安装流程。正确顺序是：先配好「安装位置」和「网络和主机名」，右下角「开始安装」才会从灰色变成可点。

---

### 坑 4：串口日志文件已存在，VMware 弹窗阻塞启动

**现象**：虚拟机卡在 `serial.log 已存在，要替换还是附加？` 的对话框上，根本没开始引导。

**根因**：给 vmx 配了 `serial0.fileType = "file"`，文件已存在时 VMware 会弹窗询问；而 `msg.autoAnswer = "TRUE"` 被删掉后弹窗没人应答，就一直等。

**解决**：开机前删掉 `serial.log`，或者点「附加」。

---

### 坑 5（核心）：重新打包 ISO 时文件名被截断，anaconda 读不到 repodata

**现象**：自动安装卡在 `Starting automated install` 后面那排省略号，**磁盘 0 写入**，几分钟不推进。

**根因**：用 Windows 的 `robocopy` 从挂载的 ISO 复制文件时，**Windows 只认 Joliet 目录树**。CentOS 7 的 repodata 文件名长达 76 字符（`<sha256>-primary.xml.gz`），Joliet 上限 64 字符，后缀 `-primary.xml.gz` 被截断。重新打包后 Rock Ridge 里的名字也变成截断的，而 `repomd.xml` 引用的是完整名字 → anaconda 永远找不到元数据，无限重试。

安装器日志（通过 `inst.sshd` 进去看到的）：

```
ERR packaging: failed to grab repo metadata for anaconda:
  repodata/136912ae...-primary.xml.gz: [Errno 14] curl#37 - "Couldn't open file"
retrying metadata download for repo anaconda, retrying (8/10)
```

**解决**：改用 Windows 自带的 `tar`（bsdtar / libarchive，支持 Rock Ridge）解包 ISO：

```powershell
tar -xf "E:\VMware\ISO\CentOS-7-x86_64-Minimal-2009.iso" -C "E:\VMware\OTB-CentOS7\iso-stage2"
```

解出来的 repodata 文件名带完整后缀，重新打包即可。

> 注：这两个打包用的临时目录（`iso-stage` / `iso-stage2`，各约 1 GB）在部署完成后已经删除以节省空间，
> 关键文件（`ks.cfg`、`isolinux.cfg`）已备份到 `scripts\`。要重新打包时按上面的命令重新解包一次即可。

打包命令（注意要**先删掉 `isolinux/boot.cat`**，否则 mkisofs 报 `same Rock Ridge name 'boot.cat'`）：

```powershell
E:\VMware\mkisofs.exe -o E:\VMware\ISO\CentOS7-OTB-auto.iso `
  -b isolinux/isolinux.bin -c isolinux/boot.cat `
  -no-emul-boot -boot-load-size 4 -boot-info-table `
  -R -V "CentOS 7 x86_64" "E:\VMware\OTB-CentOS7\iso-stage2"
```

> 卷标 `CentOS 7 x86_64` 必须和原盘一致，因为引导参数是 `inst.stage2=hd:LABEL=CentOS\x207\x20x86_64`。

---

### 坑 6：`%post` 在 chroot 里看不到安装器的光驱挂载点

**现象**：kickstart 的 `%post` 里想从光盘复制源码包，条件判断永远不成立，源码没被解出来，但脚本还是跑完了（状态标记正常写入，容易误判成功）。

**根因**：`%post` 是在新系统的 chroot（`/mnt/sysimage`）里执行的，`/run/install/repo` 是**安装器环境**的挂载点，chroot 里看不到。

**解决**：改走共享文件夹（`/mnt/hgfs/`），或者直接在 `%post` 里 `mount /dev/sr0`。

---

### 坑 7：`zstd` 库缺失，且 configure 写死了 `/usr/local/lib/libzstd.a`

**现象**：

```
checking for ZSTD_compress in -lzstd... no
configure: error: zstd library not found.
```

**根因**：

1. OpenTenBase v5.0 **强制要求 zstd**（没有 `--without-zstd` 开关），但 CentOS 7 的 vault 源里**根本没有 libzstd 包**；
2. 从源码装到 `/usr/lib64` 也没用 —— configure 的检测命令写死了 `/usr/local/lib/libzstd.a`：`gcc -o conftest ... conftest.c /usr/local/lib/libzstd.a`。

**解决**：源码编译 zstd 并装到 `/usr/local`：

```bash
cd /usr/local/src && tar zxf zstd-1.5.2.tar.gz && cd zstd-1.5.2
make -j4
make install PREFIX=/usr LIBDIR=/usr/lib64
make install PREFIX=/usr/local          # configure 要的是这个路径
echo /usr/local/lib > /etc/ld.so.conf.d/zstd-local.conf
ldconfig
```

---

### 坑 8：`lz4` 同样缺失，同样写死 `/usr/local/lib/liblz4.a`

**现象**：`checking for LZ4_compress_default in -llz4... no` / `configure: error: lz4 library not found.`

**解决**：这个源里有现成包，装完做个软链：

```bash
yum -y install lz4-devel lz4-static
ln -sf /usr/lib64/liblz4.a /usr/local/lib/liblz4.a
```

---

### 坑 9：gcc 4.8 默认 C89，pgvector 编译失败

**现象**：contrib 阶段报 `'for' loop initial declarations are only allowed in C99 mode`（`pgvector/src/bitutils.c`）。

**根因**：CentOS 7 的 gcc 4.8 默认 `-std=gnu90`，而代码里写了 `for (uint32 i = 0; ...)` 这种 C99 语法。上游用新版 gcc（默认 gnu11）所以没暴露。

**解决**：给 `src/Makefile.global` 的 CFLAGS 补上 `-std=gnu99`：

```bash
sed -i '/^CFLAGS = /s/$/ -std=gnu99/' /data/opentenbase/OpenTenBase/src/Makefile.global
```

---

### 坑 10：`opentenbase_ctl` 是 C++17 程序，系统没有 g++

**现象**：`g++ -Wall -g -std=c++17 ... -c src/main.cpp` → `make[1]: g++: Command not found`。

**根因**：`opentenbase_ctl` 用 C++17 编写，CentOS 7 自带 gcc 4.8 连 `-std=c++17` 选项都不认，装 `gcc-c++` 也没用。

**解决**：装 SCL 源里的 devtoolset-9（gcc 9.3.1）：

```bash
cat > /etc/yum.repos.d/CentOS-SCLo.repo <<'EOF'
[centos-sclo-rh]
name=CentOS-7.9.2009 - SCLo rh
baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos-vault/7.9.2009/sclo/x86_64/rh/
gpgcheck=0
enabled=1
EOF

yum -y install scl-utils devtoolset-9-gcc devtoolset-9-gcc-c++ devtoolset-9-runtime

# 编译 contrib 时套上 devtoolset 环境
su - opentenbase -c "scl enable devtoolset-9 -- bash /data/opentenbase/build-contrib5.sh"
```

> 编译出来的二进制**运行时不需要** devtoolset 库，实测直接跑正常（链接到系统的 `/lib64/libstdc++.so.6`）。

---

### 坑 11：缺 CLI11 头文件

**现象**：`src/command/command.h:6:10: fatal error: CLI/CLI.hpp: No such file or directory`

**解决**：下载 CLI11 单头文件放到 `/usr/local/include/CLI/CLI.hpp`（Makefile 里已有 `-I/usr/local/include`）：

```bash
mkdir -p /usr/local/include/CLI
curl -L -o /usr/local/include/CLI/CLI.hpp \
  https://gh-proxy.com/https://github.com/CLIUtils/CLI11/releases/download/v2.4.1/CLI11.hpp
```

---

### 坑 12：缺 indicators 头文件

**现象**：`#include <indicators/progress_bar.hpp>` 找不到。

**解决**：

```bash
cd /usr/local/src && tar zxf indicators-2.3.tar.gz
cp -r indicators-2.3/include/indicators /usr/local/include/
```

---

### 坑 13：缺 libpqxx（而且它自己构建需要 python3）

**现象**：`#include <pqxx/pqxx>` 找不到；CentOS 7 vault 源里 `libpqxx` / `libpqxx-devel` **完全没有**。

**解决**：源码编译 libpqxx 6.4.8（自带 configure，不需要新版 cmake）：

```bash
yum -y install python3          # 缺它会在生成 config-internal-compiler.h 时失败
cd /usr/local/src && tar zxf libpqxx-6.4.8.tar.gz && cd libpqxx-6.4.8
export PATH=/opt/rh/devtoolset-9/root/usr/bin:$PATH
./configure --prefix=/usr/local --enable-shared --disable-static --disable-documentation
make -sj4 && make install && ldconfig
```

---

### 坑 14：ssh 每次连接要 84 秒 + `ss` 命令找不到

**现象**：`opentenbase_ctl install` 卡在「轮询端口」不动。进虚拟机手工测同一条命令：

```
real  1m23.801s        ← 一条 ssh 要 84 秒
bash: ss: command not found
```

**根因**（两个叠加）：

1. sshd 默认 `UseDNS yes`，对客户端做反向解析，DNS 超时导致每次连接拖 80 多秒；
2. `ss` 在 `/usr/sbin` 下，普通用户 PATH 里没有，端口探测永远返回失败。

**解决**：

```bash
sed -i 's/^#\?UseDNS.*/UseDNS no/' /etc/ssh/sshd_config
sed -i 's/^#\?GSSAPIAuthentication.*/GSSAPIAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

cat > /data/opentenbase/.ssh/config <<'EOF'
Host *
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    GSSAPIAuthentication no
    PreferredAuthentications password
    ConnectTimeout 10
    LogLevel ERROR
EOF
chown opentenbase:opentenbase /data/opentenbase/.ssh/config
chmod 600 /data/opentenbase/.ssh/config
```

修完后 ssh 从 84 秒降到 **0.34 秒**。

---

### 坑 15：两次部署重叠，节点状态变脏

**现象**：集群状态显示 4/4 Running，但查询报 `57P01 terminating connection due to administrator command`。

**根因**：调试时先后启动了两次部署，前一次的清理动作（`pg_ctl stop`）踢掉了后一次刚拉起的节点，CN 缓存的到 DN 的连接处于 FATAL 状态。

**解决**：做一次干净的 stop + start：

```bash
$PG_HOME/bin/opentenbase_ctl stop  -c /data/opentenbase/opentenbase_config.ini
$PG_HOME/bin/opentenbase_ctl start -c /data/opentenbase/opentenbase_config.ini
```

---

### 坑 16（最隐蔽，最终 Boss）：`opentenbase_ctl` 安装时不写 forward 端口

**现象**：集群 4/4 Running，`CREATE TABLE` / `INSERT` 全部成功，但 **`SELECT`（尤其是聚合查询）100% 报 `57P01`**，而简单查询（`select 1`）却正常。

**定位过程**：

1. DN 日志里出现关键警告：

```
WARNING: the conn is not inited nodename cn0001 host 192.168.105.10 forward port 0
WARNING: because tcp fails to send data, force kill SIGTERM to pid 31694
FATAL: terminating connection due to administrator command
```

2. 查 `pgxc_node` 目录表，**所有节点的 `node_forward_port` 都是 0**，而配置文件里是有值的：

| 节点 | postgresql.conf 里的 port / pooler / forward |
|---|---|
| cn0001 | 11003 / 11004 / **11005** |
| dn0001 | 11006 / 11007 / **11008** |
| dn0002 | 11009 / 11010 / **11011** |

3. DN 的转发进程（源码 `src/backend/forward/fnconn.c`）找不到 CN 的转发端口 → 认定连接未初始化 → 直接 `SIGTERM` 掉后端连接。

**根因**：`opentenbase_ctl` 创建节点时没有把 forward 端口写进目录；而 `ALTER NODE` **只在 CN 生效，不会传播到 DN**（DN 上的 `pgxc_node` 仍然是 0）。

**解决**（必须在 CN 和**每个 DN** 上分别执行）：

```bash
BIN=/data/opentenbase/install/opentenbase/5.21.8/bin
export LD_LIBRARY_PATH=/data/opentenbase/install/opentenbase/5.21.8/lib
for p in 11003 11006 11009; do          # CN / DN1 / DN2
    $BIN/psql -h 127.0.0.1 -p $p -U opentenbase -d postgres -c "alter node cn0001 with (forward=11005);"
    $BIN/psql -h 127.0.0.1 -p $p -U opentenbase -d postgres -c "alter node dn0001 with (forward=11008);"
    $BIN/psql -h 127.0.0.1 -p $p -U opentenbase -d postgres -c "alter node dn0002 with (forward=11011);"
done
```

然后重启集群（让 forward manager 重新加载节点信息）。修复后稳定性测试：聚合查询 10/10 成功，全表查询 10/10 成功。

> 语法细节：`ALTER NODE` 的选项名是 `forward`（不是 `forward_port`），来自 `src/backend/pgxc/nodemgr/nodemgr.c`。

---

### 坑 17：写 Windows 快捷方式时，.bat 里的中文不能存成 UTF-8

**现象**：做「双击连 SSH」的 `连接OpenTenBase.bat` 时，双击后满屏报错：

```
'10' is not recognized as an internal or external command
'用' is not recognized as an internal or external command
'-o' is not recognized as an internal or external command
```

**根因**：cmd.exe 按**当前系统 ANSI 代码页**逐字节解析批处理文件。文件存成 UTF-8 时，中文字符的多字节序列会让解析器错位，把一行的后半截当成新命令执行。

**解决**：.bat 文件必须按系统 ANSI 代码页保存。本机 ACP = `936`（GBK）：

```powershell
# 确认本机 ACP
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage').ACP   # 936

# 按 GBK 写入
$gbk = [System.Text.Encoding]::GetEncoding(936)
[System.IO.File]::WriteAllText("C:\path\to\x.bat", $content, $gbk)
```

同时在批处理开头写 `chcp 936 >nul`（不要写 `chcp 65001`，那和 GBK 文件内容冲突）。

**补充一个同类的坑**：往 .bat 里插入中文**文本行**时，每行前面必须加 `echo `，否则 cmd 会把它当成命令去执行：

```
[提醒] 串行端口调试通道还开着            ← 报错：'[提醒]' 不是内部或外部命令
echo [提醒] 串行端口调试通道还开着        ← 正确
```

改完用「替换掉交互行、只跑回显」的方式做一次冒烟测试（看退出码是不是 0、有没有 `not recognized` 报错）。

---

### 坑 18：.bat 必须用 CRLF 换行，LF 会让 goto/标签错乱

**现象**：编码改对之后仍然报错：

```
'.105.10)' is not recognized as an internal or external command
The system cannot find the drive specified.
'indows' is not recognized as an internal or external command
```

**根因**：文件是**纯 LF 换行**（Unix 风格，由编辑工具写入）。cmd.exe 解析带 `goto` / `:label` 的多行批处理时需要 CRLF，纯 LF 会导致行边界错位、跳转错乱。

**排查方法**：

```powershell
$b = [System.IO.File]::ReadAllBytes($f)
$cr = ($b | Where-Object { $_ -eq 13 }).Count
$lf = ($b | Where-Object { $_ -eq 10 }).Count
"CR=$cr LF=$lf"    # CR=0 就是纯 LF，需要转换
```

**解决**：转成 CRLF

```powershell
$gbk = [System.Text.Encoding]::GetEncoding(936)
$t = $gbk.GetString([System.IO.File]::ReadAllBytes($f))
$t = $t -replace "`r`n", "`n" -replace "`n", "`r`n"
[System.IO.File]::WriteAllText($f, $t, $gbk)
```

> 结论：**给 Windows 写 .bat，就记住「GBK 编码 + CRLF 换行」两条**，缺一个都会出幺蛾子。

---

### 坑 19：systemd unit 里内联 `bash -c '...'` 引号被吃掉

**现象**：配好开机自启后 `systemctl start opentenbase` 正常，但 `systemctl stop opentenbase` 失败：

```
Sep 10 14:06:22 otb-c7 bash[3182]: /bin/bash: -c: option requires an argument
Sep 10 14:06:22 otb-c7 systemd[1]: opentenbase.service: control process exited, code=exited status=2
Sep 10 14:06:22 otb-c7 systemd[1]: Unit opentenbase.service entered failed state.
```

**根因**：unit 文件里写的是

```
ExecStop=/bin/bash -c '$PG_HOME/bin/opentenbase_ctl stop -c /data/opentenbase/opentenbase_config.ini'
```

systemd 对 unit 文件的引号处理有自己的规则，这行的单引号参数没能正确传给 bash，最终变成 `bash -c` 后面没有参数 → 直接报错，**集群其实根本没停**，而 systemd 却把服务标成了 failed。

**解决**：Exec* 里**不要内联 shell 命令**，一律调用独立脚本：

```
ExecStart=/data/opentenbase/start-cluster.sh
ExecStop=/data/opentenbase/stop-cluster.sh
```

脚本内部自己 `export` 所需环境变量，systemd 这边只负责调用。

**配套改进**：

- 两个脚本都加**幂等判断**（`status` 输出里 `Running: 4` 就跳过启动，`Running: 0` 就跳过停止），反复执行安全
- 别名改为走 systemd（`otb-start` → `systemctl restart`、`otb-stop` → `systemctl stop` + 兜底直调停止脚本），**保证 systemd 状态与真实状态永远一致** —— 否则会出现「集群停着但 systemd 认为在跑，`systemctl start` 无效」的坑

**验证**（连续起停两轮）：

| 操作 | systemd | 集群 |
|---|---|---|
| `otb-start` | active | Running: 4 |
| `otb-stop` | inactive | Running: 0, Stopped: 4 |
| `otb-start` | active | Running: 4 |
| `otb-stop` | inactive | 全部停止、端口释放、无残留进程 |

---

## 五、官方文档里没写的隐藏依赖（重点）

OpenTenBase 快速入门只列了这些：

```
git sudo gcc make readline-devel zlib-devel openssl-devel uuid-devel bison flex
cmake postgresql-devel libssh2-devel sshpass libcurl-devel libxml2-devel
```

实际还缺 **7 样**，全部要自己补：

| 依赖 | 用途 | 来源 |
|---|---|---|
| `zstd` | configure 强制要求 | 源码编译，装到 `/usr/local` |
| `lz4-devel` + `lz4-static` | configure 强制要求 | vault base 源 |
| `devtoolset-9` | 编译 C++17 的 opentenbase_ctl | TUNA SCLo 源 |
| `CLI11`（头文件） | opentenbase_ctl 命令行解析 | GitHub，装到 `/usr/local/include/CLI/` |
| `indicators`（头文件） | opentenbase_ctl 进度条 | GitHub，装到 `/usr/local/include/indicators/` |
| `libpqxx` | opentenbase_ctl 的 PG C++ 客户端 | 源码编译 6.4.8 |
| `python3` | libpqxx 构建脚本需要 | vault base 源 |

---

## 六、关键路径速查

**虚拟机内**

| 说明 | 路径 |
|---|---|
| 编译产物（bin/lib/include） | `/data/opentenbase/install/opentenbase_bin_v5.0/` |
| 实例实际运行的二进制 | `/data/opentenbase/install/opentenbase/5.21.8/` |
| 源码 | `/data/opentenbase/OpenTenBase/` |
| 实例数据目录 | `/data/opentenbase/run/instance/otb01/{gtm0001,cn0001,dn0001,dn0002}/data/` |
| 集群配置 | `/data/opentenbase/opentenbase_config.ini` |
| 启动包装脚本 | `/data/opentenbase/start-cluster.sh` |
| systemd 服务 | `/etc/systemd/system/opentenbase.service` |
| 编译日志 | `/data/opentenbase/build.log`、`contrib5.log` |
| 节点运行日志 | 各节点 `data/pg_log/postgresql-*.{csv,log}` |
| 共享文件夹挂载点 | `/mnt/hgfs/` |

**端口分配**

| 节点 | node | pooler | forward |
|---|---|---|---|
| gtm0001 | 11000 | – | – |
| cn0001 | 11003 | 11004 | 11005 |
| dn0001 | 11006 | 11007 | 11008 |
| dn0002 | 11009 | 11010 | 11011 |

---

## 七、如果要重做一遍

1. 建虚拟机（vmdk + vmx），光驱指向自建 kickstart 安装盘
2. 开机自动安装（约 10 分钟），`%post` 自动完成：换源、装官方依赖、建 opentenbase 用户、开 sshd 密码登录
3. 补装 7 个隐藏依赖（见第五节）
4. 编译：`./configure` → `make -sj4` → `make install` → `contrib` 部分用 devtoolset-9
5. 打包并部署集群：`opentenbase_ctl install -c opentenbase_config.ini`
6. **修 forward 端口**（坑 16，不做这一步 SELECT 会 100% 失败）
7. 配 systemd 开机自启
8. 跑体检脚本验收：`bash /mnt/hgfs/full-health-check.sh`，报告见 `/mnt/hgfs/full-check.txt`
   （验收基线和最近一次结果原本记录在本地那份《OpenTenBase-日常使用与凭据.md》里 —— 那份含明文密码，**不随本仓库发布**）

---

## 八、调试技巧（可复用）

**用 `vmrun` 在客户机里执行命令**（需要 open-vm-tools 在运行）：

```powershell
E:\VMware\vmrun.exe -T ws -gu root -gp <密码> runScriptInGuest <vmx> "/bin/bash" "<命令>"
E:\VMware\vmrun.exe -T ws -gu root -gp <密码> copyFileFromGuestToHost <vmx> <客户机路径> <宿主机路径>
```

**让客户机往共享文件夹写结果**，宿主机直接读文件，比来回复制简单得多：

```bash
bash /mnt/hgfs/xxx.sh            # 宿主机把脚本写进 share/，客户机执行
echo hello > /mnt/hgfs/out.txt   # 客户机把结果写回 share/
```

**安装器阶段排障**：引导参数加 `inst.sshd`（配合 `ip=dhcp`），就能 `ssh root@<安装器IP>` 直接进安装环境读 `/tmp/anaconda.log`、`/tmp/packaging.log`。CentOS 7 安装器的 root 默认无密码，可免密登录。

**看 VMware 侧真相**：`vmware.log` 记录了光驱挂载、SCSI 命令、引导顺序、Tools 状态，比猜快得多。

**抓虚拟机画面**：`vmrun captureScreen` 需要客户机登录凭据，纯文本控制台下会失败；此时可以用宿主机截图（`Win+Shift+S` 或截图脚本）。
