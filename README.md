# Gentoo安装笔记
<!-- ArchGu 2024.07.12 -->
Gentoo Linux作为比Arch Linux更难安装的Linux，个人觉得有折腾一下的必要，首先是安装手册：

[Gentoo安装手册](https://wiki.gentoo.org/wiki/Handbook:Main_Page/zh-cn)

- 目前Gentoo官方的23.0已经stable了，而之前的内容是17.0的，更新后有些变化。
- 由于个人偏好，“/”目录使用Btrfs文件系统并使用Timeshift备份，所以在磁盘分配时与Gentoo手册默认的Ext4方式不同，配置为：
```bash
Available profile symlink targets:
  [21]  default/linux/amd64/23.0 (stable)
  [22]  default/linux/amd64/23.0/systemd (stable)
  [23]  default/linux/amd64/23.0/desktop (stable)
  [24]  default/linux/amd64/23.0/desktop/systemd (stable)
  [25]  default/linux/amd64/23.0/desktop/gnome (stable)
  [26]  default/linux/amd64/23.0/desktop/gnome/systemd (stable) *
  [27]  default/linux/amd64/23.0/desktop/plasma (stable)
  [28]  default/linux/amd64/23.0/desktop/plasma/systemd (stable)
  ...
```

## 1.准备阶段
这个阶段主要是:
- 磁盘分区
- 下载Gentoo Stage 3镜像并配置RootFS
- chroot到RootFS(Linux系统遇到故障时可以用类似的方法排除故障)。

**注意点:**
- 比较简单的方法是使用Arch Linux安装脚本中的arch-chroot和genfstab
- 目前默认打包这个安装脚本的LiveCD有Arch Linux，Gentoo Linux
- 其它发行版的LiveCD可以用pip安装archinstall这个python包。

### 1. LiveCD启动
相关教程很多，不详细介绍。
### 2. 分区
不详细介绍,分区的具体方案与Linux用户的个人使用习惯有关，且相关教程也很多。
**使用Timeshift以Btrfs方式备份的前提:**
   - 根目录的文件系统是Btrfs文件系统(使用Btrfs的COW功能实现增量备份)
   - Btrfs分区根目录有名为"@"子卷
   - 启动Linux系统时挂载"@"子卷(/etc/fstab)

这里的分区方法是为了使用Timeshift:
```bash
# -t：指定文件系统类型
# /dev/<...> 分区名
# /mnt/gentoo 挂载点，挂载分区一般挂载到/mnt目录
mount -t btrfs /dev/<根目录分区> /mnt/gentoo
# 创建@子卷
cd /mnt/gentoo &&btrfs subvolume create ./@
# 卸载分区
umount /mnt/gentoo
```
挂载“@”子卷：
```bash
# -o:指定挂载文件系统时的选项,这里指定挂载的子卷为"@"子卷
mount -t btrfs -o subvol=@ /dev/<根目录分区> /mnt/gentoo
```
### 3. 准备RootFS
先选择镜像，Gentoo Stage 3的镜像地址：
```bash
<镜像源>/gentoo/releases/amd64/autobuilds/
```
下载镜像并解压:
```bash
cd /mnt/gentoo
# -O:保存文件
curl -O <Gentoo_Stge_3_FILE_URL>
# -x:解包,-p:保留原始权限,-v:显示详细信息，-f:指定要操作的归档文件
tar -xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```
挂载其它分区：
```bash
mount /dev/<EFI分区> /mnt/gentoo/boot/efi
# 未分HOME分区不用挂载
mount /dev/<Home分区> /mnt/gentoo/home
```

配置编译参数:
```bash
## /mnt/gentoo/etc/portage/make.conf
# 为所有语言设置编译标志
COMMON_FLAGS="-march=native -O2 -pipe"
# 编译线程数，比较好的选择是从 CPU 线程数，或系统 RAM 总量除以 2 GiB 中选择是较小的那个值。
MAKEOPTS="-j4"
# Gentoo Distfiles镜像位于:<镜像源>/gentoo/
GENTOO_MIRRORS="<Gentoo_Distfiles_MIRROR_URL>"
# CHOST,可用gcc-config -l查看
CHOST="x86_64-pc-linux-gnu"
```
先复制镜像源配置的模板文件:
```bash
# 复制模板
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```
配置镜像源：
```bash
## /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
# 安装完成后按需改为git方式同步
sync-type = rsync
# Gentoo Portage镜像源位于<镜像源>/gentoo-portage
sync-uri = <GENTOO_PORTAGE_MIRROR_URL>
```
初始化二进制主机密钥:
```bash
# Gentoo的Binhost于2023年末开始发力
getuto
```
配置Binhost(二进制包主机):
```bash
## /mnt/gentoo/etc/portage/binrepos.conf/gentoobinhost.conf
[gentoobinhost]
# 主机优先级
priority = 9999
# 二进制包镜像位于:<镜像源>/gentoo/releases/amd64/binpackages/
sync-uri = <BINHOST_URL>
```
配置获取二进制包的方式：
```bash
## /mnt/gentoo/etc/portage/make.conf
# getbinpkg 从二进制包主机获取二进制包
# binpkg-request-signature 请求二进制包的签名验证
FEATURES="getbinpkg binpkg-request-signature"
```
复制Live CD的DNS服务器配置文件：
```bash
# --dereference:复制符号链接所指向的实际文件或目录
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```
挂载其他必要的文件系统(使用arch-chroot无需此步骤)：
```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```
然后就可以chroot了(可使用arch-chroot):
```bash
chroot /mnt/gentoo /bin/bash
```
加载环境变量:
```bash
source /etc/profile
# 给shell添加注释以区分chroot环境，非必需
export PS1="(chroot) ${PS1}"
```
## 2.基本系统安装阶段
该阶段主要是:
- 从RootFS开始快速部署一个可启动的系统。

先同步ebuild数据库:
```bash
emerge-webrsync
```
选择合适的配置文件(Gentoo是编译安装，一个包的USE标记改变后需要重新编译，所以Gentoo的配置文件预设了很多常用的配置，桌面使用KDE还是GNOME，init系统使用OpenRC还是SystemD，都要考虑清楚):
```bash
# 列出可用配置文件
eselect profile list
# 选择配置文件
eselect profile set <配置文件编号>
```
可用的配置:
```bash
Available profile symlink targets:
  [21]  default/linux/amd64/23.0 (stable)
  [22]  default/linux/amd64/23.0/systemd (stable)
  [23]  default/linux/amd64/23.0/desktop (stable)
  [24]  default/linux/amd64/23.0/desktop/systemd (stable)
  [25]  default/linux/amd64/23.0/desktop/gnome (stable)
  [26]  default/linux/amd64/23.0/desktop/gnome/systemd (stable)
  [27]  default/linux/amd64/23.0/desktop/plasma (stable)
  [28]  default/linux/amd64/23.0/desktop/plasma/systemd (stable)
```
这里先感性认识一下编译所需时间（Binhost可以节省大量时间，CPU指令集不同的包仍会重新编译，大约有5%的性能优化):
```bash
genlop -t <软件包名>
```
编译时间实例:
```bash
# clang后端
 * sys-devel/llvm

     Thu Oct  5 02:15:28 2023 >>> sys-devel/llvm-15.0.7-r3
       merge time: 1 hour, 49 minutes and 47 seconds.

     Fri Oct  6 18:29:29 2023 >>> sys-devel/llvm-16.0.6
       merge time: 4 hours, 1 minute and 15 seconds.

# 这是Gnome桌面需要的组件，KDE也有类似的
 * net-libs/webkit-gtk

     Thu Oct  5 11:16:36 2023 >>> net-libs/webkit-gtk-2.40.5-r410
       merge time: 3 hours, 30 minutes and 44 seconds.

     Thu Oct  5 15:22:28 2023 >>> net-libs/webkit-gtk-2.40.5-r600
       merge time: 3 hours, 33 minutes and 8 seconds.
# 这个不用介绍了，很有名
 * net-libs/nodejs

     Fri Oct  6 22:50:27 2023 >>> net-libs/nodejs-20.6.1
       merge time: 1 hour, 55 minutes and 39 seconds.
```
配置硬件，语言和引导加载器：
```bash
## /mnt/gentoo/etc/portage/make.conf
# GRUB引导加载器，不添加后面需要重新编译
GRUB_PLATFORMS="efi-64"
# 显卡，这里是AMD核显，其它显卡具体看Wiki中Video Card词条
VIDEO_CARDS="amdgpu radeonsi"
# 输入设备，包括触摸板
INPUT_DEVICES="evdev synaptics"
# 语言
LC_MESSAGES=C.utf8
L10N="en-US zh-CN en zh"
LINGUAS="en-US zh-CN en zh"
# 接受的软件许可证，比如GPL，MIT，不接受会被MASK掉而无法安装
ACCEPT_LICENSE="*"
# 使用稳定分支
ACCEPT_KEYWORDS="amd64"
```
更新@world集合，为接下来的安装做准备：
```bash
# -a:执行操作前询问用户是否继续
# -v:输出详细的信息
# -u:确保安装的是软件包的最新版本
# -D:不仅更新指定的软件包，还更新其依赖的所有软件包
# -N:检查USE标记的变动是否导致需要安装新的软件包或者将现有的包重新编译
# @world:表示要更新系统中的所有软件包的软件集
emerge -avuDN @world
```
配置CPU的指令集（指令集编译优化）:
```bash
# 安装指令集查看工具
emerge -av app-portage/cpuid2cpuflags
# 根据CPU配置编译指令集
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```
配置时区：
```bash
# -s:软链接,-f:如果目标文件存在，强制删除它然后创建新的链接
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```
配置本地化编码生成：
```bash
## /etc/locale.gen
en_US ISO-8859-1
en_US.UTF-8 UTF-8
zh_CN GBK
zh_CN.UTF-8 UTF-8
```
生成本地化编码配置：
```bash
locale-gen
```
列出本地化设置：
```bash
eselect locale list
```
这里有:
```bash
Available targets for the LANG variable:
  [1]  C
  [2]  C.utf8
  [3]  en_US
  [4]  en_US.iso88591
  [5]  en_US.utf8
  [6]  de_DE
  [7]  de_DE.iso88591
  [8]  de_DE.utf8
  [9] POSIX
  [ ]  (free form)
```
设置本地化:
```bash
# 设置为 C.utf8
eselect locale set 2
```
重新加载环境变量：
```bash
env-update && source /etc/profile
# 给shell添加注释以区分chroot环境，非必要
export PS1="(chroot) ${PS1}"
```
安装内核固件：
```bash
emerge -av sys-kernel/linux-firmware
# Intel CPU需要安装微码，AMD CPU在linux-firmware软件包中
emerge -av sys-firmware/intel-microcode
```
以全自动方式安装内核:
**注意点**
- 这里不建议一开始就手动编译内核
```bash
# 全自动安装需添加dracut,grub这两个USE标记
echo "sys-kernel/installkernel dracut grub" > /etc/portage/package.use/installkernel
# 安装内核安装器
emerge -av sys-kernel/installkernel
# 安装二进制内核
emerge -av sys-kernel/gentoo-kernel-bin
```
配置分区挂载：
```bash
# 使用gentoo仓库中的genfstab生成脚本,是archinstall脚本中的一个脚本
emerge -av sys-fs/genfstab
genfstab -u / >> /etc/fstab
```
主机名：
```bash
# 主机名，例子为Linux的吉祥物tux
echo <tux> > /etc/hostname
```
主机名映射：
```bash
## /etc/hosts
# 定义当前系统
127.0.0.1     <tux.homenetwork> <tux> <localhost>

# 可选，定义网络上的其它系统
<192.168.0.5>   <jenny.homenetwork> <jenny>
<192.168.0.6>   <benny.homenetwork> <benny>
```
动态主机配置协议-DHCP:
```bash
emerge -av net-misc/dhcpcd
systemctl enable dhcpcd
```
宽带上网服务-PPPOE:
```bash
emerge -av net-dialup/ppp
```
无线上网服务-WIFI:
```bash
emerge -av net-wireless/iw net-wireless/wpa_supplicant
```
系统网络服务-NetworkManager:
```bash
emerge -av net-misc/networkmanager
systemctl enable NetworkManager
```
根用户密码设置:
```bash
passwd
```
为Linux第一次启动配置:
```bash
# 分配一个随机机器ID
systemd-machine-id-setup
# 提示用户设置区域设置、时区、主机名、root 密码和 root shell值
systemd-firstboot --prompt
# 重置所有已安装工具的文件为预设的策略值
systemctl preset-all --preset-mode=enable-only
```
跨网络同步系统时钟的守护服务:
```bash
systemctl enable systemd-timesyncd.service
```
文件系统支持:
| 文件系统 | 软件包 |
|----| ----|
| XFS | sys-fs/xfsprogs |
| ext4 | sys-fs/e2fsprogs |
| VFAT (FAT32, ...) | sys-fs/dosfstools |
| Btrfs | sys-fs/btrfs-progs |
| ZFS | sys-fs/zfs |
| JFS | sys-fs/jfsutils |
| ReiserFS | sys-fs/reiserfsprogs |
| NTFS | sys-fs/ntfs3g |

