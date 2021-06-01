## Arch安装过程

#### 联网

###### (无线)

```
iwctl
device list  //看无线网卡名字
station xx(name) scan
station xx(name) get-networks
station xx(name) connect （network name)
```

###### (有线)

```
ipconfig //查看网卡名字
pppoe-setup
输入网卡名字
输入宽带信息
pppoe-start //启动连接
```

##### 验证启动模式是uefi

```
ls /sys/firmware/efi/efivars
```

##### 同步镜像时间时区

```
timedatectl set-ntp true
```

#### 格式化分区并挂载

##### 查看磁盘设备表

```
lsblk
```

##### 删除旧分区

```
gdisk /dev/nvme0n1(大分区名称) 
x //专家模式
z //清除磁盘数据
Y
Y
```

##### 磁盘分区

```
cgdisk /dev/nvme0n1
————————————————————————————————————————————————————
New ->free space //设置启动分区
first sector //default
size in sectors: 1024MiB
Hex code: L ->efi  ->ef00
name: boot
New ->free space  //设置交换空间或交换文件
first sector //default
size in sectors: 16GiB
Hex code: L ->swap ->8200
name: swap
New ->free space  //设置根目录
first sector //default
size in sectors: 64GiB
Hex code //default
name: root
New ->free space //设置个人分区
first sector //default
size in sectors //default
Hex code //default
name: home
——————————————————————————————————————————————————————————————
Write ->yes  ->Quit
```

##### 创建文件系统

```
lsblk      //查看分区情况
mkfs.fat -F32 /dev/nvme0n1p1  //格式化启动分区
mkswap /dev/nvme0n1p2		 //格式化交换空间
swapon /dev/nvme0n1p2         //启用交换空间
mkfs.ext4 /dev/nvme0n1p3     //格式化系统分区
mkfs.ext4 /dev/nvme0n1p4     //格式化个人分区
```

##### 挂载分区

```
mount /dev/nvme0n1p3 /mnt		//挂载系统分区到根目录
mkdir /mnt/boot 		//在根目录创建启动分区文件夹
mount /dev/nvme0n1p1 /mnt/boot  //挂载启动分区到根目录
mkdir /mnt/home   			 //在根目录创建个人文件夹
mount /dev/nvme0n1p4 /mnt/home 	//挂载个人分区到根目录
```

#### 设置pacman服务器

```
pacman -Sy  			//将本地数据库与远程仓库进行同步
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
						//将服务器地址备份到本地
reflector --verbose --latest 15 --sort rate --save /etc/pacman.d/mirrorlist				//用reflector把最新的15个服务器按最快速度排序
```

#### 安装基本系统及固件

```
pacstrap -i /mnt linux linux-headers linux-firmware base base-devel nano(文本编辑器) amd-ucode(amdcpu驱动)
```

##### 配置FSTAB(生成文件系统)

```
genfstab -U -p /mnt >> /mnt/etc/fstab  //将挂载分区生成到系统内
cat /mnt/etc/fstab	//查看是否生成成功
```

##### 进入新系统

```
arch-chroot /mnt
pacman -Syy        //同步一下本地数据库
```

##### 配置系统时区

```
ls -s /usr/share/zoneinfo/Asia/Shanghai  //查看可选时区
ln -s /usr/share/zoneinfo/Asia/Shanghai > /etc/localtime //创建软链接同步时区到本地localtime
hwclock --systohc  //设置硬件时间
```

##### 设置系统语言

```
nano /etc/locale.gen      //用nano打开locale文件
————————————————————————————————————————————————————————————
#en_US.UTF-8
#zh-CN.UTF-8			//找到这两个选项去掉#号再 ctrl^x 保存
————————————————————————————————————————————————————————————
locale-gen  		//读取保存的locale.gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
nano /etc/locale.conf   //查看是否保存正确
export LANG=en_US.UTF-8			//应用设置的语言
```

##### 给电脑起名

```
nano /etc/hostname
————————————————————————————————————————————————————
my-computer-arch
```

##### 配置网络（改host文件）

```
nano /etc/hosts
——————————————————————————————————————————————————————
127.0.0.1	localhost
::1			localhost
127.0.0.1	my-computer-arch.localdomain	my-computer-arch
```

##### 设置root密码

```
passwd
```

##### 添加用户及权限

```
useradd -m -g users -G wheel,storage,power -s /bin/bash LiuYue
passwd LiuYue      //设置用户登录密码
EDITOR=nano visudo		//改管理员权限sudo文件
————————————————————————————————————————————————————————————————————————————
## Umcomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL			//去掉前面#
Default rootpw			//最末尾添加默认管理密码
```

#### 安装和配置Systemd启动器（或者grub启动器）

```
bootctl install
nano /boot/loader/entries/arch.conf    //打开并修改启动器文件（文件名任取）
——————————————————————————————————————————————————————————————————————————————
title My Arch Linux
linux /vmlinuz-linux  //内核版本为长期支持版为linux /vmlinuz-linux-lts
initrd /amd-ucode.img	//读initrd ,intel的为intel-ucode.img
initrd /initramfs-linux.img	 //读取内存部分，长期支持版为/initramfs-linux-lts.img
```

##### 设置uuid到启动器文件中

```
echo "options root=PARTUUID=$(blkid -s PARTUUID -o value /dev/nvme0n1p3) rw" >> /boot/loader/entries/arch.conf		// /dev/nvme0n1p3为根分区，两个>>指针为添加到启动器文件末尾，一个指针>为覆盖
nano /boot/loader/entries/arch.conf
```

##### 启用SSD的TRIM功能

```
systemctl enable fstrim.timer
```

