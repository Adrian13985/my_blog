---
title: "树莓派 3B 私有 NAS 搭建全指南：Samba、Tailscale 与家庭影音"
date: 2026-08-11T16:30:00+08:00
draft: false
tags: ["Raspberry Pi", "NAS", "Samba", "Tailscale", "Linux"]
categories: ["技术教程"]
---

树莓派 3B 虽然性能有限，但很适合承担家庭文件共享、照片备份、轻量级远程访问和本地影音播放的工作。本文以一台主机名为 `Rasnas`、用户名为 `adrian`、系统盘只有 8GB microSD 的树莓派为例，完整记录从硬盘挂载到 Windows、安卓和电视访问的搭建过程。

本文的目标不是构建高性能 RAID 或 4K 转码服务器，而是得到一台稳定、低功耗、可维护的私有 NAS。

## 第一阶段：方案和硬件边界

树莓派 3B 的 USB 与有线网口性能有限，实际文件传输通常在 `8～11 MB/s` 左右。因此，它适合单家庭或少量设备使用，不适合多人高并发访问、实时视频转码或大型 Docker 服务。

建议的结构如下：

```text
树莓派 3B
├── 8GB microSD：只存放 Raspberry Pi OS Lite 系统
├── USB 硬盘：存放 NAS 文件
├── Samba：Windows、macOS、Linux 文件共享
├── Tailscale：安全的外网访问
└── MiniDLNA（可选）：局域网电视影音播放
```

硬件建议：

1. 使用稳定的 5V/2.5A 或更高质量电源。
2. 使用网线连接路由器，避免 NAS 长期仅依赖 Wi-Fi。
3. 机械硬盘建议使用带独立供电的 USB 硬盘盒或硬盘底座。
4. NAS 数据必须放在 USB 硬盘，而不是 8GB SD 卡中。
5. 有条件时准备第二块物理硬盘用于备份。

---

## 第二阶段：写入系统并开启 SSH

下载官方 Raspberry Pi Imager：

