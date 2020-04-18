---
layout: post
title: Windows 64位下Theano新Gpu Backend的安装方法
tags: [Theano, Windows]
---
说起来，theano也用了很长时间了，对安装这一套理应感觉非常熟练，不过好像这个新的GpuBackend的安装花费了我大量的时间，因此特意写下这篇教程，为各位铺路。
让我们回到3月初的某一天，升级了theano的0.9某一版本。起初感觉并没什么特别的，不过导入时的一个警告引起了我的注意。

> The old CUDA backend will be deprecated soon, in favor of the new libgpuarray backend.

大概意思是：老的Gpu后端要过期啦，请尽快升级！
这用的好好的，干嘛升级呢，不管他。不过，过了两天，我使用的翻译工具包nematus的官网上出现了下面一段对比：

> GPU, CuDNN 5.1, theano 0.9.0dev5.dev-d5520e:
>
> THEANO_FLAGS=mode=FAST_RUN,floatX=float32,device=gpu ./test_train.sh
>
> 173.15 sentences/s
>
> GPU, CuDNN 5.1, theano 0.9.0dev5.dev-d5520e, new GPU backend:
>
> THEANO_FLAGS=mode=FAST_RUN,floatX=float32,device=cuda ./test_train.sh
>
> 209.21 sentences/s

