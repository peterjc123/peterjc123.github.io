---
layout: post
title: Windows下Python中常见的几种DLL load failed问题的原因以及解决方案
tags: [Windows, Python, C/C++]
---
## 问题背景与相关知识
在深度学习流行的当下，深度学习的框架大多是基于Python的实现，抑或是提供了Python的接口。而为了保证性能，底层计算通常是使用C/C++实现的。对于C/C++项目，其链接方式主要可以分为以下两种：
- 静态链接方式 （Static linking）

    静态链接即在链接时即确定程序会包含哪些模块。由于在链接时已经确定了所有包含的模块，那么会直接将这些模块打包，生成一个新的可执行文件或者静态链接库文件。

    因而这种方法具有如下特点：

    - 分发简单。因为其不存在运行时才能解决的依赖。

    - 由于无法事先知道用户会使用其中的哪些模块以及哪些函数，因而产生的静态链接库会比较大。

- 动态链接方式 （Dynamic linking）

    动态链接方式即在链接时仅确定包含模块的名称、以及其中所用到的函数，生成一个可执行程序或是动态链接库（DLL）。而在运行时，根据这些信息来寻找相应的模块进行加载。

    这种方法的特点有：

    - 由于在运行时才加载对应的模块，因而可以将模块进行替换。

    - 由于所依赖的模块没有被包含进来，因此生成的动态链接库不会太大。

    - 由于动态的加载，因而会受用户环境的影响，因而在分发时有可能产生问题。

在目前 Windows 的大多数深度框架下，大多采用动态链接方式。其原因在于
1. 深度学习框架中需要包含许多 OP 的实现，这些实现需要覆盖多种指令集、多个 CUDA 架构，因而生成的目标文件会比较大。而64位 Windows 下对于单个模块要求大小不能超过4GB。

2. 某些模块不提供静态链接库，比如 cuDNN 等。

在动态链接方式的第三个特点中，我们说到了分发时可能会产生问题。这类问题通常被称为"DLL Hell"。可以考虑下面一个场景：

程序 P<sub>1</sub> 依赖于 库 L 的一个版本 L<sub>1</sub>。用户为了能使用 P<sub>1</sub>，将 L<sub>1</sub> 放在系统目录中。而某一天，用户又需要使用程序 P<sub>2</sub>，而 P<sub>2</sub> 则依赖于 库 L 的另一个版本 L<sub>2</sub>。假设 L<sub>1</sub> 和 L<sub>2</sub> 是无法互相兼容的，那么用户就无法同时使用 P<sub>1</sub> 和 P<sub>2</sub>。

为了解决这样一种问题，有如下的一些解决方法
1. 每个程序自带一份动态链接库的局部拷贝。即 P<sub>1</sub> 分发时带上 L<sub>1</sub>，而 P<sub>2</sub> 分发时带上 L<sub>2</sub>，这样两者就不会互相干扰。但是很明显，这种方式会导致一个动态链接库的多次拷贝，造成空间的浪费。

2. 很明显 L<sub>1</sub> 和 L<sub>2</sub> 会互相冲突，是因为两者的名称一样，那么对于无法互相兼容的包，直接取不同的文件名就完了。这正是 Unix 下的解决方法，库文件的取名为 `libxxx.so.主要版本.修订版本号`。如果一个库无法与之前的版本互相兼容，那么需要增加主要版本号，否则可以仅增加修订版本号。此外，为了使用方便，一般还会生成一个不带修订版本号的软连接 `libxxx.so.主要版本` 指向当前主要版本的最新库文件。

虽然方法2看上去是一个不错的方法，但是它依赖于一个命名规则。这一方面不是强制要求，另一方面若仅是开发者用于调试，则非常的不方便。而对于方法1，如果某个程序还是将库安装到了系统目录，那还是会造成问题。如果我们可以额外设定一些信息来辅助动态链接库的查找，那么问题就迎刃而解了。这个信息在 Unix 中被称为 `RPATH` ，在编译链接时指定，而对于 .NET 平台，则可以新建一个 `.config` 文件来指定动态链接库的目录。

