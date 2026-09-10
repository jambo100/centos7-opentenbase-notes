# CentOS 7 安装 + OpenTenBase 部署踩坑记录

在 VMware Workstation 里从零装 CentOS 7，再在上面部署 OpenTenBase 分布式集群过程中踩到的坑、每个坑的根因，以及验证过的解决办法。

写这份记录的起因很朴素：**同一个坑不想踩第二遍**。所以每条都尽量写到「下次看到什么现象 → 是什么原因 → 敲哪几行」的程度。

## 文档

| 文档 | 内容 | 适合什么时候看 |
|---|---|---|
| [01 · CentOS 7 安装问题与解决](./docs/01-CentOS7-安装问题与解决.md) | 建虚拟机、装系统阶段的 6 个坑 + 装完必做的基础配置（源 / 静态 IP / SSH / 共享目录）+ 排障手法 | 正在装系统、或者装完发现哪儿不对 |
| [02 · OpenTenBase 部署问题与解决](./docs/02-OpenTenBase-部署问题与解决.md) | 编译到集群跑通的全过程 19 个坑，含 7 个官方文档没写的隐藏依赖 | 系统装好了，准备编译部署集群 |

## 环境

| 项目 | 值 |
|---|---|
| 宿主机 | Windows 11，i7-14650HX / 31.6 GB / KIOXIA SSD 954 GB |
| 虚拟机 | VMware Workstation 16.2.4，4 vCPU / 8 GB 内存 / 60 GB 磁盘，NAT 静态 IP |
| 系统 | CentOS Linux 7.9.2009 (Core)，内核 `3.10.0-1160.el7.x86_64` |
| 数据库 | OpenTenBase v5.0（内核 PostgreSQL 10.0 / V5.21.8），1 GTM + 1 CN + 2 DN |
| 结果 | 集群 4/4 Running，建库 / 建分布表 / 写入 2 万行 / 聚合查询全部验证通过 |

## 卡住的话，先看这三条

| 现象 | 真正的原因 |
|---|---|
| `dracut-initqueue timeout` → `Warning: /dev/root does not exist` | 点了 VMware 菜单的「安装 VMware Tools」，光驱被换成 `linux.iso`。**和 SCSI 驱动无关**，改 SATA 没用 |
| 重启后进 PXE（`Network boot from Intel E1000`），磁盘一个字节没写 | 安装界面右下角「退出」和「开始安装」挨在一起，点错会弹盘 + 硬重启 |
| 自动安装卡在 `Starting automated install`，反复重试 metadata | 重打包 ISO 时 Windows 把 repodata 长文件名截断了。用自带 `tar` 解包，别用 `robocopy` |

## 几条最值钱的结论

1. **CentOS 7 已 EOL（2024-06-30）**，新装必须把 yum 源切到 vault 镜像，否则 `yum install` 一步都走不了。
2. **`/dev/root does not exist` 十有八九不是硬盘控制器的问题**，先去 `vmware.log` 看光驱挂的是什么。
3. **`UseDNS no` 必配**：不配的话每次 SSH 要等 80 多秒，很多「部署脚本卡住不动」最后都查到这上面。
4. **VMware 控制台（纯文本模式）无法粘贴**是机制限制而非配置问题，靠 SSH / 共享文件夹 / 短别名绕过。
5. **共享文件夹和串口日志不是网络通道**：拔了虚拟网卡它们照样通。要在虚拟机里跑不可信程序，这两个得单独关。
6. OpenTenBase 官方文档列的依赖**不够用**，实机还缺 7 样（zstd / lz4 / devtoolset-9 / CLI11 / indicators / libpqxx / python3），详见文档 02。

## 说明

- 所有命令、日志片段、报错文本都来自实机，不是从文档里抄的。
- 仓库里**不含任何密码或密钥**。文中涉及凭据的位置一律用 `<密码>` 占位。
- 这套环境是**单机实验环境**：三个数据库节点在同一台虚拟机上，同生共死，不具备容灾能力，别用于生产。