这个速度提升还不小，那赶紧行动吧，点击warning里的[网址](http://deeplearning.net/software/theano/tutorial/using_gpu.html)，你会看到这样一段介绍：

> Using the GPU in Theano is as simple as setting the device configuration flag to device=cuda (or device=gpu for the old backend). You can optionally target a specific gpu by specifying the number of the gpu as in e.g. device=cuda2. You also need to set the default floating point precision. For example: THEANO_FLAGS='cuda.root=/path/to/cuda/root,device=cuda,floatX=float32'. You can also set these options in the .theanorc file’s [global] section:
> ```conf
> [global]
> device = cuda
> floatX = float32
> ```

哇，看起来好简单，只要把device从gpu改成cuda就行啦！看到这，是不是有人觉得就把gpu改成cuda，这篇教程也太简单了吧！别急，这只是噩梦的开始。
导入后会发现提示pygpu没有安装，继续回到上一次的那个页面，翻到Installation章节，他会告诉你如果是使用conda安装的，那么就执行

```powershell
conda install pygpu theano
```

否则呢，就先
```powershell
pip install theano
```

再去按照另一个[官方教程](http://deeplearning.net/software/libgpuarray/installation.html#windows-specific-instructions)安装libgpuarray

如果你走第一条路，你会发现安装过程一路顺风，但是导入过程中会发生找不到cudnn.lib的问题。那遵照官方repo的issue里的[解决方案](https://github.com/Theano/libgpuarray/issues/264)，会让你在.theanorc文件中添加如下一段代码：
> ```conf
> [dnn]
> enabled = True
> include_path=C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v8.0/include
> library_path=C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v8.0/lib/x64
> ```

改完你会发现，哇！竟然闪退了！第一条路，Die！

那第二条路呢，感觉更加惨，官方的教程会让我们用cmake编译一个库，有很大的可能性你无法完成编译，就算你完成了编译，又会回到第一条路上，还是Die!

那怎么办，就这么结束了吗？上面说的的issue更新了一段，总结了新的安装过程：

1. 卸载并重新安装Anaconda或者Miniconda

2. 配置清华镜像，当然大多数教程包括官网的教程都会是下面两行代码：
    
    ```powershell
    conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
    conda config --set show_channel_urls yes
    ```

    我建议大家可以加上他们新[推出](https://github.com/tuna/issues/issues/112)的msys2第三方通道，代码如下：

    ```powershell
    conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
    ```

    这样后面的某些库就不需要跑到官方源去下载了。

3. 安装一些theano的依赖库，参考官方的[安装教程](http://deeplearning.net/software/theano/install_windows.html#gpu-windows)

    ```powershell
    conda install numpy scipy mkl-service libpython m2w64-toolchain nose
    ```

4. 安装theano和pygpu

    ```powershell
    conda install pygpu theano
    ```

5. 清理theano的缓存，如果出错，请保证theano没有在运行，实在不行就用管理员权限。如果出现问题，可以每次在编译失败后都使用这条指令。

    ```powershell
    theano-cache purge
    ```

大多数的用户到这里应该都是没问题了，包括我自己。不过呢，最近帮几个同学安装的时候，发现了几个新的问题，大家可以参考一下。
首先是这样一个[问题](https://github.com/Theano/libgpuarray/issues/407)。

```py
>>> import theano
ERROR (theano.gpuarray): Could not initialize pygpu, support disabled
Traceback (most recent call last):
  File "/usr/local/lib/python3.5/dist-packages/theano/gpuarray/__init__.py", line 175, in <module>
    use(config.device)
  File "/usr/local/lib/python3.5/dist-packages/theano/gpuarray/__init__.py", line 162, in use
    init_dev(device, preallocate=preallocate)
  File "/usr/local/lib/python3.5/dist-packages/theano/gpuarray/__init__.py", line 65, in init_dev
    sched=config.gpuarray.sched)
  File "pygpu/gpuarray.pyx", line 614, in pygpu.gpuarray.init (pygpu/gpuarray.c:9419)
  File "pygpu/gpuarray.pyx", line 566, in pygpu.gpuarray.pygpu_init (pygpu/gpuarray.c:9110)
  File "pygpu/gpuarray.pyx", line 1021, in pygpu.gpuarray.GpuContext.__cinit__ (pygpu/gpuarray.c:13472)
pygpu.gpuarray.GpuArrayException: Error loading library: -1
```
这是我第一次在别人机器上安装时遇到的，我的解决方案在issue也写了，就是升级CUDA版本到8.0.61及以上，cudnn升级到5.1版本，当然尽量把显卡驱动的版本也进行升级。

当然我遇到最棘手的是这样一个[问题](https://github.com/Theano/libgpuarray/issues/5831)。

```py
>>> import theano 
ERROR (theano.gpuarray): Could not initialize pygpu, support disabled
Traceback (most recent call last):
 File "C:\Users\I\Anaconda2\lib\site-packages\theano\gpuarray\__init__.py", line 164, in <module>
    use(config.device)
 File "C:\Users\I\Anaconda2\lib\site-packages\theano\gpuarray\__init__.py", line 151, in use
    init_dev(device)
 File "C:\Users\I\Anaconda2\lib\site-packages\theano\gpuarray\__init__.py", line 66, in init_dev
    avail = dnn.dnn_available(name)
 File "C:\Users\I\Anaconda2\lib\site-packages\theano\gpuarray\dnn.py", line 174, in dnn_available 
    if not dnn_present():
 File "C:\Users\I\Anaconda2\lib\site-packages\theano\gpuarray\dnn.py", line 165, in dnn_present dnn_present.msg) 
RuntimeError: You enabled cuDNN, but we aren't able to use it: cannot compile with cuDNN. 
We got this error: 
In file included from C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v8.0\include/driver_types.h:53:0, 
                 from C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v8.0\include/cudnn.h:63, 
                 from c:\users\iotlab\appdata\local\temp\try_flags_5jgpcp.c:4: 
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v8.0\include/host_defines.h:84:0: warning: "__cdecl" redefined 
#define __cdecl 
^ 
<built-in>: note: this is the location of the previous definition 
C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v8.0\lib\x64/cudnn.lib: error adding symbols: File in wrong format 
collect2.exe: error: ld returned 1 exit status
```

这个问题重装了VS，Anaconda，驱动都无济于事，正当我准备放弃时，发现了repo的这样一个[issue](https://github.com/Theano/libgpuarray/issues/5838)，说是在cmd里能正常导入theano，然而在jupyter notebook中却会报这样的错误。我一抖机灵，赶紧使用jupyter console导入了theano，竟然一切正常。

<p align="center">
  <img src="/img/theano_gpu_backend_issue.jpg">
</p>

不敢相信这竟然是同一台计算机上发生的。不过呢，这个现象的产生我大概搞明白了问题的产生原因，theano的c++编译器不固定导致的。于是呢，去搜索theano.config的文档，问题就迎刃而解了。解决方案可以看issue描述，我这里也介绍一遍，
首先在.theanorc文件的[global]部分，添加一个参数cxx，如下：

```conf
cxx=C:\Anaconda2\Library\mingw-w64\bin\g++.exe
```

接着将目录添加进环境变量PATH中，以上面的路径为例，即为
```
C:\Anaconda2\Library\mingw-w64\bin
```

重新打开cmd,让环境变量生效，接着import theano，你会发现奇迹出现了：

<p align="center">
  <img src="/img/theano_gpu_backend_result.png">
</p>
