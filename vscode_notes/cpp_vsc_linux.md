### Linux上VSCode部署以及C++编程
#### C++ Programming on Linux with VSCode

### 核心问题
1. 企业独立服务器网络(无外网连接)。由于企业中常常使用的是独立的局域网，因此有一些自动安装操作无法正常完成，需要手工进行。  
2. 凡事遇上C++都很麻烦。大型项目需要注意C++工具的配置，容易出错且找不到问题。


#### 太长不看版：
0. 目标：在服务器上保存代码、管理代码，用PC上的VSCode编辑它们，并用上Intellisense辅助编写代码。实现编辑环境即运行环境；
1. 使用VSCode配合Remote-SSH插件，在服务器上安装vscode服务；注意点：需要额外开一个WSL，然后手工导入安装包到服务器，并给某些可执行文件执行权限；
2. 安装cpptools并正确配置它；特别注意`includePath`，尤其需要弄清楚gcc/g++所用的标头到底在哪里(`gcc -print-prog-name=cc1plus` -v)


#### 开发环境：  
- 大型C++ Linux服务端项目，基于Makefile构建
- CentOs 6.5
- gcc 4.9.2
- vscode 1.37
- cpptools 0.25.0
- Remote Development 0.16.0  (Remote-SSH 0.45.5)
  
#### 0. 在PC上安装好VSCode
https://code.visualstudio.com/  
安装必须的插件：Remote Development; C/C++(cpptools)

