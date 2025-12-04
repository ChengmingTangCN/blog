+++
title = 'Windows Dev Kit 2023 Install Ubuntu Kernel 6.18.0'
date = 2025-12-04T10:56:00+08:00
draft = true
tag = ['ARM64', 'Linux']
+++

因为之前的Linux内核编译时没有启用`CONFIG_ANON_VMA_NAME`, 不支持`PR_SET_VMA_ANON_NAME`. 所以想要再安装一个ubuntu官方编译的内核. 
24年初给Windows dev kit 2023安装ubuntu时基本是按照这篇教程[Windows Dev Kit 2023 Bootup](https://github.com/jglathe/linux_ms_dev_kit/discussions/1)瞎折腾的.
升级ubuntu官方构建的arm64 linux内核也遇到了一些问题, 根据回忆记录一下解决过程, 可能会有疏漏.
因为我不太熟悉linux编译与安装, 很多解决方案都是问的ai大模型, 能跑就行...

## 下载deb包

从[Ubuntu Mainline Kernel PPA](https://kernel.ubuntu.com/mainline/v6.18/)下载下面的deb包, 安装它们

```bash
$ ls linux-*.deb
linux-headers-6.18.0-061800_6.18.0-061800.202511302339_all.deb            linux-image-unsigned-6.18.0-061800-generic_6.18.0-061800.202511302339_arm64.deb
linux-headers-6.18.0-061800-generic_6.18.0-061800.202511302339_arm64.deb  linux-modules-6.18.0-061800-generic_6.18.0-061800.202511302339_arm64.deb
```

修改`/etc/default/grub`:
```
GRUB_DEFAULT=saved
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=30
```

更新grub
```bash
sudo update-grub
```

## grub选择新内核后自动跳回grub界面

猜测可能是ubuntu官方的arm64 deb没有提供dtbs, 先使用旧内核启动系统.

手动下载6.18源码，编译dtbs

```bash
tar xf ~/Downloads/linux-6.18.tar.xz
cd linux-6.18/
cp /boot/config-6.18.0-061800-generic .config
make oldconfig
make ARCH=arm64 dtbs
```

拷贝编译好的dtbs

```bash
sudo cp -r arch/arm64/boot/dts/* /boot/dtbs/6.18.0-061800-generic
cd /boot/
sudo ln -s dtbs/6.18.0-061800-generic/qcom/sc8280xp-microsoft-blackrock.dtb dtb-6.18.0-061800-generic
```

### 启动新内核后gnome登录后自动跳回登录界面

通过tty登录, 查看dmesg，发现报错提示找不到mbn文件.
```
[Wed Dec 3 18:38:34 2025] adreno 3d00000.gpu: [drm:zap_shader_load_mdt [msm]] *ERROR* Unable to load qcom/sc8280xp/microsoft/blackrock/qcdxkmsuc8280.mbn
[Wed Dec 3 18:38:34 2025] msm_dpu ae01000.display-controller: [drm:adreno_load_gpu [msm]] *ERROR* gpu hw init failed: -2
[Wed Dec 3 18:38:34 2025] platform 3d6a000.gmu: [drm:a6xx_gmu_set_oob [msm]] *ERROR* Timeout waiting for GMU OOB set GPU_SET: 0x0
```

应该是新内核找mbn文件的路径发生了改变, 在`/lib/firmware/`下搜索`qcdxkmsuc8280.mbn`, 发现在其它目录下, 拷贝它们
```bash
sudo mkdir -pv /lib/firmware/qcom/sc8280xp/microsoft/blackrock/
sudo cp -r /usr/lib/firmware/updates/qcom/sc8280xp/MICROSOFT/DEVKIT23/* /usr/lib/firmware/qcom/sc8280xp/microsoft/blackrock/
```

### 结语

现在可以正常使用系统了, 尽管识别到了未知电池.

![alt](/images/2025-12-04-11-40-46.png)



