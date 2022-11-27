# 利用 keytab 和 WireShark 解密 Kerberos


本文主要记录了如何通过一系列操作, 将生成的 keytab 文件导入 WireShark, 实现可以在 WireShark 中直接对 Kerberos 协议加密部分进行解密.

<!--more-->

## 简介

keytab (key table, 密钥表) 是一个加密文件, 包含受 Kerberos 保护的服务及其在密钥分发中心 (Key Distribution Center, KDC) 中关联的服务主体名称 (service principal name, SPN) 的长期密钥. 其用途之一就是解密 Kerberos. 

有关 keytab 的更多信息请阅读这篇文章: [Kerberos Keytabs – Explained](https://social.technet.microsoft.com/wiki/contents/articles/36470.kerberos-keytabs-explained.aspx).

## 域控上导出 NTDS.dit 和 system.hive

这里使用用卷影复制的方法, 复制的时候要和自己的卷影副本卷名对应, 复制完成后删除之前创建的卷影副本.

```PowerShell
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\NTDS.dit
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\system.hive
vssadmin delete shadows /all
```

## 安装 ntdsxtract

需要使用 Python2 来安装 pycryptodome 模块.

```Bash
git clone https://github.com/csababarta/ntdsxtract.git
cd ntdsxtract/ && python2 -m pip install pycryptodome
```

## 安装 esedbexport

>尽量在 linux 下安装, Windows 下编译可能会有一些问题. Kali Linux 2022.3 版本已经集成了 esedbexport.

安装命令

```Bash
apt-get install libesedb-utils
```

## 导出 NTDS.dit 中的内容

Active Directory 的数据库引擎是[可扩展存储引擎](https://learn.microsoft.com/en-us/windows/win32/extensible-storage-engine/extensible-storage-engine)(Extensible Storage Engine, ESE), 它基于 Exchange 5.5 和 WINS 使用的 Jet 数据库. 通俗来讲就是 Jet 数据库引擎. 

NTDS.dit 文件是一种 ESE (Extensible Storage Engine, 可扩展存储引擎) 数据库文件.

此处的 esedbexport 可以理解为将 NTDS.dit 文件的内容进行了一个导出.

将 NTDS.dit 和 system.hive 放在同一目录, 使用 esedbexport 将 NTDS.dit 中的内容进行导出, 导出的文件会存放在当前目录的 ntds.dit.export 目录下.

```Bash
esedbexport NTDS.dit
```

![](images/Pasted%20image%2020221105123107.png)

## 使用 NTDSXtract 导出 keytab

- datatable 和 link_table 是 esedbexport 从 NTDS.dit 中导出的文件, 存放于 NTDS.dit.export 目录下.
- system.hive 是之前导出的文件.
- result 是新建的目录.
- 1.keytab 是执行命令后生成 keytab 文件.

```Bash
mkdir result
python2 ntdsxtract/dskeytab.py NTDS.dit.export/datatable.4 NTDS.dit.export/link_table.7 system.hive ./result/ 1.keytab
```

![](images/Pasted%20image%2020221105125916.png)

生成 1.keytab 后, 我们可以使用 ktutil 查看当前 1.keytab 文件内的信息.

想要使用 ktutil, 需要安装 Kerberos 客户端, 这里安装的是 heimdal-clients.

```Bash
apt update
apt install heimdal-clients
```

执行命令查看 1.keytab 中包含的信息.

```Bash
ktutil -k 1.keytab list
```

![](images/Pasted%20image%2020221127145509.png)

可以看到所有在域控中注册的服务, 用户, 主机, 都列了出来.

在后面我们可以利用 1.keytab 和 WireShark 解开 Kerberos 协议中的 TGT, ST, 用户 Hash 加密的部分, 服务 Hash 加密的部分. 这对于进一步学习 Kerberos 协议的认证过程有着非常大的帮助.

## 配置 WireShark

打开 WireShark --> 编辑 --> 首选项 --> Protocols --> KRB5 --> 勾选 "Try to decrypt Kerberos blobs" --> 浏览, 导入刚才生成的 1.keytab 文件.

![](images/Pasted%20image%2020221105130238.png)

加载完 1.keytab 文件后, WireShark 会自动对当前的 Kerberos 数据包进行解密尝试, 如果解密成功, 就会是显示蓝色, 不成功就是黄色 (有的黄色包内也包裹着蓝色包, 所以要耐心的一个一个包去看).

至此配置完毕, 下篇我们来分析 Kerberos 认证过程.

**感谢耐心阅读, 文章仅供参考, 本人学艺不精, 不足之处欢迎师傅们指点和纠正!**

## 参考链接

https://mp.weixin.qq.com/s/eTiQRoEh1DvNGEuPErYsAw

https://en.wikipedia.org/wiki/Extensible_Storage_Engine

https://www.windowstechno.com/what-is-ntds-dit/

---

> 作者: TryA9ain  
> URL: https://TryA9ain.github.io/%E5%88%A9%E7%94%A8-keytab-%E5%92%8C-wireshark-%E8%A7%A3%E5%AF%86-kerberos/  

