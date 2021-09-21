### WMIRM利用

#### 0x00 wmirm

Windows远程管理（WinRM）是[WS-Management Protocol](https://docs.microsoft.com/en-us/windows/win32/winrm/ws-management-protocol)的Microsoft实现。WS-Management协议是一种基于SOAP的防火墙友好协议，旨在用于系统查找和交换管理信息。WS-Management协议规范的目的是为企业系统提供互操作性和一致性，其对防火墙友好的协议，允许来自不同供应商的硬件和操作系统进行互操作。借助winrm.exe和powershell，管理员可以使用WS-Management协议远程执行大多数Cmd.exe命令、获取远程机器信息。在内网中，可以借助WinRM来进行横向移动或者后门驻留。从Vista开始，WimRM成为widnows默认组件。其在2008开始，作为默认服务启动，但默认不开启监听模式，因此无法接收、发送数据。`Winrm quickconfig`命令可快速使用默认设置配置WinRM

​	![](/Users/cate4cafe/工作/文章/WinRM/1.jpg)

查看WinRM当前监听情况

​	![](/Users/cate4cafe/工作/文章/WinRM/2.jpg)

默认监听5985端口(HTTP)，如果配置了HTTPS，则默认监听5986端口。配置HTTPS需要自签名证书，以IIS证书为例，安装IIS后，powershell查看证书位置

`ls Cert:\LocalMachine\My\`

配置

`winrm create winrm/config/Listener?Address=*+Transport=HTTPS @{Port="5986" ;Hostname="WMSvc-SHA2-WIN-98R1JPLVTE0" ;CertificateThumbprint="DFB6B1B54FBC56D6A80C42E7E8EC3DBE4A9D3F4E"}`

查看结果

![](/Users/cate4cafe/工作/文章/WinRM/5.jpg)

查看当前配置

![](/Users/cate4cafe/工作/文章/WinRM/3.jpg)

默认情况下，WinRM只允许域内机器连接。

![](/Users/cate4cafe/工作/文章/WinRM/6.jpg)

只有设置 `TrustedHosts`或者设置了HTTPS之后，其他机器才可以连接到WinRM。设置信任主机

`winrm set winrm/config/client @{TrustedHosts="*"}` *表示允许任意机器连接



### 0x01 横向

本地也需设置信任主机

- **使用powershell**

![](/Users/cate4cafe/工作/文章/WinRM/7.jpg)



执行PS代码块

```powershell
$cred=get-Credential  #创建凭据
$session1 = new-pssession -computername 172.16.233.193 -Credential $cred #创建Session，指定多台主机可同时再多台机器执行
Invoke-Command -Session $session1 -ScriptBlock {$a="hello world"}
Invoke-Command -Session $session1 -ScriptBlock {$a}  #执行脚本块
```

![](/Users/cate4cafe/工作/文章/WinRM/9.jpg)

执行PS脚本

![](/Users/cate4cafe/工作/文章/WinRM/10.jpg)

- **CMD**

  ![](/Users/cate4cafe/工作/文章/WinRM/8.jpg)

分析执行过程。

wireshark抓包，可以看见使用http协议请求了/wsman

![](/Users/cate4cafe/工作/文章/WinRM/11.jpg)

继而使用NTLM进行身份验证，然后post了带有xml序列化数据的包，/wsman接收到后反序列化xml，执行了包里携带的payload。

![](/Users/cate4cafe/工作/文章/WinRM/12.jpg)

在sysmon记录的日志中可以看到目标机器调用了winrmhost.exe去执行了命令

![](/Users/cate4cafe/工作/文章/WinRM/13.jpg)



#### 0x03 作为持续控制后门

在IIS监听80端口的情况下，可以通过设置WinRM监听端口为80，再设置监听URI的方式来复用80端口。以此作为隐蔽的后门。

```
winrm set winrm/config/Listener?Address=*+Transport=HTTP @{Port="80"}
winrm set winrm/config/Listener?Address=*+Transport=HTTP @{URLPrefix="backdoor"}
```

此时，依然能访问到IIS服务

![](/Users/cate4cafe/工作/文章/WinRM/14.jpg)

也能访问到WinRM服务并执行命令

![](/Users/cate4cafe/工作/文章/WinRM/15.jpg)