很可惜，由于Windows平台上所背负的沉重包袱，没有类似引入这样的解决方法。此外，由于没有统一的包管理器，因此方法2也是无法使用的。然而，即使我们采用了方法1，也无法保证程序可以正确的载入所有依赖库。在一般情况下，在DLL加载失败时，Windows会跳出一个消息框来提醒用户哪个DLL加载失败了。但是，Python 为了保证代码不会被UI阻塞，因此将这个消息框弹出的消息屏蔽了，这使得调试这个问题变得更为复杂。

下面我们将具体的介绍这个问题的产生原因以及解决方法。在介绍Python下常见的几种DLL load failed错误之前，我们可以将其分为两种类型。
1. 静态 DLL 加载错误。即 DLL 库自身没有逻辑错误，但是由于缺少依赖项、加载了错误版本的 DLL 导致加载失败。这种错误使用静态调试工具即可解决。

2. 动态 DLL 加载错误。即 DLL 库的自身逻辑可能存在问题，在初始化时便遇到未处理的异常，导致 DLL 加载失败。这种错误仅能使用动态调试工具解决。

## 常见的Python下的几种DLL load failed错误
1. DLL 缺失

    常见的错误消息：

    ```py
    ImportError: DLL load failed: The specified procedure could not be found.
    ImportError: DLL load failed: The specified module could not be found.
    ImportError: DLL load failed: 找不到指定的程序
    ImportError: DLL load failed: 找不到指定的模块
    ```

    很明显，这种错误就是缺少所依赖的 DLL 导致的。由于 DLL 中包含了所依赖的 DLL 的相关信息，因此可以通过循环的枚举查找对应的 DLL 来模拟 DLL 的加载过程来调试这个问题。当然，我们也可以动态的追踪系统如何解析、查找并加载 DLL 来找出具体缺少的 DLL。

2. DLL 不兼容

    常见的错误消息：

    ```py
    ImportError: DLL load failed: The operating system cannot run %1.
    ImportError: DLL load failed: 操作系统无法运行%1
    ```

    从消息可以看出，在加载路径上存在一个无效的 DLL 导致加载失败。一般来说，是由于32位，64位 DLL 混用导致的。当然，也可能是 DLL 损坏导致的。

3. DLL 逻辑错误

    常见的错误消息：

    ```py
    ImportError: DLL load failed: A dynamic link library (DLL) initialization routine failed
    ImportError: DLL load failed: 动态链接库(DLL)初始化例程失败
    ```

    这种错误就比较有迷惑性了，看起来是一个用户错误，然而其实是一个程序逻辑错误，一般是由于初始化静态变量时时抛出了未处理的异常，导致 DLL 加载失败。作为用户来说，其实可以抛给开发者来解决。当然如果你也想过一把开发者的瘾，那么可以尝试自己调试后将堆栈调用报给开发者。

