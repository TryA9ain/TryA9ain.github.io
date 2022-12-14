# Kerberos 认证过程分析


本文主要记录了通过 WireShark 解开 Kerberos 协议对其进行分析, 很多东西不仅要知其然, 更要知其所以然. 

<!--more-->

## Kerberos 主要步骤

1. AS-REQ
2. AS-REP
3. TGS-REQ
4. TGS-REP
5. AP-REQ
6. AP-REP

这里重点分析前四点.

![](images/Pasted%20image%2020221105212043.png)

## AS-REQ

### 整体请求内容

![](images/Pasted%20image%2020221116215126.png)

- pvno: Kerberos 的版本号, 可以看到是 Kerberos 5.
- msg-type: 消息类型, 这里是 krb-as-req.
- padata: Pre-authentication Data, 预认证数据 (我也不知道怎么翻译 🤣) 里面存放的是一些认证信息.
- req-body: 请求体.

### padata

![](images/Pasted%20image%2020221116215843.png)

- padata-type: [pA-PAC-REQUEST](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-kile/765795ba-9e05-4220-9bd3-b34464e413a7) 一种 padata 类型, 明确请求在票据中包含或排除 PAC.

	padata 的类型有很多种, 可以参考这篇文章: [Kerberos PA-DATA](https://cwiki.apache.org/confluence/display/DIRxPMGT/Kerberos+PA-DATA).

- include-pac: True 包含PAC.

	[PAC](#pac) (Privilege Attribute Certificate) 是一个做权限认证的扩展.

### req-body

![](images/Pasted%20image%2020221116221033.png)

- kdc-options: 用于与 KDC 约定一些选项设置.
- cname: 客户端用户名.
- realm: 域名.
- sname: 服务端用户名, 在 AS-REQ 中 sname 是 krbtgt, 类型是 kRB5-NT-SRV-INST.
- till: 到期时间.
- nonce: 随机生成的一个数, 用于检测重放攻击.
- etype: 协商加密类型, KDC 通过 etype 中的加密类型来选择用户对应的 NT Hash 来解密.

## KRB Error: KRB5KDC_ERR_PREAUTH_REQUIRED

![](images/Pasted%20image%2020221116223927.png)

第一个AS-REQ 对应的响应信息为 "KRB Error: KRB5KDC_ERR_PREAUTH_REQUIRED".

这是因为在 Kerberos 5 之前, Kerberos 允许不使用密码进行身份认证, 而在 Kerberos 5 中, 密码信息是必须的, 这种过程称之为 "预认证".

可能出于向后兼容考虑, Kerberos 在执行预认证之前首先会尝试不使用密码进行身份认证 (从第一个 AS-REQ 包中能看到并不包含密码信息), 因此 Kerberos 5 在登录期间发送初始 AS-REQ 后我们总是能看到一个错误信息.

预认证过程不包含密码信息, 接下来才是正常的 Kerberos 认证.

## AS-REQ

用户输入帐号密码访问域内的服务, 本机会向 KDC 的 AS 认证服务发送一个 AS-REQ 请求, 请求主要包含用户 NT Hash 加密的时间戳, 请求用户名, 协商 NT Hash 的加密类型等信息.

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221116224907.png)

- pvno: kerberos 的版本号.
- msg-type: 消息类型, 这里就是 krb-as-req.
- padata: Pre-authentication Data, 预认证数据 (我也不知道怎么翻译 🤣) 里面存放的是一些认证信息.
- req-body: 请求体.

#### padata

![](images/Pasted%20image%2020221116225023.png)

- PA-DATA PA-ENC-TIMESTAMP: 用户 NT Hash 加密后的时间戳
	- padata-type: padata 类型
		- padata-vaule: padata 的值
			- etype: 加密类型
			- cipher: 加密后的值
- PA-DATA PA-PAC-REQUEST: PAC扩展
	- padata-type: padata 类型
		- padata-vaule: padata 的值
		- include-pac: 是否包含 PAC, True 为包含, False 为不包含.

#### req-body

![](images/Pasted%20image%2020221116225341.png)

- kdc-options: 用于与 KDC 约定一些选项设置.
- cname: 客户端用户名.
- realm: 域名.
- sname: 服务端用户名, 在 AS-REQ 中 sname 是 krbtgt, 类型是 kRB5-NT-SRV-INST.
- till: 到期时间.
- nonce: 随机生成的一个数, 用于检测重放攻击。
- etype: 协商加密类型, KDC 通过 etype 中的加密类型来选择用户对应的 NT Hash 来解密。

### 解密后的数据包

>由于有的数据包过大, 贴完整的数据包出来意义不是特别大, 本文只贴一些重点的解包.

#### PA-DATA PA-ENC-TIMESTAMP

![](images/Pasted%20image%2020221116230040.png)

WireShark 对于每一个被解密的部分都进行了说明是使用 keytab 中的哪个主体进行解密的. 

可以看到 PA-DATA pA-ENC-TIMESTAMP 是使用了 keytab 中 ligang 的信息来成功解密出时间戳.

## AS-REP

KDC 收到请求后, 通过活动目录查询到该用户的密码 NT Hash, 用该 NT Hash 对请求包的 pA-ENC-TIMESTAMP 进行解密, 解密成功后, 还会检查时间戳, 如果时间戳的范围在五分钟内且数据包无重放, 则认证成功.

认证成功后, KDC 会向客户端返回两个东西:

- 用户 NT Hash 加密后的 Logon session key (AS 随机生成).
- krbtgt NT Hash 加密后的 TGT.

TGT 主要包含 Logon session key, 时间戳和 PAC.

Logon session key 的作用是作为用户和 KDC 后几个阶段之间通信加密的会话密钥.

### 原数据包

#### 整体相应内容

![](images/Pasted%20image%2020221116230803.png)

主要来看 ticket 和 enc-part 这两部分:

- ticket: 即 TGT.

- enc-part: 这部分是用户 NT Hash 加密的, 里面包含 Logon session key.

#### ticket

这里的 ticket 其实就是 TGT.

![](images/Pasted%20image%2020221116231127.png)

- tkt-vno: TGT 版本号.
- realm: 域名.
- sname: 请求的服务名称 krbtgt.
- enc-part: 这部份是用 krbtgt NT Hash 加密的, 其中包含了请求用户的 [PAC](#pac) 信息.

#### enc-part

![](images/Pasted%20image%2020221116231314.png)

内容是由用户 NT Hash 加密的 Logon session key.

### 解密后的数据包

#### ticket 中的 enc-part

可以看到是用 krbtgt 的信息解密的, 解密出 Logon session key, 时间戳, [PAC](#pac) (authorization-data 字段里的内容, 这里放到了后面来介绍).

![](images/Pasted%20image%2020221118231202.png)

#### enc-part

用 ligang 用户信息解密出 Logon session key, 可以看到与 TGT 中的 Logon session key 是相同的.

![](images/Pasted%20image%2020221118231746.png)

## TGS-REQ

用户通过 AS-REP 拿到 TGT 和用户 NT Hash 加密后的 Logon Session Key, 使用用自己的 NT Hash 解密获得 Logon session key, 再用 Logon session key 加密客户端用户名, 时间戳等信息和 TGT 一起向 KDC 的 TGS 服务发起请求请求 ST. (这一步不需要帐号或者密码主要以 TGT 作为凭证).

### 原数据包

#### 整体请求内容

![](images/Pasted%20image%2020221117212656.png)

- padata: 这个部分包含之前的 AS-REP 中的 TGT 以及 Logon session key 加密的内容.
- req-body: 请求体.

#### padata

![](images/Pasted%20image%2020221117213109.png)

- AP-REQ: 这部分会携带 AS-REP 中获取到的 TGT, 客户端的认证信息.
	- ticket: 即 TGT.
	- authenticator: Logon session key 加密的客户端信息 (请求的用户名, 时间戳).

#### req-body

![](images/Pasted%20image%2020221117213246.png)

- realm: 域名.
- sname: 请求的指定服务.
- etype: 支持的加密类型.

### 解密后的数据包

#### padata

- padata --> ap-rep --> ticket

	就是 AS-REP 中获取到的 TGT, TGT 解包前面已经解过了, 此处不在详细分析.

- padata --> ap-rep --> authenticator

	这部分就是 Logon session key 加密的客户端信息 (请求的用户名, 时间戳), 可以看到是由 AS-REP --> enc-part --> Logon session key (用户 NT Hash 加密的 cf4832b1 开头的值) 解密出来的.
	<!-- 可以看到是由 AS-REP 中 enc-part 中的 Logon session key (用户 NT Hash 加密的 cf4832b1 开头的值) 解密出来的. -->

![](images/Pasted%20image%2020221117215326.png)

## TGS-REP

KDC 在收到上述请求后, 会用 krbtgt 的 NT Hash 解密提取 TGT 并从 TGT 中获取用户名, Logon session key 和时间戳, 接着再用 Logon session key 去解密 authenticator 获取到请求的用户名和时间戳, 将两处解密的用户名和时间戳进行对比, 并确认 TGT 是否还在生命周期内, 如果对比相同, 且 TGT 还在生命周期内, 则颁发 ST.

至此客户端 和 KDC 之间的通信已全部结束, 下面就该是客户端和服务端之间的交互了。

ST (服务票据) 中包含 Server Session key, 客户端名称, 票据有效时间, PAC 等信息.

### 原数据包

#### 整体响应内容

![](images/Pasted%20image%2020221117220321.png)

- ticket: 这个就是 ST, 由服务帐号的 NT Hash 加密.
- enc-part: Logon session key 加密的 Server Session key 等信息.

#### ticket

![](images/Pasted%20image%2020221117221523.png)

这个就是 ST, 由服务帐号的 NT Hash 加密.

#### enc-part

![](images/Pasted%20image%2020221117221549.png)

Logon session key 加密的 Server Session key 等信息.

### 解密后的数据包

#### ticket

可以看到是用 WIN-2008$ 服务帐号信息解密的.

- key: 这里 73950060 开头的 key 就是 Server Session key.
- authorization-data: 用户的 [PAC](#pac).

![](images/Pasted%20image%2020221117222619.png)

#### enc-part

可以看到依旧是用 cf4832b1 开头的 Logon session key 解密出来的, 并且 Server Session key 和 ST 票据中的相同.

剩下的就是: 拿着 ST 去进行 AP-REQ 和 AP-REP 阶段了.

![](images/Pasted%20image%2020221117223247.png)

## PAC

PAC (Privilege Attribute Certificate, 特权属性证书), 其中所包含的是各种授权信息, PAC 用来表示用户的权限, 如果没有 PAC, Kerberos 协议就仅能证明用户身份, 无法表示用户权限.

TGT 和 ST 中的 authorization-data 字段里的内容就是 PAC.

有PAC 的请求流程:

- 用户发起 AS-REQ, KDC 从 AS-REQ 请求中取出 cname 字段并查询活动目录, 找到 sAMAccountName 属性为 cname 字段的值的用户, 用该用户的身份生成一个对应的 PAC. 用户也可以指定 include-pac: False/True 来要求 KDC 在返回的票据中是否包含 PAC.

- 用户发起 TGS-REQ, KDC 收到 TGT 后, 利用 krbtgt 密钥对其解密并验证 PAC 的签名. 如果签名正确, 则证明 PAC 未经过篡改, 然后在 PAC 的尾部重新生成两个签名: Server Signature (要请求的服务的 NT Hash 生成的签名) 和 KDC Signature. 然后把新的 PAC 拷贝到 ST 中. (此步骤不校验 PAC 中用户有没有访问服务的权限, 只要 TGT 正确就返回 ST).

	>如果是 S4U2self 请求的话, ST 中的 PAC 是重新生成的 (生成的是模拟用户的 PAC).

- 用户拿着 ST 去请求服务, 服务使用 NT Hash 解密 ST 获取 PAC 后检查用户的 SID (UserId), 所在组 (GroupIds) 并与自身的 ACL 进行对比, 判断用户是否有访问服务的权限. (如果服务开启了验证 PAC 的签名, 还会向 KDC 发送 KERB_VERIFY_PAC 验证 PAC 的签名).

>如果 TGT 中不带有 PAC, 那么 ST 中也不会带有 PAC, 也就没有权限访问任何服务.

### 解密后的数据包

#### 普通域用户的 PAC

主要来看下 Type: Logon Info (1), 这个结构是登录信息, 主要靠它来验证用户身份.

![](images/Pasted%20image%2020221105224416.png)

Type: Logon Info (1) 展开主要注意:

- Acct Name: ligang

- GroupRIDs
	- Group_MEMBERSHIP:
		- Group RID: 513

表示 ligang 这个帐户在 Group RID: 513 (域用户组) 中.

![](images/Pasted%20image%2020221118215602.png)

#### 域管的 PAC

- Acct Name: yunwei

- GroupRIDs
	- Group_MEMBERSHIP:
		- Group RID: 513
	- Group_MEMBERSHIP:
		- Group RID: 512

可以看到 yunwei 这个帐户在 Group RID: 513 (域用户组) 和 Group RID: 512 (域管组) 中, 所以 yunwei 是域管.

![](images/PAC.png)

**感谢耐心阅读, 文章仅供参考, 本人学艺不精, 不足之处欢迎师傅们指点和纠正!**

## 参考文章

https://loong716.top/posts/Kerberos/

https://myzxcg.com/2021/08/Kerberos-%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90%E4%B8%80/

---

> 作者: TryA9ain  
> URL: https://TryA9ain.github.io/kerberos%E8%AE%A4%E8%AF%81%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/  

