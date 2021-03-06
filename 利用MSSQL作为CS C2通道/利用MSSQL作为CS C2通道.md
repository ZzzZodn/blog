### 利用MSSQL作为CS C2通道

#### 0x00 前言

在多次的攻防演练中，都遇到了DMZ区网段和数据库网段隔离，从DMZ区只能通过数据库端口访问到数据库段的数据库服务器的环境

![](./media/网络隔离.jpg)

这种条件下，想要突破隔离，可以使用套接字复用技术，复用web服务器与数据库服务器端口链接的套接字，利用该套接字传输流量，比如mssql下的mssqlproxy便是利用了该技术。但是在windows下，套接字复用受到API限制，对多线程链接处理不好，导致实战中限制过多，只能单线程执行命令，扫描与RDP无法使用。还能如何利用数据库作为一条通道，使我们能够稳定、多链路的传输流量呢？

#### 0x01 结合CS 扩展C2

CS的扩展C2可以让我们自定义流量协议、传输过程来实现被控制端和teamserver的通信，参考[Cobalt Strike 的 ExternalC2](https://rcoil.me/2019/10/%E3%80%90%E6%B8%97%E9%80%8F%E6%8A%80%E5%B7%A7%E3%80%91Cobalt%20Strike%20%E7%9A%84%20ExternalC2/)。鉴于数据库快速稳定的读写和高性能的I/O功能，我们便可以利用数据库表来作为CS的C2通道介质。当teamserver下发命令时，第三方控制端将命令insert到数据库供第三方客户端select出来。第三方select出命令之后，将其写入到命名管道由beacon读取并执行。命令结果以相同方法传回到teamserver，这样就完成了teamserver与beacon间的通信，能通过数据库端口上线数据库服务器，突破网络隔离。上线流程如下：

![](./media/mssql代理实现.jpg)



1. 控制端连接mssql数据库，根据需求创建所需要的库、id自增表(设置主键id为自增来标志数据data列。server表供控制端写入数据，客户端读出，client表则相反。通过自增ID来区分每次通信的数据。)
2. 控制端将从CS C2处请求到的stage写入到Server表中的data列
3. 客户端根据id从server表中读出stage，并将stage通过进程注入，获得smb beacon。当注入成功，beacon会将一段元数据写入到client表
4. 控制端从client表读出元数据，将其发送到cs c2，完成上线

之后，以此步骤不断循环，mssql便是一条稳定的C2通道。

#### 0x02 优势

利用mssql作为通道，不仅可以打通隔离的网段，由于通信成都是借助sql语句的select、insert操作，还可以十分的隐蔽我们的流量。加上C2对流量自定义处理的灵活性，完全可以规避掉CS的通信流量特征。此外，客户端和控制端落地时不携带payload，在一定程度上也能绕过杀软的静态查杀。即使在传递stage的时候，被杀软识别到stage特征，我们也可以在控制端和C2之间再架设一个中间人的角色，让它将C2来的流量加密之后再发送到控制端，以此来绕过杀软的静态检测。