## 常见的 DLL 加载调试工具
下面将介绍一些常见的 DLL 加载调试工具，我们将这些工具按大小以及复杂性升序排序，并依次进行介绍。
1. Dependency Walker / Dependencies

    Dependency Walker 是一种经典的静态 DLL 加载调试工具，可以在 [https://www.dependencywalker.com/](https://www.dependencywalker.com/) 下载到。然而由于其好久没更新了，对于新系统中的一些 Windows 组件没有做适配，因此有人用 .NET 重写了一个版本， 称为 Dependencies。项目主页：[https://github.com/lucasg/Dependencies](https://github.com/lucasg/Dependencies)。

2. Process Monitor

    Process Monitor 是属于被微软收编的 Sysinternal 工具的一部分，是用于监控程序的 Windows 调用的一种调试工具。有人说写这个工具的大佬可能比现有的微软内部员工更懂 Windows 内部逻辑（笑）。他可以满足一部分用于 DLL 加载调试的需求。项目地址：[https://docs.microsoft.com/en-us/sysinternals/downloads/procmon](https://docs.microsoft.com/en-us/sysinternals/downloads/procmon)

3. Debugging Tools for Windows (WinDBG)

    做 Unix 开发的应该都知道 GDB 吧，那么 Windows 下的 GDB 是啥呢？其实就是 WinDBG，这个上能查蓝屏dump，下能查程序崩溃。虽然它可以单独使用，其实其包含在一套工具链 Debugging Tools for Windows 中，其中另外的工具有 CDB（CLI 版本）、GFlags（别误会，不是Google的那个命令行处理库，而是 Global Flags，可以开启一些开发者选项）、KD（内核调试）以及 NTSD（以前 XP 时代经常用来强杀系统进程，当然其实它也是一个调试工具）。下载链接：[https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools)

4. Visual Studio

    最后就是大名鼎鼎的开发工具 Visual Studio 了。对于程序异常，VS 来处理应该是最最简单与方便的了。当然对于用户错误来说，装一个几个 GB 的开发工具，未免有些小题大做了。主页：[https://visualstudio.microsoft.com/](https://visualstudio.microsoft.com/)

## 错误的调试方法
1. DLL 缺失

    a) 使用 Dependency Walker 或 Dependencies 打开待调试的 DLL 即可。

    <p align="center">
        <img src="/img/dll_missing_dependencies.png">
    </p>

    可以看到程序会列举出缺少的 DLL 文件（例如图中的`shm.dll`），我们只要将其补上就可以完成 DLL 的加载了。

    b) 使用 Process Monitor 来追踪加载过程。可以先使用 Ctrl + L 添加过滤条件，即 `Process Name is Python.exe` 以及 `Operation is CreateFile`. 然后，以PyTorch举例，在 CMD 中运行 `python -c "import torch"` 或是在IDE中运行包含`import torch`的脚本。然后 Process Monitor 中会生成一系列的日志。

    <p align="center">
        <img src="/img/dll_missing_procmon.png">
    </p>
    
    找到最后一个不成功的 DLL 查找请求，即可定位到缺少的 DLL 为 `shm.dll`。补上该 DLL 就能修复加载问题。

    c) 使用 Debugging Tools for Windows 进行定位。在 CMD / Powershell 窗口中执行如下命令。

    ```powershell
    # 打开 DLL 调试选项
    gflags /i python.exe +sls
    # 挂载调试器并执行命令
    cdb -o -c "~*g; q" python -c "import torch"
    # 关闭 DLL 调试选项
    gflags /i python.exe -sls
    ```

    <p align="center">
        <img src="/img/dll_missing_cdb.png">
    </p>

    可以看到，它直接找出了问题的原因，非常方便。


2. DLL 不兼容

    a) 仍然可以使用 Dependencies。在下方会列举 DLL 面向的平台。然而由于其为静态检查工具，因此这个并不能完全查出所有可能的问题。

    b) 使用 Process Monitor。参照 1b 进行问题的定位。但是需要将第二过滤条件需要改为 `Operation is ReadFile`，并查找最后一个成功的 DLL 调用。

    c) 使用 Debugging Tools for Windows。参考 1c 。

3. DLL 逻辑错误

    a) 使用 Debugging Tools for Windows。参考 1c 。不过因为这时候程序会 crash ，因此我们需要打印堆栈调用，而不能单纯的执行到第一个错误就退出。因此需要将1c中的第二条命令改为`cdb -o -c "~*kv; q" python -c "import torch"`。

    b) 由于是抓异常，因此 VS 可以闪亮登场了。先打开一个 Python 窗口，然后使用 VS 挂载该进程，接着在 Python 窗口中键入 `import torch` 回车。VS在抓到异常后会弹出提升，在Call Stack窗口中包含了当前的堆栈调用。当然作为代码调试来说，最好在有调试符号（即 PDB 文件）的情况下进行，这样可以看到崩溃的行号以及对于的源文件，甚至部分变量的值，方便开发者进行问题的定位。

    <p align="center">
        <img src="/img/dll_init_failure_vs.png">
    </p>

## 未来展望
其实，作为 Python 自身也在考虑如何解决这个问题。从 Python 3.8 开始，引入了 `os.add_dll_directory` 这个 API。也就是说对于环境变量 `PATH` 中的目录，不再能影响 DLL 的加载过程了。当然，并不是所有 Python 发行版都是这样做的，比如 Anaconda 还是保留了原有的加载逻辑。当然，还有一个非常 exciting 的改进，就是 Python 不再对 DLL 加载失败的弹窗进行屏蔽了。这样用户可以更容易的知道哪个 DLL 的加载出了问题，而不需要借助外部工具。当然，我也希望微软也能在编译工具进行改进，无论是引入`RPATH`也好，或是运行时读取配置文件也罢，只要能动态通过非环境变量的方式对DLL加载的路径进行设置，相信都能使 Python 中使用 C/C++ 拓展的库 在 Windows 上的体验更上一层楼。