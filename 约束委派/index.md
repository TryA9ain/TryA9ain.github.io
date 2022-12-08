# 约束委派详解


本文主要记录了通过 WireShark 解开 Kerberos 协议对约束委派进行详细分析.

<!--more-->

## 基本概念

微软在 Windows 2003 上发布了约束委派, 其中包括一组 Kerberos 协议扩展: S4U2Self 和 S4U2Proxy. 

需要 SeEnableDelegationPrivilege 权限才能配置约束委派和非约束委派, 约束委派在配置的时候需要**指定允许委派到某个服务**.

例如:

WIN-2008\$ 配置了到 WEB.missyou.com 的 CIFS 服务的约束委派, 那么 WIN-2008\$ 可以模拟成任何用户去请求 WEB.missyou.com 的 CIFS 服务, **且仅能请求该服务**.

![](images/Pasted%20image%2020221206151452.png)

如果模拟的用户 (PA-FOR-USER 中的用户) 配置了 "敏感帐户, 不能被委派" 或者为 "Protected Users Security Group (受保护的用户安全组)" 中的成员, 则返回的 ST 中不包含 FORWARDABLE 标志, **约束委派中 S4U2proxy 阶段需要的 ST 必须是可转发的**.

![](images/Pasted%20image%2020221207112120.png)

## TrustedToAuthenticationForDelegation

**建议先行阅读 [MS-SFU specification](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/c98bade9-cad1-4745-bd4d-d13926103022) 规范**.

来看下官网的介绍:

