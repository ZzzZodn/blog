### 内网横向系列——smbexec

#### 0x01

smbexec是impacket套件中使用smb协议用于全交互式工具远程执行命令的一个小工具。可用于rdp等有交互环境登录使用或socks代理环境。需要管理员权限。

#### 0x02

使用介绍：

- 通过密码

  `python smbexec.py ./administrator:Win2008@192.168.0.1`

- 通过NTLM HASH

  `smbexec.exe -hashes:DF92E298362E3E180EC0EE7226AFB825 ./administrator@192.0.1`

- 使用当前用户上下文

  `smbexec.exe -k targetName #kerberos认证 `

#### 0x03

smbexec原理和[atexec](http://greatagain.dbappsecurity.com.cn/#/book?id=658&type_id=1)类似,都是使用rdp协议认证，继而使用smb协议请求一个远程服务开放命名管道，进而利用远程服务启动cmd.exe执行命令。所不同的是，smbexec使用了Dummy Activity来维持网络的连接达到交互shell的效果。依然以wireshark流量来分析执行的过程

![](/Users/cate4cafe/工作/文章/smbexec执行过程/media/1.png)

正常的smb协商之后，请求\svcctl管道。SCM Manager通过命名管道svcctl远程公开，用户连接到目标系统后可以使用命名管道svcctl更改[SCM配置设置](https://docs.microsoft.com/en-us/windows/win32/services/service-control-manager)。这样就可以通过服务来调用cmd.exe执行我们想要的命令。

![](/Users/cate4cafe/工作/文章/smbexec执行过程/media/2.png)

通过SVCTL创建服务

![](/Users/cate4cafe/工作/文章/smbexec执行过程/media/3.png)

在C盘下创建了__output文件，用来做数据中继。使用sysmon可以看到具体的操作

![](/Users/cate4cafe/工作/文章/smbexec执行过程/media/4.png)

之后删除文件，关闭服务。

