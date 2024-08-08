# 介绍

deepin v23 加入了 arm64 支持，这里尝试将 deepin 系统刷入平板中，平常使用中，带个笔记本电脑有时候也会嫌比较麻烦，把 Linux 系统刷入平板中既满足了使用需要，又满足了轻便的需求。为什么不使用 termux ？虽然 termux 很方便，但是不想直接嵌套一层系统运行，希望能够获取更好的使用性能。然后上网查阅资料后，选中了小米平板5，不得不说小米为发烧而生。下面是关于小米平板5刷入系统的介绍，关于获取 root 权限，以及解 bootloader 锁的内容不做过多介绍。先叠个甲，如果有人想尝试刷机，请先确认具备刷机相关知识，产生的后果自行负责。

# 制作根文件系统

```bash
git clone https://github.com/chenchongbiao/dev-tools
go mod vendor
make
```

```bash
sudo ./bin/dp-build build rootfsimg -n="deepin" -v="beige" -c="main,commercial,community" -a="arm64" -s="deb https://community-packages.deepin.com/beige/ beige main commercial community" -d "mipad5"
```

如果需要自己制作默认不安装图形界面。可以用nmcli 配置下网络。安装以下桌面包。

```bash
sudo apt install deepin-desktop-environment-base deepin-desktop-environment-cli deepin-desktop-environment-core deepin-desktop-environment-extras
```

# 编译内核

## 安装编译环境

```bash
sudo apt install libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm gcc-aarch64-linux-gnu
```

## 获取内核源码

```bash
git clone https://github.com/maverickjb/linux-6.1.10.git
```

## 编译源码

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- xiaomi_nabu_maverick_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image.gz dtbs
make ARCH=arm64 install INSTALL_PATH=../install/boot
make ARCH=arm64 dtbs_install INSTALL_DTBS_PATH=../install/boot/dtbs

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
rm -rf ../install/lib/modules/
make ARCH=arm64 modules_install INSTALL_MOD_PATH=../install
```

## 打包boot.img

安装 mkbootimg

```
sudo apt install mkbootimg
```

```bash
cat linux-6.1.10/arch/arm64/boot/Image.gz linux-6.1.10/arch/arm64/boot/dts/qcom/sm8150-xiaomi-nabu-maverick.dtb > zImage
mkbootimg --kernel zImage --cmdline "console=tty0 root=/dev/sda33 rw rootwait" --base 0x00000000 --kernel_offset 0x00008000 --tags_offset 0x00000100 --pagesize 4096 --id -o boot.img
```

/dev/sda32 是用来安装 deepin 系统的分区名称。

ps: mkbootimg 1:34.0.4-1 上使用了gki 模块，但是打包并没有引入该模块。使用了 Docker 安装低版本的mkbootimg 使用。

## 编译UEFI固件

```
git clone --recursive git@github.com:edk2-porting/edk2-msm.git
```

将 edk2-msm/Platform/Xiaomi/sm8150/FdtBlob/nabu/  中的dtb文件 sm8150-xiaomi-nabu.dtb 替换为前面编译的内核DTB文件，重命名为 sm8150-xiaomi-nabu.dtb 并构建镜像：

```bash
./builder.sh -d nabu
```

# 对 UFS 进行分区

## userdata 重新分区

要修改 UFS 上的分区，需要使用 Orangefox Recovery 第三方恢复环境，[xiaomi-nabu-orangefox.img](https://drive.google.com/file/d/1gCNtoDMNCAmMR61xegvCC3mxv28gMJbi/view?usp=drive_link)

手机USB接入系统，需要使用 adb 工具。

```bash
sudo apt install adb fastboot
```

手机的开发者选项打开USB调试。

进入 bootloader

```bash
adb reboot bootloader
```

开始启动恢复映像。屏幕打开后，使用 adb shell 继续操作。

需要使用 fastboot 工具

```bahh
sudo apt install adb fastboot
```

```bash
fastboot boot xiaomi-nabu-orangefox.img
```

```bash
adb shell
```

查看分区

```bash
ls -l /dev/block/bootdevice/by-name/ | grep userdata
```

```
lrwxrwxrwx 1 root root 16 1970-01-16 07:05 userdata -> /dev/block/sda31
```

userdata 分区位于整个磁盘的第31个分区。

要调整 userdata  分区的大小，需要使用parted命令工具。adb shell 终端中输入 parted 。

```bash
parted /dev/block/sda
```

输入 print 命令列出 /dev/block/sda 的所有分区：

```bash
(parted) print
print
Model: SAMSUNG KLUFG8RHGB-B0E1 (scsi)
Disk /dev/block/sda: 509GB
Sector size (logical/physical): 4096B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name             Flags
 1      24.6kB  32.8kB  8192B                switch
 2      32.8kB  65.5kB  32.8kB               ssd
 3      65.5kB  98.3kB  32.8kB               dbg
 4      98.3kB  131kB   32.8kB               bk01
 5      131kB   262kB   131kB                bk02
 6      262kB   524kB   262kB                bk03
 7      524kB   1049kB  524kB                bk04
 8      1049kB  1573kB  524kB                keystore
 9      1573kB  2097kB  524kB                frp