引导加载器Grub:
**注意点**
- ubuntu的update-grub其实就是个Shell脚本
- gentoo更新内核后有eclean-kernel工具可以重新生成grub配置，日常也够用
```bash
emerge -av sys-boot/grub
# 如果make.conf未添加GRUB_PLATFORMS="efi-64"，需要添加并使用以下命令重新安装，如果已添加就不需要
emerge -avuN sys-boot/grub
# 主要是--efi-directory=这个参数需要注意
grub-install --target=x86_64-efi --efi-directory=/boot/efi
# 生成配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```
多系统引导:
```bash
# 安装efi引导管理器
emerge -av sys-boot/efibootmgr
# 多系统探测工具
emerge -av sys-boot/os-prober
# 启用多系统探测功能，“>>”为追加内容，">"是覆盖，会导致原文件内容丢失！！！
echo "GRUB_DISABLE_OS_PROBER=false" >> /etc/default/grub
# 生成Grub配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```
添加常用用户:
**注意点**
- 不创建常用用户无法启动图形界面
```bash
# 添加用户，例子为Gentoo的吉祥物larry
# -m:自动创建用户的登录目录（家目录）
# -G:指定用户所属的附加组
useradd -m -G users,wheel,audio -s /bin/bash <larry>
# 设置密码，larry为用户名
passwd <larry>
```
常用提权工具sudo安装及配置:
```bash
emerge -av app-admin/sudo
# 将普通用户加入sudo组以使用sudo，“>>”为追加内容，">"会导致文件内容丢失！！！
echo "%wheel ALL=(ALL:ALL) ALL" >> /etc/sudoers
```
退出chroot环境:
```bash
exit
```
卸载所有分区:
```bash
umount /mnt/gentoo/home
umount /mnt/gentoo/boot/efi
umount /mnt/gentoo
umount ...
```
重启:
```bash
reboot
```
重启后如果可以正常进入gentoo，删除Stage 3安装包：
```bash
sudo rm /stage3-*.tar.*
```
## 3.安装桌面
安装Gnome:
**注意点**
- 如果Terminal无法打开，一般是locale.gen未配置正确）:
```bash
emerge -av gnome-base/gnome
# 更新环境配置
env-update && source /etc/profile
# 开机启动gdm登陆管理器
systemctl enable gdm
# gnome插件安装器
emerge -av gnome-extra/gnome-browser-connector
```
安装KDE:
```bash
# 桌面 基本应用程序 实用应用程序
emerge -av kde-plasma/plasma-meta kde-apps/kdecore-meta kde-apps/kdeutils-meta
# 开机启动sddm登陆管理器
systemctl enable sddm
```
声音问题：
**注意点**
- Linux的声音服务很复杂，需要慢慢排查

