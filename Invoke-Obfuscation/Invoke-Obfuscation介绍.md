# Invoke-Obfuscation介绍

项目地址：[Invoke-Obfuscation](https://github.com/danielbohannon/Invoke-Obfuscation)

该项目混淆技术来隐藏powershell.exe命令行参数中的大部分命令。

导入项目

![](A:\文章\Invoke-Obfuscation\media\2019-11-25-1.png)

支持的加密功能列表

![](A:\文章\Invoke-Obfuscation\media\2019-11-25-2.png)

TOKEN支持分部分混淆，STRING整条命令混淆，COMPRESS将命令转为单行并压缩，ENCODING编码，LAUNCHER选择执行方式。

## 使用

- **set scriptblock / scriptpath**

  指定要混淆的代码块或者脚本路径

- **undo**

  取消上一次混淆

- **back**

  返回上一层菜单

- **out "path"**

  混淆结果输出到文本

- **token**

  对字符进行混淆

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-27-1.png)

  对命令混淆

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-27-2.png)

  对参数混淆

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-27-3.png)

  

​    使用ALL 混淆

![](A:\文章\Invoke-Obfuscation\media\2019-11-27-4.png)

- **STRING**

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-27-5.png)

- **Encoding**

  支持base64、hex等编码

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-28-6.png)

![](A:\文章\Invoke-Obfuscation\media\2019-11-28-7.png)

- **COMPRESS**

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-28-8.png)

- **LAUNCHER**

  ![](A:\文章\Invoke-Obfuscation\media\2019-11-28-9.png)

![](A:\文章\Invoke-Obfuscation\media\2019-11-28-10.png)