10      2097kB  4194kB  2097kB               countrycode
11      4194kB  8389kB  4194kB               misc
12      8389kB  12.6MB  4194kB               vm-data
13      12.6MB  16.8MB  4194kB               bk06
14      16.8MB  25.2MB  8389kB               logfs
15      25.2MB  33.6MB  8389kB               ffu
16      33.6MB  50.3MB  16.8MB               oops
17      50.3MB  67.1MB  16.8MB               devinfo
18      67.1MB  83.9MB  16.8MB               oem_misc1
19      83.9MB  101MB   16.8MB  ext4         metadata
20      101MB   134MB   32.9MB               bk08
21      134MB   168MB   34.2MB               splash
22      168MB   201MB   33.6MB               bk09
23      201MB   9328MB  9127MB               super
24      9328MB  9328MB  131kB                vbmeta_system_a
25      9328MB  9328MB  131kB                vbmeta_system_b
26      9328MB  9396MB  67.1MB               logdump
27      9396MB  9530MB  134MB                minidump
28      9530MB  9664MB  134MB                rawdump
29      9664MB  10.7GB  1074MB  ext4         cust
30      10.7GB  10.9GB  134MB   ext4         rescue
31      10.9GB  509GB   499GB                userdata
```

重新划分 31 分区

```bash
(parted) resizepart 31                            
resizepart 31
End?  [210GB]? 209GB                              
209GB
Warning: Shrinking a partition can cause data loss, are you sure you want to
continue?
Yes/No? Yes                                       
Yes
```

使用 print 再次查看结果，分区31的分区变小了。

## 扩展主分区

如果需要将分区数量从 32 扩展到 64，q 退出分区工具，执行下面的命令。

```bash
sgdisk --resize-table 64 /dev/block/sda   
```

## 创建 esp 分区

```bash
parted /dev/block/sda
```

```bash
(parted) mkpart esp fat32 209GB 210GB
# set the esp partition as `EFI system partition type`
```

退出分区工具执行

```bash
mkfs.fat -F32 -s1 /dev/block/sda32
```

进入分区工具，启用 esp

```bash
set 32 esp on
```

使用 print 再次查看结果

```bash
(parted) print                                          
print
Model: SAMSUNG KLUFG8RHGB-B0E1 (scsi)
Disk /dev/block/sda: 509GB
Sector size (logical/physical): 4096B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name             Flags
 1      24.6kB  32.8kB  8192B                switch
 2      32.8kB  65.5kB  32.8kB               ssd
 3      65.5kB  98.3kB  32.8kB               dbg
 4      98.3kB  131kB   32.8kB               bk01
 5      131kB   262kB   131kB                bk02
 6      262kB   524kB   262kB                bk03
 7      524kB   1049kB  524kB                bk04
 8      1049kB  1573kB  524kB                keystore
 9      1573kB  2097kB  524kB                frp