如果是驱动问题:
```bash
# 安装驱动
emerge -av sys-firmware/sof-firmware
```
如果使用pulseaudio:
```bash
# 如果是pulseaudio未启动，启动它
pulseaudio --start
```
pulseaudio未开机启动：
```bash
## /etc/pulse/client.conf 
autospawn = yes
```
如果使用pipewire(KDE):
```bash
# 安装必要软件包
emerge -av media-video/pipewire media-video/wireplumber sys-auth/rtkit
# 添加常用用户到pipewire组
# -a:追加
# -G:指定用户所属的附加组
usermod -aG pipewire <larry>
# 将常用用户从audio组退出,为了更好的体验，非必需
# -r:
usermod -rG audio <larry>
# 添加配置文件
cp /usr/share/pipewire/pipewire.conf /etc/pipewire/pipewire.conf
# 添加配置文件
cp /usr/share/pipewire/pipewire.conf ~/.config/pipewire/pipewire.conf
# 为常用用户启用服务，以常用用户运行
systemctl --user enable --now pipewire pipewire-pulse wireplumber
```
Overlay:
```bash
# Overlay仓库管理
emerge -av app-eselect/eselect-repository
# Git版本管理工具
emerge -av dev-vcs/git
# 获取Overlay仓库列表并展示
eselect repository list
# 启用Gentoo用户仓库guru，Gentoo中文用户仓库gentoo-zh，至于pentoo仓库的作用，你猜:)
eselect repository enable guru gentoo-zh pentoo
```
如果想自己编写ebuild，需要搭建本地仓库:
```bash
# 创建本地仓库,仓库名为:localrepo
eselect repository create <localrepo>
```

