---
layout: post
title: 关于Windows PRs并入PyTorch的master分支
tags: [PyTorch, Windows]
---
经过几个月的努力，随着11月8号[PR 2941](https://github.com/pytorch/pytorch/pull/2941)并入PyTorch之后，我们终于将关于Windows支持的相关PR全部并入了PyTorch的master分支，现在你可以直接对master分支进行编译了。编译需要的组件有：

CPU版本：

1. Visual Studio 2017 C++ Build Tools

2. CMake 3.0 及以上

3. 64位Windows系统

4. 64位Anaconda/Miniconda 或者 Python 3.5及以上

GPU版本：

1. CUDA 8.0 及以上

2. NVTX (在 CUDA 中为VS的插件，若安装失败，可以解压CUDA安装包，在CUDAVisualStudioIntegration中找到）

3. 对于CUDA 8 的编译还需要Visual Studio 2015 with Update 2 及以上

可选项：

1. cuDNN 6.0 及以上

2. BLAS 运算库 （主要是OpenBLAS和MKL）

**更新：已添加一个[repo](https://github.com/peterjc123/pytorch-scripts)用于一键进行编译安装，欢迎体验使用。**

编译步骤如下：

1. clone 官方 repo，并执行一些预备处理

    ```batch
    git clone --recursive https://github.com/pytorch/pytorch
    cd pytorch
    xcopy /Y aten\src\ATen\common_with_cwrap.py tools\shared\cwrap_common.py
    ```

2. 在开始菜单找到x86_x64 Cross Tools Command Prompt for VS 2017，打开并切换目录至pytorch的目录下。如果找不到，他的位置一般在`C:\Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\VC\Auxiliary\Build\vcvarsx86_amd64.bat`。

3. 在x86_x64 Cross Tools Command Prompt for VS 2017下执行如下一些预配置（在set命令后请务必不要多打空格或Tab）

    ```batch
    # 如果不需要 CUDA 支持
    set NO_CUDA=1

    # 如果安装有多个 CUDA 版本，默认会编译最后安装的版本，若要覆盖
    set CUDA_PATH=%CUDA_PATH_V8_0%
    # 或者
    set CUDA_PATH=%CUDA_PATH_V9_0%

    # 对于 CUDA 8 的编译
    set CMAKE_GENERATOR=Visual Studio 14 2015 Win64

    # 对于 CUDA 9 / CPU 的编译
    set CMAKE_GENERATOR=Visual Studio 15 2017 Win64

    # 你也可以使用 Ninja 来加速 CUDA 的编译
    pip install ninja
    set CMAKE_GENERATOR=Ninja
    # 如果使用 Ninja 为 CUDA 8 进行编译
    set PREBUILD_COMMAND=%VS140COMNTOOLS%\..\..\VC\vcvarsall.bat
    set PREBUILD_COMMAND_ARGS=x86_amd64

    # 如果需要多次编译，可以使用 clcache 来加快下次编译的速度
    pip install git+https://github.com/frerich/clcache.git
    set USE_CLCACHE=1
    set CC=clcache
    set CXX=clcache

    # 如果需要添加 BLAS 支持(OpenBLAS, MKL)
    set LIB=[PATH_TO_BLAS_LIBS];%LIB%

    # （仅Conda）如果你的Python版本低于3.5
    set PYTHON_VERSION=3.5 # 3.6 or up is also fine 
    conda create -q -n test python=PYTHON_VERSION numpy mkl cffi pyyaml
    activate test

    # （仅Python）请安装第三方的numpy和mkl包和官方的pyyaml
    pip install numpy.whl
    pip install mkl.whl
    pip install pyyaml

    # 如果你同时安装了 VS 2015 和 2017
    set DISTUTILS_USE_SDK=1
    ```

4. 开始编译安装

    ```powershell
    python setup.py install
    ```

目前针对Windows的已修复项：

1. 在backward过程中抛出异常会导致死锁 [PR 2941](https://github.com/pytorch/pytorch/pull/2941)

2. 在Dataloader开多线程时，会存在内存泄漏 [PR 2897](https://github.com/pytorch/pytorch/pull/2897)

3. torch.cuda下的一个缩进bug [PR 2941](https://github.com/pytorch/pytorch/pull/2941)

4. 增加对新 CUDA 和 cuDNN 版本的支持 [PR 2941](https://github.com/pytorch/pytorch/pull/2941)

目前Windows的已知问题：

1. 部分测试会遇到权限不足问题 [PR 3447](https://github.com/pytorch/pytorch/pull/3447)

2. 分布式 torch.distributed 和 多显卡 nccl 不支持

3. python 3.5 以下的版本不支持

4. 多线程的使用方式与 Unix 不同，对于DataLoader的迭代过程一定要使用如下代码做保护。如遇到多线程下的问题，请先将num_worker设置为0试试是否正常。

    ```py
    if __name__ == '__main__':
    ```

另外，大家一定很关心什么时候能出正式Windows正式版，日前，Soumith大神给出了他的[回复](https://github.com/pytorch/pytorch/issues/3649#issuecomment-344280941)：

<p align="center">
  <img src="/img/pytorch_windows_release_plan.jpg">
</p>

所以这次应该还是见不到正式的Windows版本，但是各位可以期待到时候我的Conda包。
