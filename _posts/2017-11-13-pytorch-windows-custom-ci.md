---
layout: post
title: 关于PyTorch在Windows下的CI和whl包
tags: [PyTorch, Windows, 'CI']
---
前两天给大家分享了PyTorch master分支在Windows下的编译过程，当然这个编译过程比较繁琐，如果能有直接能安装使用的包就能方便不少。现在比较流行的方式是使用CI（循环构建）来完成这个任务。

可能大家对CI不太了解，我稍微介绍一下。在Github浏览项目时，大家很可能会碰到一些徽章，比如像下面这张。它说明上一次在主分支下的构建过程成功了。这类徽章清楚的显示了CI在后台执行的结果。其实对于那些需要快速迭代的大型项目，用CI可以在对每次代码的改动之后检验代码的有效性，大大改善项目开发的速度。CI为我们进行编译、测试、打包或者分发都提供了相当的便利。提供Linux环境的CI中最著名的就是Travis CI了，而在Windows下的CI我觉得首推AppVeyor。AppVeyor的功能基本是比较全的，你可以自由的对环境进行一些配置，当然他也内置了不少工具，可用于使用各种编程语言，软件框架的编译。此外，AppVeyor的机器还包括Nvidia的显卡（貌似独此一家），也就是说可以用于PyTorch CUDA版本的编译，甚至也可以跑一些小模型。（算力并不强）当然，免费的午餐也是有条件的，每次build不能超过1个小时，CPU核心只有2个，不能并行跑多个项目等等。所以有钱的朋友们可以自己买VPS来跑CI。

<p align="center">
    <img src="/img/ci_badge_example.jpg">
    <br>
    <font size="-1">CI 徽章</font>
</p>

用CI来完成编译、测试、打包和发布这个过程是非常容易的，为什么这么说呢？可以看到下面那张AppVeyor的设置图，直接就对应了每个过程有相应的配置，再每个配置项中你可以自行设定相应的脚本。什么意思呢？就是当你提交代码之后，他会遵循编译、测试、打包和发布这个流程，轮流的执行你设定的代码。有个很酷的功能是它可以打印build时的该主机的地址和端口，然后你可以用远程桌面直接去访问该主机。我没有尝试过，各位有兴趣的话可以自己试试看。

<p align="center">
    <img src="/img/ci_settings_example.jpg">
    <br>
    <font size="-1">CI 的设置项</font>
</p>

关于具体的搭建过程就不再赘述了，下面说正题，首先是使用CPU和GPU版本的运行要求。

CPU版本：

1. Windows 64位系统

2. 64位 Python 3.5 或者 3.6

3. MKL、Numpy、PyYAML

4. VC++ 2017 Redist

GPU版本还需要：

1. CUDA 8

2. cuDNN 6

3. NVTX（在 CUDA 中为VS的插件，若安装失败，可以解压CUDA安装包，在CUDAVisualStudioIntegration中找到，可以通过检验环境变量NVTOOLSEXT_PATH是否存在来判断其是否已经安装）

如果同时安装了CUDA 8和9的可以通过在使用前临时修改环境变量的方式使用。

```cmd
set PATH=%CUDA_PATH_V8_0%\bin;%PATH%
```

如果是包含多个版本cuDNN的，就需要把cuDNN 6的动态链接库拷贝至torch包的lib目录下。

当然你也可以把CUDA 8和cuDNN 6的动态链接库全部拷进torch包的lib目录下使用，主要依赖项包括CUDA8.0\bin目录下的各种dll和nvToolsExt64_1.dll。

如果你觉得这些方法都不能接受，那还请遵循上次的[教程](/2017-11-12-pytorch-windows-into-master/)自己进行编译。

关于搭建好的PyTorch的Windows CI。首先是CPU版本的，主页面在此。在主页面会有分别对应Python 3.5和3.6的build，选定相应的版本，然后会跳转至相应build的主页面，那怎么下载build好的whl包呢？在页面中部，会有一个菜单，如下图所示：

<p align="center">
    <img src="/img/ci_output_menu.jpg">
</p>

点击Artifacts，就可以发现下面出现了生成好的whl包，点击即可下载。

而GPU版本的就复杂的多，因为AppVeyor的1小时build时间限制，我们不得不将使用不同架构的显卡的build拆开来，所以大家先去了解下自己的显卡架构。我们主要对三种架构的显卡做了编译，Pascal、Kepler和Maxwell。GPU版本的CI主页面在[这里](https://ci.appveyor.com/project/peterjc123/pytorch-elheu)。进去可以发现，总共包含了12种build。主要是以下三个环境变量的区别：`APPVEYOR_BUILD_WORKER_IMAGE`、`PYTHON_VERSION`和`TORCH_CUDA_ARCH_LIST`。

- `APPVEYOR_BUILD_WORKER_IMAGE` 主要分为 VS 2017 和 2015。这个并不是使用的编译器，而是主要用于系统的区分。也就是说Windows 10和Windows Server 2016使用VS 2017的包，Windows 7 及其他用 VS 2015的包。

- `PYTHON_VERSION` 这个就是Python版本的区别，不多说了。

- `TORCH_CUDA_ARCH_LIST` 这主要是上面说的显卡架构区别，大家自己关注自己的显卡架构。

当这三个变量都确定之后，你就可以下载相应的包了，过程与上面相似。

当然这个还是我自己搭的第三方CI，但是既然Windows PRs都已经并入主分支了，那官方的Windows CI和包还会远吗？让我们拭目以待吧。

**更新：官方Windows CI已经列入计划**

<p align="center">
    <img src="/img/windows_ci_into_plan.jpg">
</p>