##### 开启32位应用支持（选择安装）

```
nano /etc/pacman.conf
——————————————————————————————————————————————————————————————————————
#[multilib]								//去掉前面#
#Include = /etc/pacman.d/mirrorlist		//去掉前面#
————————————————————————————————————————————————————————————————————————
pacman -Syy 同步本地数据库
```

#### 安装系统软件

```
pacman -S networkmanager network-manager-applet dialog (wpa_supplicant) dhcpcd(无线网络) rp-pppoe(有线网络)
systemctl enable NetworkManager   //开机自动启动无线网络管理器
systemctl enable dhcpcd
systemctl start adsl
pacman -S mtools dosfstools	reflector openssh
#bluez bluez-utils 				//蓝牙服务
#cups  							//打印机
#xdg-utils xdg-user-dirs 
#alsa-utils pulseaudio  pualseaudio-bluetooth //音频管理
```

##### 安装显卡驱动

```
pacman -S xf86-video-amdgpu mesa lib32-vulkan-radeon libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau  //amd显卡

#xf86-video-intel //intel显卡
#lib32-mesa //32位支持
#nvidia dkms libglvnd lib32-libglvnd nvidia-utils lib32-nvidia-utils opencl-nvidia lib32-opencl-nvidia nvidia-settings  //nvidia显卡
```

###### amd驱动配置

```
nano /boot/loader/entries/arch.conf
————————————————————————————————————————————————————————
在options最后添加 amdgpu.dpm=0 amdgpu.noretry=0
```

###### nvidia驱动配置

```
nano /etc/mkinitcpio.conf		//将英伟达显卡程序设置入arch的内核参数
————————————————————————————————————————————————————————————————————————————
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
//ctrl+x保存
——————————————————————————————————————————————————————————————————————————————
mkdir /etc/pacman.d/hooks		//创建一个钩子,保证驱动更新后不会出错
nano /etc/pacman.d/hooks/nvidia.hook
————————————————————————————————————————————————————————————————————————————————
[Trigger]
Operation=Install
Operation=Upgrage
Operation=Remove
Type=Package
Target=nvidia
Target=linux

[Action]
Depends=mkinitcpio
When=PostTransaction
Exec=/usr/bin/mkinitcpio -P
————————————————————————————————————————————————————————————————————————
nano /boot/loader/entries/arch.conf
————————————————————————————————————————————————————————————————————-————
在options最后添加  nvidia-drm.modeset=1
```

##### 安装xorg窗口系统

```
pacman -S xorg-server xorg-apps xorg-xinit xorg-twm xorg-xclock xterm konsole
pacman -S ttf-dejavu wqy-microher (Dejavu和微米黑字体)
#pacman -S xf86-input-synaptics //笔记本触摸板驱动
```

##### 安装KDE桌面环境(plasma)

```
pacman -S sddm plasma
systemctl enable sddm //开机启动sddm显示管理器
```

#### 完成安装重启

```
exit 				 //退出archiso
umount -R /mnt 		 //取消挂载所有空间
reboot  			 //重启
————————————————————————————————————————————————————————————————————————————————
sudo pacman -S neofetch dolphin(文件管理器) firefox-extended-support-release(ESR版) ark(压缩软件) p7zip git wget bashtop(资源监视软件) partitionmanager(磁盘分区管理器) kdeconnect(K易连连手机) yakuake(下拉式终端)
#firefox-developer-edition(开发者版本) steam(游戏平台) kate(高级文本管理器,类如记事本)
```

#### 安装输入法

```
sudo pacman -S fcitx fcitx-im kcm-fcitx fcitx-libpinyin fcitx-googlepinyin(谷歌拼音)
nano .pam_environment   //设置桌面环境自启动
__________________________________________________________
#fcitx
GTK_IM_MODULE DEFAULT=fcitx
QT_IM_MODULE  DEFAULT=fcitx
XMODIFIERS    DEFAULT=\@im=fcitx
___________________________________________________________
reboot
ctrl^space //激活输入法
ctrl^shift //切换输入法
```

#### 更改pacman镜像源

```
sudo pacman -Syyu     //更新系统
sudo pacman -Sc       //清理缓存
#sudo pacman -S archlinux-keyring //如果无法下载可更新密钥证书
sudo nano /etc/pacman.d/mirrorlist	//查看本地镜像源
sudo reflector --verbose --country China --country Japan --sort rate --age 12(最近12小时) --save /etc/pacman.d/mirrorlist
sudo pacman -Syy    //同步本地数据库
————————————————————————————————————————————————————————————————————————————————————
sudo nano /etc/xdg/reflector/reflector.conf   //调整自动更新的配置文件
# sudo systemctl enable reflector.service			//设置开机启动
——————————————————————————————————————————————————————————————————————————————————————————
#添加archlinuxcn源提速
sudo nano /etc/pacman.conf
    ————————————————————————————————————————————————————————————
[archlinuxcn]
Server = https://repo.archlinuxcn.org/$arch						//末尾添加
    ———————————————————————————————————————————————————————————
sudo pacman -Syy 		//同步
sudo pacman -S archlinuxcn-keyring		//添加cn源密钥
sudo pacman -S archlinuxcn-mirrorlist-git
```

安装Yay下载第三方aur软件

```
sudo pacman -Syyu //更新系统
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
exit
yay -S xx(aur软件名)
```

Powerpill多线程下载配置(提速) ^aur^

```
yay -S powerpill
sudo nano /etc/pacman.conf
SigLevel = PackageRequired //每个启用的库[core][extra][community][multilib]的中间都要添加一句该语法
sudo powerpill -Syy
```