#### 1. 配置Remote-ssh
配合Remote-ssh[教程](https://code.visualstudio.com/docs/remote/ssh#_getting-started)，弄好服务器的ssh配置，可能需要预备端口转发（企业网络经常需要）。登录。  
如果你的服务器能够连接上互联网，那么就照着教程，会自动在服务器上下载安装vscode-server，接着一般就没太大问题了。但是很多企业网络是不联通互联网的，于是就需要**搬运vscode-server到服务器上**。

#### 2. 搬运vscode-server
首先，会报错：
```
.....
Downloading with wget
ERROR: (某种网络错误)
```

所以我们需要曲线救国。先使用Remote-WSL (也在Remote Development包里)，在PC机上自动构建一个[WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux)虚拟机，它能联网，也能够正确安装vscode-server。  
然后，我们在terminal中登录到WSL上，找到：`~/.vscode-server`这个目录，打包带走，然后手动上传到真正的服务器上，放在`~`目录下，即可完成安装。  
我们可以简单看一下它包含的内容：  
- `bin` 下面还有一个hash过的目录，这个便是vscode本体。需要注意的是这个本体是按照VSCode版本hash的，因此不必担心不同机器不同时候安装的bin不一样，只要同一个VSCode版本，一定对应同一个bin目录，这也是为什么我们可以从WSL上打包带走放到服务器上能跑通的原因。但是如果VSCode更新了，那么这个hash就不一样了，我们需要重新进行一次上述流程
- `extensions` 插件，插件倒和本体不一样，可以直接用VSCode在服务器上安装(即使无互联网连接的服务器)，Remote-SSH下，在插件页面插件会显示`Install on remote XXX`，意思就是可以在服务器上安装。安装为remote模式之后插件就是在服务器上而不是PC机上运行了，像cpptools这种耗资源大户在使用体验上会有很大提升。不过我们需要用的cpptools还是需要手动安装(并不难)。

搬运完文件之后，我们得注意一个问题，就是搬运过程中有部分可执行程序的执行权限被弄丢了，因此我们需要给手动加上。我们可以看OUTPUT里的日志和弹窗来定位那些东西出了问题。目前这个版本已知的有：
```
.vscode-server/bin/xxxxxxx/bin/code    # vscode本体
.....(作者也记不清了，遇到有功能用不了看错误弹窗即可)
```

#### 3.安装插件
多数插件可以直接用VSCode在服务器直接上。C/C++ Extension比较特别，需要我们从github上手动下载vsix文件，链接(https://github.com/microsoft/vscode-cpptools/releases) 然后从Palette上执行`Install From vsix...`安装它。注：不需要上传到服务器。  
安装好之后我们会在`extensions`目录下看到它，另外运行的时候还会看到`~/.cpptools`目录(路径可配置)


#### 4.配置工程属性和C/C++ Extension
一般我们建议把工程弄成vscode的workspace，比较方便。需要配置的配置文件是`xx.code-workspace`管理工程属性，和`.vscode/c_cpp_properties.json`管理cpptools配置。有Remote-SSH后vscode的配置体系变成了user - remote -  workspace - folder 注意配置不要相互冲突。cpptools配置从近期一个版本开始已经有了UI，可以配合UI上的说明食用。  

工程属性：  
可以参照vscode的配置说明根据自己习惯配置；这里推荐设置一下`files.exclude`，考虑到可能会在代码的位置编译、甚至运行，会有很多`.o/.d/.log`的中间文件，可以适当屏蔽，例如`"**/*.o": true`。另外由于代码一般会有git管理，debug建议另开一个目录，也可以更灵活的进行程序配置、日志、调试等。作者目前的项目一般在Terminal中手动执行make以及手动执行项目程序调试，尚未尝试使用gdb配置cpptools调试，故编译和调试部分尚待研究(参考`launch.json, tasks.json`)。

C/C++ Extension 配置：  
cpptools的配置允许有多个，即`configurations`是个列表。我们配一个Linux的配置，默认选择它就行。配置项如下：
```
"compilerPath": "/usr/local/bin/g++",
"cStandard": "c11",
"cppStandard": "c++14",
"intelliSenseMode": "gcc-x64"
```
指定编译器。Intellisense是依赖于编译器的，因此需要指定一个。Linux上我们就指定服务器上的g++了，也省的win/msvc和linux/gcc有着千差万别。

```
"editor.quickSuggestions": true,
"C_Cpp.intelliSenseEngine": "Default",
"C_Cpp.autocomplete": "Default",
"C_Cpp.intelliSenseEngineFallback": "Disabled",
```
Intellisense和辅助编辑工具的配置。注意这几项需要在.workspace中配置。特别注意`intelliSenseEngine`一项。MS家的Intellisense是分两种模式的：`Default`和`Tag Parser`。`Default`模式是基于编译器编译的，它比较准确，能知道某个类型有什么成员、什么方法，给出的suggestions和define jump也都靠谱，但前提是能够正确调用编译器并成功编译大部分代码；如果由于各种问题没法编译代码，会降级为`Tag Parser`模式，这个模式只是简单的单词匹配，不准确，给出的suggestions基本不能看，但define jump还行，如果同名会peek出多个可能的结果让用户选择。  
`C_Cpp.autocomplete`和`editor.quickSuggestions`是是否调用辅助提示的，通常我们都给打开。  
`C_Cpp.intelliSenseEngineFallback`是指如果发现#include错误是否降级为`Tag Parser`模式。如果总是有include头文件引用问题的话可以尝试打开，但我们尽量把工具配置对，用上`Default`模式。


```
"includePath": [
    "工程文件目录",
    "工程依赖包目录",
    "标准头文件目录",
    .....
]
```
`includePath`如其名。需要注意的是我们需要显式的配置gcc的标准头文件，否则Intellisense会找不到标准库里的东西，会各种提示#include的错误，并且给不出可靠的suggestions。

```
"browse": {
    "path": [
        "${workspaceFolder}/dev/src",
        "${workspaceFolder}/dev/src/proto/generatedcpp",
        "/xapian_core/include",
        "/usr/local/include/c++/4.9.2",
        "/usr/local/include/c++/4.9.2/x86_64-unknown-linux-gnu",
        "/usr/local/include/c++/4.9.2/backward",
        "/usr/local/lib/gcc/x86_64-unknown-linux-gnu/4.9.2/include",
        "/usr/local/include",
        "/usr/local/lib/gcc/x86_64-unknown-linux-gnu/4.9.2/include-fixed" 
    ],
    "limitSymbolsToIncludedHeaders": true,
    "databaseFilename": ""
}
```
`Tag parser`使用的检索范围。


都配置好之后，我们就可以用Intellisense辅助编写代码了。尝试一下define jump和suggestions，以及实时编译吧！  
不过大型项目中辅助工具还是有延迟，可以看到右下角的小火焰就是Intellisense还在跑，毕竟编译是件很耗时耗资源的事。这也是为什么我们希望能在服务器上而不是PC机上运行cpptools的原因。

