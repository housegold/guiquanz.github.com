---
layout: post
title: 对Lua的一些简单改进
date: 2012-08-02
categories:
    - 技术
tags:
    - lua
---
## 引子

备注：本文仅针对[Lua官方](http://www.lua.org)发布的`lua-5.2.1`版本。以下假设你的代码目录是`~/lua-5.2.1`。

`Lua`是一门非常简单、灵活的脚本语言。整体实现也非常精巧，有效`c代码不过1多万行`。以下介绍一些其中可以进一步完善的简单事项（复杂的问题，建议通过c扩展或lua的模块实现。除非你在使用中发现特殊的`bug`）。

## 生成动态库文件

在官方发布的lua版本中没有提供编译生成动态库（或共享库）的支持。编译之后，默认生成`liblua.a`静态库文件。当想将lua作为`C/C++`的嵌入式扩展时，编译软件需要连接共享库版本的lua库，所以需要对`Makefile`文件进行修改。涉及修改的内容也非常简单，具体流程如下：

1.编辑`~/lua-5.2.1/Makefile`文件，修改`TO_LIB`变量的值。修改后变量值，如下：

<pre class="prettyprint linenums">
TO_LIB= liblua.a liblua.so
</pre>

2.编辑`~/lua-5.2.1/src/Makefile`文件：

（1）.找到`LUA_A`变量，在后面追加一行`LUA_SO`相关的内容。追加的具体内容，如下：

<pre class="prettyprint linenums">
LUA_SO=	liblua.so
</pre>

（2）.将`LUA_SO`变量值增加到`ALL_T`中。修改后`ALL_T`的值，如下：

<pre class="prettyprint linenums">
ALL_T= $(LUA_A) $(LUA_T) $(LUAC_T) $(LUA_SO)
</pre>

（3）.增加动态库相关的编译`target`。具体内容，如下：

<pre class="prettyprint linenums">
$(LUA_SO): $(BASE_O)
	$(CC) --shared -o $@ $?
</pre>

这个编译项可以追加到`“$(LUA_A)”target`之后（在两个`target`中间加一个空行，便于阅读）。
进行以上修改之后，我们的lua版本支持动态库的编译了（至少支持`gcc`编译环境）。生成的文件是`~/lua-5.2.1/src/liblua.so`。至于安装涉及的操作系统对动态库版本的管理因平台而异，请依据具体平台的要求进行处理，如果你很在乎。

建议：当然，编译之后可以不安装，开发使用本地的版本。因为有些系统工具可能依赖于旧版本的lua，升级为新版本之后，这些工具可能就不能正常运行了。笔者在`Fedora 17`上就碰见过类似问题。安装`lua-5.2.0`版本之后，系统工具`rpm`及`yum`都崩溃了，只好重新安装了一个`lua-5.1.0`的版本。当然有很多方法可以解决这类问题。为了避免类似情况，建议在产品开发中使用本地的`lua`版本，采用类似`redis`新版本（2.6.0：尚未发布）的模式，便于管理、维护。

估计`lua`官方不提供动态库的编译，大概就是因为不同平台对动态库的管理模式不同罢了。要做到完全的跨平台很难，而且相关代码也会很复杂。不如，让需要的人自己处理好了。（令缺毋烂，简单原则）

特殊说明：以上的修改其实是一个缺乏可移植性的方案，至少用`HP-UX`的`cc/aCC`及`AIX`的`cc/xlC`是无法编译。我会在后期进行优化，同时，让整个编译过程自动化（不用指定平台参数等）。

## 支持非so类型的c扩展库

类`unix`平台动态库的实现方式有两种（仅我所了解的）：

一种是，以`“共享对象”`的方式管理，文件以`so`为后缀。典型的平台有Linux、FreeBSD等。
另一种是，以HP-UX为主，以`“共享库”`的方式管理，文件以`sl`为后缀。两种文件的实际编译方式也不相同（后续会专门补充相关内容）。
目前修改之后的Lua仅支持so文件。如果需要在HP-UX平台运行应用，有以下三种可选处理方案：

（1）.编译c扩展模块时，统一生成`so`为后缀的文件：优点，不用修改lua代码。缺点，管理模式有别于主机平台（容易产生幻觉）。

（2）.修改lua代码，支持`sl`为后缀的共享库。同时，在编译lua时，指定编译开关。具体修改，如下：

可以针对HP-UX平台进行特殊处理，编译时增加相应的宏即可（示例为`_HP-UX`）。
编辑`~/lua-5.2.1/src/luaconf.h`文件，进行以下的修改：

<pre class="prettyprint linenums">
#if defined(_HP-UX) 
#define SHARED_LIB_EXT "sl" 
#else
#define SHARED_LIB_EXT "so" 
#endif
#define LUA_VDIR	LUA_VERSION_MAJOR "." LUA_VERSION_MINOR "/"
#define LUA_ROOT	"/usr/local/"
#define LUA_LDIR	LUA_ROOT "share/lua/" LUA_VDIR
#define LUA_CDIR	LUA_ROOT "lib/lua/" LUA_VDIR
#define LUA_PATH_DEFAULT  \
		LUA_LDIR"?.lua;"  LUA_LDIR"?/init.lua;" \
		LUA_CDIR"?.lua;"  LUA_CDIR"?/init.lua;" "./?.lua"
#define LUA_CPATH_DEFAULT \
		LUA_CDIR"?."SHARED_LIB_EXT";" LUA_CDIR"loadall."SHARED_LIB_EXT";" "./?."SHARED_LIB_EXT
</pre>

（3）.当然，还可以通过修改lua运行环境中c动态库的搜索路径_G.package.cpath的值来实现。

但鉴于这样修改不利于代码及产品的管理及维护（这样带来很多不确定性。每个执行环境都需要修改，如果改错了呢？），不建议此方案。
例如，我的主机上c扩展库的寻找路径，如下：

<pre class="prettyprint linenums">
> print(_G.package.cpath)
/usr/local/lib/lua/5.2/?.so;/usr/local/lib/lua/5.2/loadall.so;./?.so
> 
</pre>

备注：`本文仅仅是一个备案，后续会进行整体的完善`。