10      2097kB  4194kB  2097kB               countrycode
11      4194kB  8389kB  4194kB               misc
12      8389kB  12.6MB  4194kB               vm-data
13      12.6MB  16.8MB  4194kB               bk06
14      16.8MB  25.2MB  8389kB               logfs
15      25.2MB  33.6MB  8389kB               ffu
16      33.6MB  50.3MB  16.8MB               oops
17      50.3MB  67.1MB  16.8MB               devinfo
18      67.1MB  83.9MB  16.8MB               oem_misc1
19      83.9MB  101MB   16.8MB  ext4         metadata
20      101MB   134MB   32.9MB               bk08
21      134MB   168MB   34.2MB               splash
22      168MB   201MB   33.6MB               bk09
23      201MB   9328MB  9127MB               super
24      9328MB  9328MB  131kB                vbmeta_system_a
25      9328MB  9328MB  131kB                vbmeta_system_b
26      9328MB  9396MB  67.1MB               logdump
27      9396MB  9530MB  134MB                minidump
28      9530MB  9664MB  134MB                rawdump
29      9664MB  10.7GB  1074MB  ext4         cust
30      10.7GB  10.9GB  134MB   ext4         rescue
31      10.9GB  209GB   198GB                userdata
32      209GB   210GB   1000MB               esp              boot, esp
```

检查分区是否正确，q 退出。

## 新建分区

```bash
(parted) mkpart deepin ext4 210GB 509GB
mkpart deepin ext4 210GB 509GB
```

deepin 为分区名，ext4 为文件系统，210GB起始位置，509为前面userdata原来的结束位置。

```bash
(parted) print                                            
print
Model: SAMSUNG KLUFG8RHGB-B0E1 (scsi)
Disk /dev/block/sda: 509GB
Sector size (logical/physical): 4096B/4096B
Partition Table: gpt
Disk Flags: 

Number  Start   End     Size    File system  Name             Flags
 1      24.6kB  32.8kB  8192B                switch
 2      32.8kB  65.5kB  32.8kB               ssd
 3      65.5kB  98.3kB  32.8kB               dbg
 4      98.3kB  131kB   32.8kB               bk01
 5      131kB   262kB   131kB                bk02
 6      262kB   524kB   262kB                bk03
 7      524kB   1049kB  524kB                bk04
 8      1049kB  1573kB  524kB                keystore
 9      1573kB  2097kB  524kB                frp
10      2097kB  4194kB  2097kB               countrycode
11      4194kB  8389kB  4194kB               misc
12      8389kB  12.6MB  4194kB               vm-data
13      12.6MB  16.8MB  4194kB               bk06
14      16.8MB  25.2MB  8389kB               logfs
15      25.2MB  33.6MB  8389kB               ffu
16      33.6MB  50.3MB  16.8MB               oops
17      50.3MB  67.1MB  16.8MB               devinfo
18      67.1MB  83.9MB  16.8MB               oem_misc1
19      83.9MB  101MB   16.8MB  ext4         metadata
20      101MB   134MB   32.9MB               bk08
21      134MB   168MB   34.2MB               splash
22      168MB   201MB   33.6MB               bk09
23      201MB   9328MB  9127MB               super
24      9328MB  9328MB  131kB                vbmeta_system_a
25      9328MB  9328MB  131kB                vbmeta_system_b
26      9328MB  9396MB  67.1MB               logdump
27      9396MB  9530MB  134MB                minidump
28      9530MB  9664MB  134MB                rawdump
29      9664MB  10.7GB  1074MB  ext4         cust
30      10.7GB  10.9GB  134MB   ext4         rescue
31      10.9GB  209GB   198GB                userdata
32      209GB   210GB   1000MB               esp              boot, esp
33      210GB   509GB   299GB   ext4         deepin
```

重启设备。

# 获取 MIUI 设备树和引导

这里需要获取MIUI的设备树和引导用来后续切换系统使用。

需要获取平板的root权限，这里不做介绍。平板上安装 [linuxswitch](https://github.com/timoxa0/Switch2Linux-Nabu/releases/download/v1.0.2/linuxswitch.apk) 后，点击 `Dump android images`，平板这里使用板内部存储中的 `linux` 文件夹，并且从该文件夹中取出 `android.boot.img` 和 `android.dtbo.img`，把前面打包的 `boot.img` 重命名为  `linux.boot.img` 放入平板内部存储中的 `linux` 文件夹。

# 刷入系统

进入 bootloader

```bash
adb reboot bootloader
```

找到准备好的系统镜像，开始之前，我们需要禁用 Android Verified Boot (AVB) 功能，否则将无法启动系统。

```bash
╰─❯ fastboot flash vbmeta_ab --disable-verification --disable-verity vbmeta.img                                                                                    
Sending 'vbmeta_ab' (4 KB)                         OKAY [  0.013s]
Writing 'vbmeta_ab'                                (bootloader) Partition vbmeta_a flashed successfully
(bootloader) Partition vbmeta_b flashed successfully
OKAY [  0.006s]
Finished. Total time: 0.046s
```

清除设备树

```bash
fastboot erase dtbo_ab
```

刷入内核映像 boot.img

```bash
fastboot flash boot_ab linux.boot.img
Sending 'boot_b' (13052 KB)                        OKAY [  0.327s]
Writing 'boot_b'                                   OKAY [  0.052s]
Finished. Total time: 0.663s
```

刷入rootfs img

```bash
fastboot flash deepin rootfs.img
```

deepin 为前面创建的分区名。

最后重启

# 初始化配置

制作镜像时默认创建了用户，用户名：deepin，密码：deepin

第一次进入系统需要默认是竖屏，需要在控制中心-->显示-->方向270度。

系统默认英文，控制中心-->键盘和语言-->语言-->添加简体中文。

音频问题禁用 pulseaudio，使用pipewire-pulse。

```bash
systemctl --user disable pulseaudio.{socket,service}
systemctl --user mask pulseaudio

