---
layout: post
title: CircleCI下开启远程桌面的调试
tags: [Windows, CircleCI, RDP, 调试]
---
## TL;DR
1. 对于某个 job 使用"重启带SSH的任务"。

2. 提取 CircleCI 中给出的 SSH 命令

    ```powershell
    ssh -p 54782 35.237.241.189 -- cmd.exe
    ```

3. 确定本地以及远程用于端口转发的端口，并添加用于端口转发的参数（注意避免常用端口）

    ```powershell
    ssh -p 54782 35.237.241.189 -L [远程端口]:localhost:[本地端口] -- cmd.exe
    ```

4. 敲入上述命令，建立远程连接

5. 在上述窗口中，继续敲入如下命令

    ```powershell
    # 建立远程VM的本地端口转发
    netsh interface portproxy add v4tov4 listenport=[远程端口] connectaddress=127.0.0.1 connectport=3389
    # 设置新密码（注意复杂性要求）
    net user circleci [新密码]
    ```

6. 打开远程桌面或其他支持 RDP 连接的客户端，使用如下凭据登录：

    ```powershell
    用户名： circleci
    密码： [上一步设置的新密码]
    ```

-------

现在软件迭代的速度越来越快，为了保证质量，很多项目都采用了循环构建（Continuous Integration, CI）来做测试。现在 Github 平台上主要使用的 CI 服务有 Travis, Azure Pipelines, CircleCI 以及 Appveyor 等。

Travis 是老牌的 CI 服务，Azure Pipelines 有财大气粗的微软撑腰，免费服务很不错。Appveyor 有着经典的 Windows 支持。而 CircleCI 作为后起之秀，表现也可圈可点。

由于 PyTorch 主要使用 CircleCI 进行发布以及测试，因此我也详细体验过其推出的Windows支持。其主要的硬伤主要在于两点：

1. 系统支持少。仅支持 Windows Server 2019。

2. 不支持建立远程桌面连接来调试，仅支持 SSH 用于调试。

第一个问题并不大，因为一般来说新代码一般不会有太大的兼容性问题。但是第二点确实是比较大的问题，因为 Windows 有很多工具链都是基于 GUI 的，在 CircleCI 的 SSH 调试环境中连 VIM 都几乎没法用，更不用说其他更加复杂的调试需求了。

为了解决这个问题，我找到了这样一篇相关的[文章](https://circleci.com/docs/2.0/browser-testing/)，介绍了在 Linux / MacOS 上如何进行调试，其中"Using a Local Browser to Access HTTP server on CircleCI"这一章中介绍了如何使用 SSH 进行端口转发。

```sh
ssh -p 64625 ubuntu@54.221.135.43 -L 3000:localhost:8080
```

我们先用一个简易python服务器试试水。

```powershell
# CMD窗口1
ssh -p 64625 circleci@54.221.135.43 -L 8000:localhost:8000 -- cmd.exe
python -m http.server

# CMD窗口2
curl http://localhost:8080
```

<p align="center">
    <img src="/img/ssh-port-forwarding-simple-test.png">
</p>

看上去貌似是可以工作的。那我们照猫画虎转发到RDP的端口行不行呢？

```powershell
ssh -p 64625 circleci@54.221.135.43 -L 3389:localhost:8080 -- cmd.exe
```

结果：

```
bind: Address already in use
error: bind: Address already in use
error: channel_setup_fwd_listener: cannot listen to port: 3389
```

会提示我们端口已被占用，无法直接监听。那么其实我们可以绕个弯，先用 ssh 来转发到未监听的端口，再用本地的端口转发到目标端口 3389 不就行了吗？

尝试如下：

```powershell
# CMD窗口
ssh -p 64625 circleci@54.221.135.43 -L 8000:localhost:8000 -- cmd.exe
netsh interface portproxy add v4tov4 listenport=8000 connectaddress=127.0.0.1 connectport=3389
```

然后打开远程桌面，连接localhost:8000，发现已经连通了。

<p align="center">
    <img src="/img/ssh-port-forwarding-remote-test.png">
</p>

但是密码是啥呢？不知道密码就没法登录啊。然而其实我们并不需要知道密码。由于 CircleCI 的账户自带管理员权限，因此更改密码是非常简单的一件事情。

```powershell
# CMD窗口
net user circleci [新密码]
```

就这一行就好了。当然需要注意的是 Windows Server 对于密码具有很高的要求，需要包含大小写字母、特殊符号、数字，以及足够的长度（貌似最小长度12），另外不能包含用户名，所以需要设置一个强度足够高的密码。

最后，就可以开始尽情的调试了。

<p align="center">
    <img src="/img/ssh-port-forwarding-final-result.png">
</p>
