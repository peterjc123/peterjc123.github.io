---
layout: post
title: PyTorch 0.4.0 正式版 - Windows 说明
tags: [PyTorch, Windows]
---
让大家久等了，这次 Windows 的 PyTorch 终于正式发布了。欢迎大家去官网下载安装并体验使用，在此我补充一些官方没有的关于 Windows 的发布说明和常见问题。

# 0.4.0 发布说明
## 错误修复：

1. 修复多进程下的内存泄漏问题 [PR #5585](https://github.com/pytorch/pytorch/pull/5585)

2. 使用多线程版本 MKL 替代顺序版 MKL ，在 CPU 上带来10%的速度提升 [PR #6416](https://github.com/pytorch/pytorch/pull/6416)

3. 重新添加 Compute Capability 5.0 显卡的支持

## 新功能：

1. 在编译中加入 MAGMA

2. 添加 CUDA 9.1 build

3. 提供 Wheels 包

4. 支持新的cpp拓展 [PR #5548](https://github.com/pytorch/pytorch/pull/5548)

## 已知问题：

1. 使用CPU来跑模型比Linux/Mac要慢很多，具体视模型复杂度和CPU核心数而定。原因是MSVC对新版本OpenMP的支持很差，因此我们会在下一个版本对一些组件进行替换，从而解决这个问题。使用GPU速度不受影响，与 Linux/Mac 没有区别。

2. 包括分布式, NCCL等在内的一些组件在Windows下不受支持，详细可见：[Issue #4092](https://github.com/pytorch/pytorch/issues/4092)

## 常见问题
### 安装
1. Conda 依赖项冲突

    ```
    Solving environment: failed 
    
    UnsatisfiableError: The following specifications were found to be in conflict: 
    - libxml2 -> libiconv -> *[track_features=vc14] 
    - libxml2 -> libiconv -> vc==14 
    - pytorch 
    Use "conda info <package>" to see the dependencies for each package.
    ```

    对于0.3.0及以下，请强制升级相关的包。

    ```powershell
    conda update libiconv 
    # 或者强制升级所有包 
    conda update --all
    ```

    对于0.4.0及以上，请将 vc 恢复至默认状态。

    ```powershell
    conda install -c defaults vc=14
    ```

2. 在通道内找不到相应的包

    ```
    Solving environment: failed

    PackagesNotFoundError: The following packages are not available from current channels:

    - pytorch

    Current channels:
    - https://conda.anaconda.org/pytorch/win-32
    - https://conda.anaconda.org/pytorch/noarch
    - https://repo.continuum.io/pkgs/main/win-32
    - https://repo.continuum.io/pkgs/main/noarch
    - https://repo.continuum.io/pkgs/free/win-32
    - https://repo.continuum.io/pkgs/free/noarch
    - https://repo.continuum.io/pkgs/r/win-32
    - https://repo.continuum.io/pkgs/r/noarch
    - https://repo.continuum.io/pkgs/pro/win-32
    - https://repo.continuum.io/pkgs/pro/noarch
    - https://repo.continuum.io/pkgs/msys2/win-32
    - https://repo.continuum.io/pkgs/msys2/noarch
    ```

    PyTorch 在32位系统上不受支持。请使用64位的 Windows 和 Python。

3. 从我的通道迁移至官方通道

    如果你使用 peterjc123 通道的 0.3.1 版本 , 那么你可以直接进行升级。

    ```powershell
    conda install -c pytorch pytorch
    ```

    否则请先卸载所有与 PyTorch 相关的包。

    ```powershell
    conda uninstall pytorch 
    # 如果是CUDA版本的话 
    conda uninstall cuda80 cuda90 
    # 如果 已安装的 PyTorch 版本 >= 0.3 
    conda install -c defaults vc=14  
    # 安装官方的包 
    conda install -c pytorch pytorch
    ```

4. 为什么没有 Python 2 的包?

    因为它还不够稳定，在正式发布之前，我们需要解决一系列的问题。实在需要的话，你可以自己进行编译。

5. 怎么安装torchvision?

    你可以通过 PyPI 进行安装。

    ```powershell
    pip install torchvision
    ```

6. 导入错误 1

    ```py
    from torch._C import * 
    
    ImportError: DLL load failed: The specified module could not be found.
    ```

    这个问题说明缺少了必要的一些动态链接库。其实我们在发布时已经将几乎所有PyTorch需要的文件全部打包了，但是没有包含VC 2017 运行时。你可以尝试通过下面的命令进行安装：

    ```powershell
    conda install -c peterjc123 vc vs2017_runtime
    ```

    另外一个原因是你安装了GPU版本的PyTorch，但是却没有Nvidia显卡。这种情况下，请将其替换为CPU版本。

7. 导入错误 2

    ```py
    from torch._C import * 

    ModuleNotFoundError: No module named 'torch._C'
    ```

    这个错误与上一个不同。这通常会在以下情况发生：你的代码目录中有一个名为torch.py的文件。对文件做一下重命名就能解决。

8. 导入错误 3

    ```py
    ModuleNotFoundError: No module named 'torch'
    ```

    这通常是你安装PyTorch的Python环境(pip/conda)和使用的Python环境不同造成的。你可以通过键入以下的指令进行检查。

    ```powershell
    where python.exe 
    # 如果使用 conda 进行安装 
    where conda.exe 
    # 如果使用 pip 进行安装 
    where pip.exe
    ```

    注意这些命令输出的第一行，他们应该指向同一个Python环境。

    ```
    C:\Anaconda2\Scripts\conda.exe 
    C:\Anaconda2\python.exe 
    C:\Anaconda2\Scripts\pip.exe
    ```

9. 网络错误

    ```py
    CondaError: EOFError('Compressed file ended before the end-of-stream marker was reached',)
    ```

    从 `[Anaconda dir]\pkgs` 删除损坏的PyTorch包并重试。

### 使用（多进程）

1. 多进程错误 1

    ```py
    RuntimeError:
        An attempt has been made to start a new process before the
        current process has finished its bootstrapping phase.

        This probably means that you are not using fork to start your
        child processes and you have forgotten to use the proper idiom
        in the main module:

            if __name__ == '__main__':
                freeze_support()
                ...

        The "freeze_support()" line can be omitted if the program
        is not going to be frozen to produce an executable.
    ```

    Python的 multiprocessing 库在 Windows 下的实现与 Linux 不同, 它使用的是 spawn ，而不是 fork 。所以我们需要在代码入口加上一个判断来防止程序多次执行。可以将代码重构为如下结构：

    ```py
    import torch

    def main()
        for i, data in enumerate(dataloader):
            # do something here

    if __name__ == '__main__':
        main()
    ```

2. 多进程错误 2

    ```py
    ForkingPickler(file, protocol).dump(obj) 
    
    BrokenPipeError: [Errno 32] Broken pipe
    ```

    这个错误当子进程过早结束时会发生。你的代码可能存在一些问题，可以将 DataLoader 的 num_worker 设置为0来进行调试。

3. 多进程错误 3

    ```powershell
    Couldn’t open shared file mapping: <torch_14808_1591070686>, error code: <1455> at D:\Downloads\pytorch-master-1\torch\lib\TH\THAllocator.c:154

    [windows] driver shut down
    ```

    请升级你的显卡驱动。如果升级后问题仍然存在，要么是你的显卡太老了，或者是计算太过复杂。可以尝试根据这个帖子修改TDR设置。

4. CUDA IPC 操作

    ```powershell
    THCudaCheck FAIL file=torch\csrc\generic\StorageSharing.cpp line=252 error=63 : OS call failed or operation not supported on this OS
    ```

    这些在Windows上是不受支持的。有两个替代方案：

    a) 不要使用 `multiprocessing`。

    b) 共享 CPU tensors。

### CFFI拓展

1. 这种类型的拓展在 Windows 下支持比较差，而且这些拓展太多了，只靠几个人是没法改完的。如果硬要使用的话，可以看下面的说明。

2. 在Windows下编译，你需要在Extension对象中指明 `libraries`。

    ```py
    ffi = create_extension(
        '_ext.my_lib',
        headers=headers,
        sources=sources,
        define_macros=defines,
        relative_to=__file__,
        with_cuda=with_cuda,
        extra_compile_args=["-std=c99"],
        libraries=['ATen'] # 在必要时添加CUDA相关的库，如cudart
    )
    ```

3. 无法解析的外部链接 `extern THCState *state;`

    ```cpp
    error LNK2001: unresolved external symbol state caused by extern THCState *state;
    ```

    将代码后缀名从.c改为.cpp，并对代码做如下改动。

    ```cpp
    #include <THC/THC.h>
    #include <ATen/ATen.h>

    THCState *state = at::globalContext().thc_state;

    extern "C" int my_lib_add_forward_cuda(THCudaTensor *input1, THCudaTensor *input2,
                                        THCudaTensor *output)
    {
    if (!THCudaTensor_isSameSizeAs(state, input1, input2))
        return 0;
    THCudaTensor_resizeAs(state, output, input1);
    THCudaTensor_cadd(state, output, input1, 1.0, input2);
    return 1;
    }

    extern "C" int my_lib_add_backward_cuda(THCudaTensor *grad_output, THCudaTensor *grad_input)
    {
    THCudaTensor_resizeAs(state, grad_input, grad_output);
    THCudaTensor_fill(state, grad_input, 1);
    return 1;
    }
    ```

4. C++拓展

    这个类型的拓展在Windows下的支持比上面要好的多。但是他还是需要一些手工配置。首先，你需要打开x86_x64 Cross Tools Command Prompt for VS 2017，然后在其中打开git-bash，他一般的位置在`C:\Program Files\Git\git-bash.exe`，最后你可以开始进行编译。

### 其他

1. MAGMA 和 NCCL 支持

    我们编译的包中包含了 MAGMA，但是当你的矩阵很大的时候 ，程序很可能不能正常工作。原因是在代码中它使用了 `long` 而不是 `int64_t` ，这就可能会在Windows上造成溢出问题。至于 NCCL，Nvidia只提供了Linux系统的二进制文件，也没有开源相应的代码，因此在NCCL for Windows发布之前，我们没有办法对其进行编译。

2. 分布式支持

    我们会在未来对其进行研究和实现。

3. 老显卡架构支持

    ```powershell
    THCudaCheck FAIL file=torch\lib\thcunn\generic/Threshold.cu  line=34 error=48 : no kernel image is available for execution on the device
    ```

    为了和其他系统保持同步，我们无法对老显卡提供支持。不过你可以借助我的[repo](https://github.com/peterjc123/pytorch-scripts)从源码进行编译，也可以安装 [Google Drive](https://drive.google.com/drive/folders/0B-X0-FlSGfCYdTNldW02UGl4MXM?usp=sharing) 中的旧版本的包，亦或者安装我生成的 CI [包](https://ci.appveyor.com/project/peterjc123/pytorch-elheu/branch/v0.4.0) 。