systemctl --user enable pipewire.{socket,service}
systemctl --user enable pipewire-pulse.{socket,service}
```

刷入系统没自动扩容需要手动操作。

```bash
sudo resize2fs /dev/sda33
```

# 安装Switch2Linux

在 Linux 上 [s2a](https://github.com/timoxa0/Switch2Linux-Nabu/releases/download/v1.0.1/s2a.zip) 工具，解压，并把之前的 `android.boot.img` 和 `android.dtbo.img` 拷到系统上，放到 s2a 文件夹里。

```bash
.
├── install.sh
├── s2a
│   ├── android.boot.img
│   ├── android.dtbo.img
│   └── s2a
├── S2A.desktop
└── s2a.svg
```

运行 `sudo ./install.sh` 安装 s2a，注销并重新登录，即可在菜单栏中找到 Switch2Android 选项。

在安卓侧可以从 Switch to Linux 这个 app 切换到 Linux。

# 尝试启用 UEFI

不过看上去并没有成功，刷入后也没自动识别出放在efi分区的引导，可能引导文件不对。不过看项目没看到资料，先搁置。

## 编译UEFI固件

克隆 edk2-msm 仓库

```bash
git clone --recursive git@github.com:edk2-porting/edk2-msm.git
```

```bash
sudo apt install clang
cd edk2-msm
./build.sh -d nabu
```

生成boot-nabu.img文件

## 修改 fstab

```bash
╰─❯ sudo blkid /dev/sda32                                                                                         
/dev/sda32: UUID="0136-67CF" BLOCK_SIZE="4096" TYPE="vfat" PARTLABEL="esp" PARTUUID="5542f54c-b776-49bf-992e-3cec1e7bfe3d"
╰─❯ sudo blkid /devsda33                                                                                             
/dev/sda33: UUID="dfc62099-10f1-4f7d-a21c-bbc5f44d2345" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="deepin" PARTUUID="5344ad3c-085c-4265-bee7-045aef1bb791"
```

修改 /etc/fstab 为上面的 UUID

```bash
UUID=dfc62099-10f1-4f7d-a21c-bbc5f44d2345 / ext4 errors=remount-ro,x-systemd.growfs 0 1
UUID=0136-67CF /boot/efi vfat umask=0077 0 1
```

## 安装GRUB工具并生成RAM盘

```bash
sudo apt-get install grub2-common grub-efi
sudo grub-install --target=arm64-efi --boot-directory=/boot
```

```bash
sudo apt install initramfs-tools
sudo mkinitramfs -o /boot/initrd.img 6.1.10-nabu+
```

生成GRUB配置文件：

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## 刷入 boot.img

进入 bootloader

```bash
fastboot flash boot boot-nabu.img
```

# 参考

[It&#39;s Linux, On Mi Pad 5](https://linux-on-nabu.gitbook.io/linux-for-mi-pad-5)

[maverick-jia](https://www.cnblogs.com/maverick-jia)

[在小米平板 5 上安装 Arch Linux](https://blog.chyk.ink/2024/06/22/mipad5-archlinux)

[如何指导 在 nabu 上刷入 ubuntu 镜像后的配置](https://xdaforums.com/t/configurations-after-flash-the-ubuntu-images-on-nabu.4610535/#post-89371063)
