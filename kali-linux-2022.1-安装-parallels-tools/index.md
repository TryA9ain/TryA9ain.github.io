# Kali Linux 2022.2 安装 Parallels Tools


闲来无事, 一直听说 MAC 上 PD 虚拟机很丝滑, 就想上手体验一下, 但在安装 Kali 时遇到了一些问题, 就有了此文.

<!--more-->

## 软件版本

- Kali: 2022.1-amd64
- Parallels Desktop: 17.1.4
- macOS: Catalina 10.15.7

## 版本问题

在逛 [pd 的论坛](https://forum.parallels.com/forums/linux-guest-os-discussion.61/) 时发现里面说了 pd 现在还不支持最新的 5.18 内核, Kali 2022.2 版本 为 5.16.0 内核, 测试安装失败, Kali 2022.1 为 5.15.0 内核, 测试安装成功.

## 安装 Parallels Tools

选择 操作 --> 安装 Parallels Tools, 桌面会出现一个光盘图标, 由于里面的文件是**只读模式**, 因此需要先拷贝出来, 再运行.

```Bash
cd Desktop/
mkdir pd
cp -r /media/cdrom0/* /root/Desktop/pd/
chmod +x install
```

此处尝试 `./install` 安装报错.

显示缺少:

- libelf-dev
- printer-driver-postscript-hp
- dkms

安装:

```Bash
apt-get install dkms
apt-get printer-driver-postscript-hp
apt install libelf-dev
```

安装完毕后, 再次运行 `./install` 提示缺少 linux-header.

下载相关文件:

```Bash
wget https://old.kali.org/kali/pool/main/l/linux/linux-headers-5.15.0-kali3-amd64_5.15.15-2kali1_amd64.deb
wget https://old.kali.org/kali/pool/main/l/linux/linux-headers-5.15.0-kali3-common_5.15.15-2kali1_all.deb
wget https://old.kali.org/kali/pool/main/l/linux/linux-kbuild-5.15_5.15.15-2kali1_amd64.deb
```

下载完毕后, 进行安装.

```Bash
dpkg -i linux-headers-5.15.0-kali3-amd64_5.15.15-2kali1_amd64.deb
dpkg -i linux-headers-5.15.0-kali3-common_5.15.15-2kali1_all.deb
dpkg -i linux-kbuild-5.15_5.15.15-2kali1_amd64.deb
```

安装完毕后, 再次进入 pd 目录运行 `./install` 安装, 安装成功后会提示选择重启即安装成功, 即可畅享丝滑.

总体使用来说, 个人觉得 PD 确实比 Vmware Fusion 丝滑一些, 但绝对没有天差地别, 体会最深的一点是: 因为本人用的是触摸板, 在 PD 里面 和 Vmware Fusion 里面其实都不是特别好用, 但是 PD 至少是能将就用, Vmware Fusion 是真的不想将就触摸板. 个人觉得对于虚拟机工具来说, 选择一个自己习惯用的即可.

## 参考文章

https://www.sqlsec.com/2021/04/pdtools.html

---

> 作者: TryA9ain  
> URL: https://TryA9ain.github.io/kali-linux-2022.1-%E5%AE%89%E8%A3%85-parallels-tools/  