> **TRUE**: the KDC MUST set the FORWARDABLE [ticket](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/4a624fb5-a078-4d30-8ad1-e9ab71e0bc47#gt_838d3fe1-e504-4442-93cc-75de14e6f569) flag ([RFC4120] section 2.6) in the [S4U2self](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/4a624fb5-a078-4d30-8ad1-e9ab71e0bc47#gt_2214804a-4a44-46f4-b6d2-a78f4ff39a39) service ticket.
>
> **FALSE** and _ServicesAllowedToSendForwardedTicketsTo_ is nonempty: the KDC MUST NOT set the FORWARDABLE ticket flag ([RFC4120] section 2.6) in the S4U2self service ticket.[<16>](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/a47e0084-d6c3-40ba-8c3c-f1eeb3d85ecf#Appendix_A_16)

个人理解的是:

- 如果服务帐户的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 TRUE => 返回 FORWARDABLE 的 ST.

- 如果服务帐户的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 FALSE, 并且服务帐户在 msDS-AllowedToDelegateTo 中有服务 => 返回无 FORWARDABLE 的 ST.

### TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 TRUE

参考[官方文档](https://learn.microsoft.com/zh-cn/windows-server/security/group-managed-service-accounts/configure-kerberos-delegation-group-managed-service-accounts) 来设置 TRUSTED_TO_AUTH_FOR_DELEGATION 为 True.

```PowerShell
Set-ADAccountControl -Identity win-2008$ -TrustedForDelegation $false -TrustedToAuthForDelegation $true
```

![](images/Pasted%20image%2020221202144527.png)

使用 Rubeus 模拟 S4U2self 请求, 并查看 S4U2self ST 的 Flags.

```PowerShell
Rubeus.exe s4u /user:win-2008$ /rc4:2e34c7eac82c4ef73938495c98510401 /domain:missyou.com /impersonateuser:Administrator /dc:dc.missyou.com /altservice:CIFS/web.missyou.com /outfile:test3.kirbi /nowrap /self
Rubeus.exe describe /ticket:test3_Administrator_to_HOST@MISSYOU.COM.kirbi
```

![](images/Pasted%20image%2020221207140424.png)

可以看到当 TrustedToAuthForDelegation 为 True 时, KDC 必须在 S4U2self ST 中设置 FORWARDABLE 标志.

### TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 FALSE

先来查看 WIN-2008\$ 的 TRUSTED_TO_AUTH_FOR_DELEGATION 属性和 msDS-AllowedToDelegateTo 属性.

```PowerShell
Get-ADComputer win-2008 -Properties TrustedToAuthForDelegation
Get-ADComputer win-2008 -Properties msDS-AllowedToDelegateTo
```

![](images/Pasted%20image%2020221202142823.png)

![](images/Pasted%20image%2020221207140018.png)

使用 Rubeus 模拟 S4U2self 请求, 并查看 S4U2self ST 的 Flags.

```PowerShell
Rubeus.exe s4u /user:win-2008$  /rc4:2e34c7eac82c4ef73938495c98510401 /domain:missyou.com /impersonateuser:Administrator /dc:dc.missyou.com /altservice:CIFS/web.missyou.com /outfile:test4.kirbi /nowrap /self
Rubeus.exe describe /ticket:test4_Administrator_to_CIFS@MISSYOU.COM.kirbi
```

![](images/Pasted%20image%2020221207135439.png)

可以看到如果服务帐户的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 FALSE, 并且服务帐户在 msDS-AllowedToDelegateTo 中有服务, KDC 会返回无 FORWARDABLE 的 ST.

## S4U2self

**建议先行阅读 [MS-SFU specification](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-sfu/c98bade9-cad1-4745-bd4d-d13926103022) 规范**.

Kerberos S4U2self 扩展允许一个服务以用户的名义为自己请求一个 ST, 即允许服务模拟用户向自己请求 ST, 并在 S4U2proxy 中使用.

这样做的目的是为了让那些不支持 Kerberos 协议的客户端能够执行 Kerberos 委派, 这也被称为**协议转换**.

**在域内任何具有 SPN 的主体都可以发起 S4U2self 请求**, 下面简单介绍 KDC 在收到 S4U2self 请求时进行的检查:

- 如果服务帐户没有任何服务 => 返回错误 KDC-ERR-S-PRINCIPAL-UNKNOWN.
- 如果模拟的帐户 (PA-FOR-USER 中的帐户) 配置了 "敏感帐户, 不能被委派" 或者为 "Protected Users Security Group (受保护的用户安全组)" 中的成员 => 返回无 FORWARDABLE 的 ST.
- 如果服务帐户的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 TRUE => 返回 FORWARDABLE ST.
- 如果服务帐户的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志为 FALSE, 并且服务帐户在 msDS-AllowedToDelegateTo 中有服务 => 返回无 FORWARDABLE 标志的 ST.

来举个例子: 

```
                                                                KDC
                          .---. >---2) TGS-REQ-------------->  .---.
                         /   /|     + SPN: HTTP/websrv        /   /|
  ____                  .---. |     + For user: client       .---. |
 |    | >-1) SPNEGO-->  |   | '     + TGT websrv$            |   | '
 |____|     (NTLM)      |   |/                               |   |/ 
 /::::/                 '---'  <----3) TGS-REP-------------< '---'  
 client                websrv   + ST client > HTTP/websrv
```


1. 由于 HTTP/websrv 服务不支持 Kerberos 协议, 所以客户端将通过 NTLM (或其他认证协议) 对其进行身份验证.
2. websrv 通过向 KDC 发送 TGS-REQ 来为客户端请求 S4U2self ST.
3. KDC 将检查 websrv\$ 的 TRUSTED_TO_AUTH_FOR_DELEGATION 标志以及 FOR-USER (此处为 client) 是否受到保护而无法进行委派, 如果一切正常, KDC将为客户端返回一个 HTTP/websrv ST, 这个 ST 可能是 FORWARDABLE, 也可能不是, 具体取决于前面提到的变量.

## S4U2proxy

S4U2proxy 拓展的作用是:

使用 WIN-2008\$ 的 TGT 模拟 Administrator 用户去向 KDC 请求一张用于访问 WEB.missyou.com 的 CIFS 服务的 ST, S4U2self 阶段获取到的 ST 将作为 TGS-REQ 的 additional-tickets 字段一起发送给 KDC.

在 S4U2proxy 阶段, KDC 会检查 WIN-2008\$ 的 msDS-AllowedToDelegateTo 属性, 检查 WEB.missyou.com 的 CIFS 服务是否在该属性中, 如果验证成功, 则返回 ST.

## 整体流程

1.  WIN-2008\$ 以 WIN-2008\$ NT Hash 加密时间戳并设置可转发的字段向 KDC 申请可转发的 TGT (不仅是约束委派, 正常的 TGT 申请都会带上可转发字段, 并且返回的TGT 也是可转发的).
2.  **S4U2self**: WIN-2008\$ 携带可转发的 TGT 并使用 S4U2self 扩展模拟用户 Administrator 获得针对 WIN-2008\$ 自身的可转发的 ST, **S4U2Self 阶段不需要 Administrator 的凭据**.
3.  **S4U2proxy**: WIN-2008\$ 携带步骤 1 申请到的 TGT 和步骤 2 获取的可转发的 ST (S4U2self ST) 发送给 KDC, 请求模拟 Administrator 用户访问 WEB.missyou.com 的 CIFS 服务的 ST.
4.  WIN-2008\$ 拿着步骤 3 获取的 ST (S4U2proxy ST) 模拟用户 Administrator 向 WEB.missyou.com 的 CIFS 服务发起请求.

使用 rubeus 模拟约束委派请求:

```PowerShell
Rubeus.exe s4u /outfile:1a.kirbi /user:win-2008$  /rc4:2e34c7eac82c4ef73938495c98510401 /impersonateuser:administrator /msdsspn:CIFS/web.missyou.com /ptt
dir \\web.missyou.com\c$
```

WireShark 抓包:

![](images/Pasted%20image%2020221205172253.png)

AS-REQ 与 AS-REP 就是正常的用 WIN-2008\$ 凭据去申请 TGT, 和之前的文章 [Kerberos 认证过程分析](https://trya9ain.github.io/kerberos%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/) 的内容差不多就不细说了.

## 第一个 TGS-REQ (S4U2self)

第一个 TGS-REQ 就是通过 S4U2self 扩展模拟 administrator 用户获取 WIN-2008\$ 的 ST.

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221205220206.png)

#### padata

![](images/Pasted%20image%2020221205221132.png)

PA-DATA pA-TGS-REQ: 这部分包含了 WIN-2008$ 的 TGT 和 Logon session key 加密的时间戳等信息.

PA-DATA pA-FOR-USER: 此结构标识模拟的用户 administrator 以及域名 MISSYOU.com.

#### req-body

可以看到 CName 是 win-2008\$, SName 也是 win-2008\$.

![](images/Pasted%20image%2020221205221329.png)

### 解密后的数据包

主要看下此处 TGT 中的 PAC 信息, 可以看到当前 PAC 还是 WIN-2008\$ 权限, Group RID: 515 (Domain Computers 组).

![](images/Pasted%20image%2020221207172039.png)

## 第一个 TGS-REP (S4U2self)

### 原数据包

注意此处的 CName 已经变成了 administrator.

![](images/Pasted%20image%2020221205222708.png)

### 解密后的数据包

这里重点关注此处的 ST 中的 PAC 权限变成了 Administrator.

- Group RID: 512, Domain Admins.
- Group RID: 520, Group Policy Creator Owners.
- Group RID: 513, Domain Users.
- Group RID: 519, Enterprise Admins.
- Group RID: 518, Schema Admins.

![](images/Pasted%20image%2020221207172228.png)

## 第二个 TGS-REQ (S4U2proxy)

第二个 TGS-REQ 请求就是通过 S4U2proxy 扩展模拟 administrator 用户获取 WEB.missyou.com 的 CIFS 服务的 ST.

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221206141623.png)

#### padata

注意这里的 ticket 是 WIN-2008\$ 的 TGT (后续解包会看到).

![](images/Pasted%20image%2020221206141822.png)

#### req-body

可以看到多出来了一个 additional-tickets 字段, 这个字段的 ticket 就是 S4U2self 获取到的 ST (必须是有可转发字段的).

![](images/Pasted%20image%2020221206142109.png)

### 解密后的数据包

#### padata

主要来看下 TGT 的解密, 重点关注 PAC 中的信息, 可以看到当前的权限是 WIN-2008\$, 不是 Administrator 哦.

![](images/Pasted%20image%2020221206142955.png)

![](images/Pasted%20image%2020221206143345.png)

#### additional-tickets

这个字段的 ticket 就是 S4U2self 获取到的 ST (必须是有可转发字段的).可以看到跟之前 S4U2self TGS-REQ 中的 ST 是一样的.

![](images/Pasted%20image%2020221206144300.png)

![](images/Pasted%20image%2020221206144620.png)

## 第二个 TGS-REP (S4U2proxy)

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221206144801.png)

- ticket: 这个就是 ST, 由服务账号的 NT Hash 加密.

- enc-part: Logon session key 加密的 Server Session key 等信息.

可以看到是 administrator 请求 WEB.missyou.com 的 CIFS 服务.

![](images/Pasted%20image%2020221206144954.png)

### 解密后的数据包

可以看到是用 WEB$ 服务账号信息解密的.

重点关注此处的 ST 中的 PAC 权限为 administrator.

![](images/Pasted%20image%2020221206145409.png)

![](images/Pasted%20image%2020221206150224.png)

## AP-REQ

用户解密 Logon session key 加密的内容, 获得 Server Session key, 然后把用户名, 时间戳等信息用 Server Session key 加密, 同 ST 发送给目标服务.

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221208101110.png)

