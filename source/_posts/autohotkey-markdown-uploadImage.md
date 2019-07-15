title: AutoHotkey&qshell实现图片一键上传七牛并返回markdown引用（适用1.x版本）
categories: Tools
tags: [AutoHotkey]
toc: false
date: 2016-08-30 11:41:56
---

**qimage** 是一个提升 markdown 贴图体验的实用小工具，支持 windows 及 mac。其中 **[qiniu-image-tool-win](https://github.com/jiwenxing/qiniu-image-tool-win)** 为windows版本，基于`AutoHotkey`和`qshell`实现，可以自定义快捷键，一键上传图片或截图至七牛云，获取图片的markdown引用至剪贴板，并自动粘贴到当前编辑器。<!--more-->

> 注意此文档只适用于1.x版本，2.x及以上版本请参考[windows版本markdown一键贴图工具](//jverson.com/2017/05/28/qiniu-image-v2/)。


## 用法
使用方法很简单，只需两步即可完成图片的上传和使用，[github](https://github.com/jiwenxing/qiniu-image-tool-win) 有预览动图：

1. 复制本地图片、视频或js等其它类型文件至剪贴板（ctrl+c）or 使用喜欢的截图工具截图 or 直接复制网络图片
2. 切换到编辑器，`ctrl+alt+v`便可以看到图片链接自动粘贴到当前编辑器的光标处

如果您使用mac，请参考[使用alfred在markdown中愉快的贴图](//jverson.com/2017/04/28/alfred-qiniu-upload/)

## 安装
### 下载源码
相对于 mac 版本，windows 版安装更加简单。首先从 [github](https://github.com/jiwenxing/qiniu-image-tool-win/releases) 下载最新的release版本并解压到任意目录，在`qimage-win`文件夹中看到的目录结构应该是如下这样：
![](//
jverson.oss-cn-beijing.aliyuncs.com/afa184acc926d86a6ea786d7634500e7.jpg)
其中`dump-clipboard-png.ps1`是保存截图的`powershell`脚本，`qimage-win.ahk`即完成文件上传的`AutoHotkey`脚本。

### 安装 **AutoHotkey**    
文件夹中`AutoHotkey_1.1.25.01_setup.exe`是1.1.25版本的`AutoHotkey`安装包，您可以直接双击安装，也可以去[AutoHotkey官网](https://autohotkey.com/)下载安装最新版本，这是一款免费的、Windows平台下开放源代码的热键脚本语言，利用其通过自定义热键触发一系列系统调用从而完成自动化操作。

### 注册七牛账号并创建一个bucket   
[七牛](https://www.qiniu.com/)是一个云服务提供商，很多个人博客现在都喜欢用七牛的对象存储服务做图床，速度确实不错，有比较完整的文档和开发工具，另外实名以后有10G的免费空间使用，基本上满足使用。


### 配置脚本
文件夹中选中`qimage-win.ahk`文件，右键选择编辑脚本使脚本在编辑器中打开，找到下面这段代码:
         
```autohotkey
;;;; config start, you need to replace them by yours
ACCESS_KEY = G4T2TrlRFLf2-Da-IUrHJKSbYbJTGpcwBVFbz3D
SECRET_KEY = 0wgbpmquurY_BndFuPvDGqzlfWHCdL8YHjz_fHJ
BUCKET_NAME = fortest  ;qiniu bucket name
BUCKET_DOMAIN = //
jverson.oss-cn-beijing.aliyuncs.com/  ;qiniu domain for the image
WORKING_DIR = E:\TOOLS\qiniu-image-tool-win\  ;directory that you put the qshell.exe 
;;;; config end
```

修改这里的五个配置项的值，其中前四个配置项都与七牛账号相关：  

#### ACCESS_KEY & SECRET_KEY
这是qshell操作个人账号的账号凭证，登陆七牛账号后在`个人面板->密钥管理`中查看，或者直接访问`https://portal.qiniu.com/user/key`查看。   

#### BUCKET_NAME & BUCKET_DOMAIN    
在`对象存储->存储空间列表`中选择或新建一个存储空间即bucket，点击该bucket在右边看到一个测试域名，该域名即bucketDomain是图片上传后的访问域名。这里要特别注意域名不要少了前面的http头和最后的那个斜杠。

查看以上四个参数的操作如下图所示：
![](//
jverson.oss-cn-beijing.aliyuncs.com/883c2cf5633cac4fba7b3719284ab678.gif)

#### WORKING_DIR  
这是设置您的工作目录，即这些脚本所在的目录，比如我将从github上下载的release压缩包解压到了`E:\TOOLS`目录下，那我的`WORKING_DIR`就是`E:\TOOLS\qiniu-image-tool-win\`。注意不要少了最后那个反斜杠。*另外需要特别注意的是路径中不能包含中文，而且不能有类似Program Files这类包含空格的路径。*

## 运行脚本
至此所有的安装和配置过程都结束了，右键点击`qiniu-image-upload.ahk`文件选择运行脚本（Run Script），这时便可以在任务栏看到一个H字母的绿色图标在运行。这时便可以使用`ctrl+alt+v`尝试上传图片了。

## 调试脚本
如果以上操作完成后没有按照预期达到图片上传的效果，感兴趣的筒子可以先自己调试找一下原因，一般报错信息会打印在cmd命令行中，但是cmd窗口一闪而过可能看不清楚，这时候将可选参数`DEBUG_MODE := false`改为`DEBUG_MODE := true`打开调试模式，保存后在右下角任务栏找到运行着的H绿色小图标右键选择`Reload This Script`，再次尝试，这时候cmd窗口不会自动关闭，便可以看到具体的报错信息从而对症下药解决问题。

如果您在使用过程中有任何问题，也欢迎留言或在github中提交[issues](https://github.com/jiwenxing/qiniu-image-tool-win/issues)。

### 常见问题一: 七牛uphost有误
具体的错误信息如下所示：
> Uploading G:\Users\Cooper\Desktop\9999.png => markdown : 201705082057_244.png ...
Progress: 100%
Put file error, 400 incorrect region, please use up-z2.qiniu.com, Reqid: 0wUAAGp6j6faorwU
Last time: 0.43 s, Average Speed: 415.6 KB/s

由于在七牛的[官方文档](https://github.com/qiniu/qshell/wiki/fput)中uphost为非必填项，脚本中使用的是默认值`//up.qiniu.com`，有时当与你空间所在机房不匹配时便会报以上的错误信息，不过错误信息中已经给出了建议的uphost，例如上面的错误信息就给出明确的提示 “请使用up-z2.qiniu.com”，这时将可选配置项`UP_HOST = //up.qiniu.com`修改为`UP_HOST = //up-z2.qiniu.com`，保存并reload脚本即可。

### 常见问题二: powershell执行权限问题
具体的错误信息如下所示：
> set-executionpolicy : 对注册表项“HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PowerShell\1\ShellIds\Micro
soft.PowerShell”的访问被拒绝。 要更改默认(LocalMachine)作用域的执行策略，请使用“以管理员身份运行
”选项启动 Windows PowerShell。要更改当前用户的执行策略，请运行 "Set-ExecutionPolicy -Scope Current
User"。

这是powershell执行权限问题，解决方法请参考[issue#3](https://github.com/jiwenxing/qiniu-image-tool-win/issues/3)，在此感谢Daniels5


## 修改默认项
以下操作非必需，是对一些默认设置的修改，请根据喜好自行选择。

### 修改快捷键
脚本中默认的快捷键是`^!V`，即`ctrl+alt+v`(^代表ctrl，!为alt)，如果您希望修改为其它自己习惯的快捷键，直接修改并reload脚本即可生效。    
关于hotkey的符号与按键对应关系请参考 [You can use the following modifier symbols to define hotkeys](https://autohotkey.com/docs/Hotkeys.htm)


## 参考
- [qshell fput 文件上传指令](https://github.com/qiniu/qshell/wiki/fput)
- [Powershell-Scripts for dumping images from clipboard](https://github.com/octan3/img-clipboard-dump)


<a href="https://github.com/jiwenxing" target="_blank" rel="external"><a href="https://github.com/jiwenxing/qiniu-image-tool-win" title="Fork me on GitHub" class="fancybox" rel="article"><img style="position: absolute; top: 0; left: 0; border: 0;" src="https://camo.githubusercontent.com/121cd7cbdc3e4855075ea8b558508b91ac463ac2/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f6c6566745f677265656e5f3030373230302e706e67" alt="Fork me on GitHub" data-canonical-src="https://s3.amazonaws.com/github/ribbons/forkme_left_green_007200.png"></a><span class="caption">Fork me on GitHub</span></a>