配置仓库:
```bash
## /var/db/repos/<localrepo>/metadata/layout.conf
masters = gentoo
auto-sync = false
thin-manifests = true
sign-manifests = false
```

还有我大MC:
**注意点**
- MC1.12.2是Java8，Java多版本共存每个发行版的实现方式不同，Gentoo使用Slot

安装Java:
```bash
# 利用slot实现Java多版本共存
emerge -av dev-java/openjdk-bin:8
emerge -av ...
# 列出可用的JVM版本
eselect java-vm list
# 设置系统默认的 JVM
eselect java-vm set system *
# 设置用户默认的 JVM
eselect java-vm set user *
```
**注意点**
- MC在Gentoo无法正常启动，原因是lwjgl这个文件链接到了glibc，Gentoo Wiki也介绍了解决办法:

```bash
emrge -av games-action/prismlauncher
```
配置flatpak:
```bash
flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```
## 4. 优化
配置编译优化:
```bash
## /mnt/gentoo/etc/portage/make.conf
# ccache
FEATURES="ccache"
CCACHE_SIZE="4G"
# binpkg
BINPKG_FORMAT="gpkg"
BINPKG_COMPRESS="xz"
FEATURES="buildpkg"
# 自动清理，笔者未查到该参数
AUTO_CLEAN="yes"
# Global USE flag，这里只是展示一个技巧，全局USE谨慎使用
# AUDIO="alsa jack pulseaudio"
# NET="network networkmanager connection-sharing wifi"
# ELSE="cjk emoji"
# NOUSE="-doc"
# USE="${AUDIO} ${NET} ${ELSE} ${NOUSE}"
```
常用系统软件:
```bash
emerge -av genlop getoolkit
```
主题设置:
```bash
# 非必需
emerge -av lxappearance qt5ct xsettingsd
```