主要看 Kerberos.

![](images/Pasted%20image%2020221208095535.png)

- ticket: 这里的 ticket 字段就是 ST.

- authenticator: 就是 Server Session Key 加密的客户端信息

### 解密后的数据包

#### ticket

可以看到使用 WEB\$ 帐户信息解密 ST, 解密出 bcf9a2a9 开头的 Server Session key, PAC, 时间戳等信息.

此处的 PAC 权限为 Administrator.

![](images/Pasted%20image%2020221208100125.png)

![](images/Pasted%20image%2020221208100431.png)

#### authenticator

可看到使用 bcf9a2a9 开头的 Server Session key 解密 authenticator.

![](images/Pasted%20image%2020221208100821.png)

## AP-REP

### 原数据包

#### 整体请求内容

服务器收到用户发来的 ST 后, 用自己的 NT Hash 解密 ST 获得 Server Session key, 再用该 Server Session key 解密被加密的客户端信息 (请求的用户名、时间戳等), 并进行校验以及与 ST 中的客户端信息进行比对, 相同则认证通过.

![](images/Pasted%20image%2020221208101217.png)


![](images/Pasted%20image%2020221208101326.png)

### 解密后的数据包

可看到使用 bcf9a2a9 开头的 Server Session key 解密 authenticator.

![](images/Pasted%20image%2020221208101609.png)

**感谢耐心阅读, 文章仅供参考, 本人学艺不精, 不足之处欢迎师傅们指点和纠正!**

## 参考文章

https://myzxcg.com/2021/08/Kerberos-%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90%E4%BA%8C/

https://zer1t0.gitlab.io/posts/attacking_ad/

---

> 作者: TryA9ain  
> URL: https://TryA9ain.github.io/%E7%BA%A6%E6%9D%9F%E5%A7%94%E6%B4%BE/  