[Raspberry Pi Imager](https://www.raspberrypi.com/software/)

在 Imager 中选择：

```text
设备：Raspberry Pi 3
系统：Raspberry Pi OS Lite (64-bit)
存储：8GB microSD
```

在自定义设置中填写：

```text
主机名：Rasnas
用户名：adrian
SSH：启用
时区：Asia/Shanghai
```

8GB SD 卡只适合 Lite 系统。不要安装桌面环境、Nextcloud 或大量 Docker 镜像；它们会快速消耗空间和内存。

Raspberry Pi Connect 不是 NAS 的必需组件。局域网管理使用 SSH，外网访问使用 Tailscale 即可；只有确实需要浏览器远程桌面或网页终端时，才考虑安装 Connect。

启动后，在 Windows PowerShell 中连接：

```powershell
ssh adrian@rasnas.local
```

首次连接会看到主机指纹确认：

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

确认这就是自己的树莓派后输入：

```text
yes
```

如果 `rasnas.local` 无法解析，请到路由器后台查看树莓派 IP，再执行：

```powershell
ssh adrian@192.168.1.50
```

登录后确认身份：

```bash
hostname
whoami
```

更新系统并清理安装缓存：

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt clean
sudo reboot
```

建议在路由器中为树莓派设置 DHCP 静态租约，例如固定为 `192.168.1.50`。固定 IP 会让 Windows 映射网络驱动器和故障排查更可靠。

---

## 第三阶段：初始化并自动挂载 USB 硬盘

插入 USB 硬盘后，先识别设备：

```bash
lsblk -o NAME,SIZE,FSTYPE,MODEL,MOUNTPOINTS
```

通常：

```text
mmcblk0：microSD 系统卡
sda：USB 硬盘
sda1：USB 硬盘的第一个分区
```

**格式化会清空数据。** 只有在确认 `/dev/sda` 是目标硬盘、且其中数据可被清除时，才能执行以下命令。

```bash
sudo fdisk /dev/sda
```

在 `fdisk` 中依次输入：

```text
g
n
1
回车
回车
w
```

将新分区格式化为 ext4：

```bash
sudo mkfs.ext4 -L NASDATA /dev/sda1
```

格式化时出现 `Creating journal` 属于正常过程。此时不要断电、拔硬盘或按 `Ctrl+C`。

本次硬盘创建得到的 UUID：

```text
148d9bac-8c3f-434a-95a4-5caf97c314fe
```

创建挂载目录：

```bash
sudo mkdir -p /srv/nas
```

编辑自动挂载配置：

```bash
sudo nano /etc/fstab
```

添加下面一行：

```text
UUID=148d9bac-8c3f-434a-95a4-5caf97c314fe /srv/nas ext4 defaults,noatime,nofail 0 2
```

其中：

- `noatime` 减少不必要的访问时间写入。
- `nofail` 允许硬盘未连接时系统仍能完成启动。

重新加载并挂载：

```bash
sudo systemctl daemon-reload
sudo mount -a
```

验证结果：

```bash
findmnt /srv/nas
df -h /srv/nas
```

正常输出类似：

```text
/srv/nas /dev/sda1 ext4 rw,noatime
/dev/sda1 458G 2.1M 435G 1% /srv/nas
```

如果 `mount -a` 显示 systemd 仍在使用旧版 `fstab`，运行：

```bash
sudo systemctl daemon-reload
sudo mount -a
```

---

## 第四阶段：创建 NAS 目录和权限

创建数据目录：

```bash
sudo mkdir -p /srv/nas/private
sudo mkdir -p /srv/nas/public
sudo mkdir -p /srv/nas/backup
```

创建共享用户组，并将 `adrian` 加入其中：

```bash
getent group nasusers >/dev/null || sudo groupadd nasusers
sudo usermod -aG nasusers adrian
```

设置目录归属和权限：

```bash
sudo chown -R root:nasusers /srv/nas
sudo chmod 2770 /srv/nas/private
sudo chmod 2775 /srv/nas/public
sudo chmod 2770 /srv/nas/backup
```

权限中的开头 `2` 表示 setgid：在这些目录中新建的文件会自动继承 `nasusers` 组，避免多人协作时出现权限混乱。

用户组变更后，需要重新登录 SSH：

```bash
exit
```

再次连接：

```powershell
ssh adrian@rasnas.local
```

测试写入：

```bash
touch /srv/nas/private/test.txt
ls -l /srv/nas/private
```

---

## 第五阶段：安装和配置 Samba 文件共享

安装 Samba：

```bash
sudo apt update
sudo apt install samba samba-common-bin -y
```

为 `adrian` 设置 Samba 密码：

```bash
sudo smbpasswd -a adrian
```

此密码用于 Windows、安卓或其他 SMB 客户端访问，可以与 SSH 密码相同，也可以不同。

备份默认配置：

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

编辑 Samba 配置：

```bash
sudo nano /etc/samba/smb.conf
```

在文件末尾添加：

```ini
[private]
   path = /srv/nas/private
   browseable = yes
   read only = no
   valid users = @nasusers
   force group = nasusers
   create mask = 0660
   directory mask = 2770

[public]
   path = /srv/nas/public
   browseable = yes
   read only = no
   valid users = @nasusers
   force group = nasusers
   create mask = 0660
   directory mask = 2775
```

注意：`public` 只是共享名称，并不代表匿名公开访问。以上配置中，两个目录均要求 `nasusers` 组中的账户登录。

检查配置：

```bash
testparm
```

出现以下输出表示配置可用：

```text
Loaded services file OK.
Server role: ROLE_STANDALONE
```

`Weak crypto is allowed` 是 Samba 的兼容性提示，并非配置失败。

启动服务并设置开机自启：

```bash
sudo systemctl enable --now smbd
sudo systemctl status smbd --no-pager
```

正常状态应包含：

```text
Active: active (running)
Status: smbd: ready to serve connections...
```

如果本机测试命令不存在：

```bash
smbclient -L localhost -U adrian
```

安装测试客户端：

```bash
sudo apt install smbclient -y
```

然后重新执行测试。输入 Samba 密码后，应能看到 `private` 与 `public`。

---

## 第六阶段：Windows 访问和映射网络驱动器

在 Windows 文件资源管理器地址栏中输入：

```text
\\rasnas\private
```

如主机名解析不稳定，直接使用静态 IP：

```text
\\192.168.1.50\private
```

公共目录：

```text
\\rasnas\public
```

登录时使用：

```text
用户名：adrian
密码：通过 smbpasswd 设置的 Samba 密码
```

验证时应能在 `private` 和 `public` 中创建、读取、修改和删除测试文件。

也可以在“此电脑”中选择“映射网络驱动器”，将：

```text
\\rasnas\private
```

映射为例如 `Z:` 盘，并勾选“登录时重新连接”。

---

## 第七阶段：使用 Tailscale 安全地从外网访问

不要在路由器中转发 Samba 的下列端口：

```text
445
139
```

直接将 SMB 暴露到公网风险很高。正确方式是让访问设备和树莓派加入同一个 Tailscale 网络。

安装 Tailscale：

```bash
sudo apt update
sudo apt install curl -y
curl -fsSL https://tailscale.com/install.sh | sh
sudo systemctl enable --now tailscaled
```

登录并授权设备：

```bash
sudo tailscale up
```

命令会显示一个 URL。使用浏览器打开它，并登录自己的 Tailscale 账号。

查看树莓派的 Tailscale IPv4：

```bash
tailscale ip -4
```

通常会得到：

```text
100.x.x.x
```

查看设备状态：

```bash
tailscale status
```

Windows 客户端下载地址：

[Tailscale for Windows](https://tailscale.com/download/windows)

Windows 登录同一个 Tailscale 账号后，可在任意网络中访问：

```text
\\100.x.x.x\private
```

如果日后启用 UFW 防火墙，记得允许 Tailscale 接口：

```bash
sudo ufw allow in on tailscale0
```

---

## 第八阶段：安卓手机连接 NAS

安卓手机需要：

1. 安装并登录 Tailscale。
2. 安装支持 SMB 的文件管理器或播放器，例如 Solid Explorer、VLC。
3. 使用树莓派的 Tailscale 地址连接。

在文件管理器中添加 SMB 存储时填写：

```text
服务器：100.x.x.x
端口：445（默认）
用户名：adrian
密码：Samba 密码
域：留空或 WORKGROUP
```

在家连接同一 Wi-Fi 时，也可以使用局域网 IP，例如 `192.168.1.50`。但使用 Tailscale IP 可以同时覆盖家庭网络和外网环境。

建议关闭手机 Wi-Fi、仅使用移动数据做一次测试：先确认 Tailscale 已连接，再用文件管理器或 VLC 打开共享目录中的文件。

---

## 第九阶段：在线播放 NAS 视频

树莓派 3B 适合“直放”，即直接把原始视频文件发给支持其编码格式的播放设备；不适合 Jellyfin、Plex 等服务器的实时转码。

为了获得更好的兼容性，视频优先使用：

```text
MP4 + H.264 + AAC
```

### 1. 通过 SMB 播放

只要视频位于 `private` 或其子目录中，Windows、安卓和 VLC 都可以通过 SMB 直接播放：

```text
\\rasnas\private\电影\video.mp4
```

外网中使用 Tailscale 地址：

```text
\\100.x.x.x\private
```

这种方式保留 Samba 登录认证，适合私人视频。

### 2. 通过 MiniDLNA 在电视播放

如果电视支持 DLNA，可以安装轻量的 MiniDLNA。它只负责索引和分发文件，不转码，因此适合树莓派 3B。

创建影音目录：

```bash
sudo mkdir -p /srv/nas/videos
sudo chown root:nasusers /srv/nas/videos
sudo chmod 2770 /srv/nas/videos
```

安装并赋予读取权限：

```bash
sudo apt update
sudo apt install minidlna -y
sudo usermod -aG nasusers minidlna
```

编辑配置：

```bash
sudo nano /etc/minidlna.conf
```

设置：

```ini
media_dir=V,/srv/nas/videos
friendly_name=Rasnas Media
inotify=yes
```

启动服务：

```bash
sudo systemctl enable --now minidlna
sudo systemctl restart minidlna
sudo systemctl status minidlna --no-pager
```

电视与树莓派位于同一局域网时，在电视的“媒体”“网络媒体”“DLNA”或“家庭网络”菜单中寻找：

```text
Rasnas Media
```

MiniDLNA 没有 Samba 用户认证。能发现该 DLNA 服务的局域网设备通常都能播放索引的视频。因此，建议把适合家庭电视播放的内容放入 `/srv/nas/videos`，私密视频仍放在 `/srv/nas/private` 中，并使用 SMB 播放。

如果确实想让电视播放 `private` 下的视频，可以改用：

```ini
media_dir=V,/srv/nas/private/videos
```

但这会使其中的视频对局域网 DLNA 设备可见。

---

## 第十阶段：断电、重启和硬盘更换

### 1. 正确关机

计划断电或拔硬盘时，先执行：

```bash
sudo systemctl stop smbd
sudo poweroff
```

等待树莓派完全关机后，再断开电源或硬盘。

### 2. 意外断电后

重新通电并 SSH 登录后，检查挂载和 Samba：

```bash
findmnt /srv/nas
df -h /srv/nas
sudo systemctl status smbd --no-pager
```

若硬盘没有自动挂载：

```bash
sudo systemctl daemon-reload
sudo mount -a
```

### 3. 文件系统出错时

不要对已经挂载的硬盘执行 `fsck`。先停止 Samba 并卸载：

```bash
sudo systemctl stop smbd
sudo umount /srv/nas
findmnt /srv/nas
```

确认 `findmnt` 没有输出后，再检查：

```bash
sudo fsck -f /dev/sda1
```

修复完成后重新挂载并启动 Samba：

```bash
sudo mount /srv/nas
sudo systemctl start smbd
```

### 4. 更换为新硬盘

硬盘更换前，优先将数据备份到另一块物理硬盘。新硬盘连接后：

1. 使用 `lsblk` 确认新硬盘设备名。
2. 分区并格式化为 ext4。
3. 执行 `sudo blkid /dev/sda1` 获取新 UUID。
4. 在 `/etc/fstab` 中用新 UUID 替换旧 UUID。
5. 执行 `sudo systemctl daemon-reload` 和 `sudo mount -a`。
6. 重新创建目录与权限。
7. 从备份恢复文件。

恢复目录与权限：

```bash
sudo mkdir -p /srv/nas/private /srv/nas/public /srv/nas/backup
sudo chown -R root:nasusers /srv/nas
sudo chmod 2770 /srv/nas/private /srv/nas/backup
sudo chmod 2775 /srv/nas/public
```

Samba 配置通常无需修改，因为共享路径始终是 `/srv/nas/private` 和 `/srv/nas/public`。

---

## 第十一阶段：备份与日常维护

`/srv/nas/backup` 与主数据位于同一块硬盘时，不是真正的备份；硬盘损坏会让它们同时丢失。

推荐使用第二块物理硬盘，假设挂载点为 `/srv/backup`，可执行：

```bash
sudo rsync -aHAX --info=progress2 \
  /srv/nas/private/ \
  /srv/backup/private/
```

备份原则：

```text
3 份数据
2 种存储介质
1 份异地备份
```

日常检查命令：

```bash
df -h
lsblk -f
findmnt /srv/nas
sudo systemctl status smbd --no-pager
tailscale status
```

8GB SD 卡需要定期清理 APT 缓存：

```bash
sudo apt clean
```

---

## 相关网站

- [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- [Raspberry Pi 官方文档](https://www.raspberrypi.com/documentation/)
- [Samba 官方文档](https://www.samba.org/samba/docs/)
- [Tailscale Raspberry Pi 安装文档](https://tailscale.com/kb/1174/install-rpi)
- [Tailscale Windows 客户端](https://tailscale.com/download/windows)
- [OpenMediaVault](https://www.openmediavault.org/)
- [Syncthing 文档](https://docs.syncthing.net/)

---

## 小结

完成上述配置后，`Rasnas` 已具备以下能力：

```text
局域网文件共享：\\rasnas\private 与 \\rasnas\public
Windows 网络驱动器：可映射为本地盘符
安卓访问：Tailscale + SMB 文件管理器或 VLC
外网访问：Tailscale，不需要开放路由器端口
视频播放：SMB 直放；局域网电视可选 MiniDLNA
异常恢复：fstab 自动挂载、ext4 检查和可替换硬盘流程
```

对于树莓派 3B，这是一套性能边界清晰、维护成本较低、隐私可控的家庭私有 NAS 方案。
