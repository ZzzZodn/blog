### Atexec命令执行原理分析

#### 前言

Atexec是impacket套件中用于横向移动、执行命令的组件。

- 执行命令

  ![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/截屏2020-02-10上午10.26.08.png)

- 横向移动

  利用FOR命令pass the hash

  `FOR /F %%i in (ips.txt) do atexec.exe -hashes :xxxxxxxxxxxxx ./administrator@%%i  whoami `

  `FOR /F %%i in (hashes.txt) do atexec.exe -hashes %%i ./administrator@192.168.145.11  whoami  `

  同理，也可以用密码明文去碰撞内网机器



#### 前置知识

- SMB命令

  ```
  Tree -- 用于建立到服务器共享的客户端连接 
  Create -- 用于创建和打开新文件\命名管道
  Read -- 请求对FileId指定的文件进行读取操作
  Write -- 将数据写入服务器上的文件或命名管道
  ```

  

- IPC$、ADMIN$

  ```
  IPC$ -- 共享"命名管道"的资源，为了让进程间通信而开放的命名管道，通过提供可信任的用户名和口令，连接双方可以建立安全的通道并以此通道进行加密数据的交换，从而实现对远程计算机的访问
  ADMIN$ -- 可以认为是路径C:\Windows的符号链接
  ```

  

- ATSVC

  ```
  Microsoft AT调度程序服务。通过SMB进行身份验证后，可以从IPC$上的\PIPE\atsvc命名管道访问此服务
  ```

  

#### 流程分析

看atexec执行成功的输出，可以知道大概流程是创建一个远程任务--执行--结果输出到临时文件--删除任务。

Atexec使用smb协议，这里使用wireshark抓取执行时的SMB流量以作分析。

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/WX20200210-154551@2x.png)

第一步，协商SMB版本号。客户端发送SMB negotiate protocol request数据报，列出支持的SMB协议，协商使用的SMB版本

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/1-1.png)

目标收到协商后，返回使用的SMB版本号

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/1-2.png)

版本协商完毕，开始进行认证。

第二步，认证开始时，使用Microsoft Negotiate选择认证方式（kerberos、NTLM），Windows会优先使用安全性高的kerberos协议，只有当kerberos协议不可用时才会使用NTLM协议。

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/2-1.png)

使用NTLM 挑战/响应数据报中包含了用户、计算机名、NTLM hash等信息

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/2-2.png)

　　

第三步，验证通过后，使用Tree命令连接目标IPC$。

第四步，连接成功后，绑定\PIPE\atsvc管道，通过atsvc管道访问任务调度服务，使用[SchRpcRegisterTask](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tsch/849c131a-64e4-46ef-b015-9d4c599c5167)注册任务，[SchRpcRun](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tsch/77f2250d-500a-40ee-be18-c82f7079c4f0)执行任务，[SchRpcDelete](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tsch/360bb9b1-dd2a-4b36-83ee-21f12cb97cff)删除任务。

在atexec.py的源码中，可以在创建任务详情的xml中发现是是通过cmd执行我们的命令，把命令结果写入到临时文件中。之后，再通过SMB Read命令去读取该临时文件的内容。

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/3-1.png)

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/3-2.png)

在TCP流中可以看到Read的内容

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/3-3.png)

在目标机器上，通过sysmon、process Monitor可以看到更详尽的内容

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/4-1.png)

​					(任务文件和临时文件的创建)

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/4-2.png)

（任务注册、执行、删除）

在sysmon看到执行的命令

![](/Users/cate4cafe/工作/文章/Atexec命令执行原理分析/media/4-3.png)