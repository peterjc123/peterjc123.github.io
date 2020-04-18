---
layout: post
title: 关于 PyTorch 0.3.0 在Windows下的安装和使用
tags: [PyTorch, Windows, 'CI']
---
**更新：Conda的编译问题已经解决，请访问[此处](/2017-05-12-pytorch-package/)安装新版本。**

另外，再把相应的发布日志一同搬过来吧。

Windows版本的改动:
### 错误修复
1. backward中的错误会导致死锁

2. DataLoader多线程时的内存泄漏

3. torch.cuda中的缩进bug

### 新功能
1. 添加对 CUDA 和 cuDNN 新版本的支持

2. 添加对 Ninja 和 clcache 编译器的支持

### 已知问题
1. 有些测试不能通过

2. 不能支持 torch.distributed（分布式）、NCCL（多卡）和 Magma

3. 不支持 3.5 及以下的版本

4. num_worker设置为0以上的值会导致内存泄漏。另外代码入口得用以下的if语句包裹。这里稍微解释下为什么要这样，因为Windows不支持fork，多进程只能采用spawn，而如果你不用这个条件包裹，那么代码就会被再次执行一遍，这样如果不加限制，那么就会有无限多的进程了。

    ```py
    if __name__ == '__main__':
    ```

另外这两天还有个好消息是，Windows的CI已经正式在搭建中了，估计下周就能完工。以后大家可以更加方便的使用master分支进行编译了。我写的一些[脚本](https://github.com/peterjc123/pytorch-scripts)可以帮助你们方便的进行编译。
