---
layout: post
title: 新版本Django在Windows和Linux下的部署
tags: [Django, Windows, Linux, Python]
---
伴随着人工智能和深度学习的火热发展，Python也成为了一门非常热门的语言。我们可以看到热门的深度学习框架都提供Python的接口，有些甚至只有Python的接口，这一定程度上推动了Python的普及。当然，在我们完成模型的训练之后，我们总是要搭建一个演示的平台，要么开放给用户使用，要么是内部进行测试。

虽然说Python是著名的胶水语言，可以和很多语言和框架进行互操作，但是要是能直接使用Python编写的框架进行服务器的部署的话，则可以简化程序的编写过程。Python下最著名的网站框架就是Django了，因此今天就介绍下新版本Django的部署方法，主要以Windows和Linux为例。那为何要写这篇呢？因为Django这几年也变化了很多，然而网上有很多陈旧的部署教程，会对我们的部署过程进行干扰，所以我觉得有这个必要好给你们传授下一些部署的经验。当然，部署前请用python manage.py runserver检查一下服务器是否正常工作。

我们先看看Windows下怎么进行Django的部署，我借助的是Azure中的Django模板中所使用wfastcgi库以及微软自己的服务器平台IIS。过程如下：

1. 安装IIS（实测需要6或者7.5及以上，安装包在[此](https://www.microsoft.com/zh-CN/download/details.aspx?id=1038)），请务必勾选服务器技术中的CGI。

2. 安装wfastcgi

    ```powershell
    pip install wfastcgi
    ```

3. 执行指令，获得FastCGI的Handler参数。

    ```powershell
    wfastcgi-enable
    ```
    会获得一串类似`c:\anaconda2\python.exe|c:\anaconda2\lib\site-packages\wfastcgi.pyc`的字符串，其实就是python的路径和handler的路径用|进行连接而成的字符串，记录下来，后面会用到。

4. 在网站根目录下新建一个web.config,填入以下内容

    ```xml
    <configuration>
      <system.webServer>
        <handlers>
          <add name="Python FastCGI"
              path="*"
              verb="*"
              modules="FastCgiModule"
              scriptProcessor="C:\Python36\python.exe|C:\Python36\Lib\site-packages\wfastcgi.py"
              resourceType="Unspecified"
              requireAccess="Script" />
        </handlers>
      </system.webServer>

      <appSettings>
        <!-- Required settings -->
        <add key="WSGI_HANDLER" value="django.core.wsgi.get_wsgi_application()" />
        <add key="PYTHONPATH" value="C:\MyApp" />

        <!-- Optional settings -->
        <add key="WSGI_LOG" value="C:\Logs\my_app.log" />
        <add key="WSGI_RESTART_FILE_REGEX" value=".*((\.py)|(\.config))$" />
        <add key="APPINSIGHTS_INSTRUMENTATIONKEY" value="__instrumentation_key__" />
        <add key="DJANGO_SETTINGS_MODULE" value="my_app.settings" />
        <add key="WSGI_PTVSD_SECRET" value="__secret_code__" />
        <add key="WSGI_PTVSD_ADDRESS" value="ipaddress:port" />
      </appSettings>
    </configuration>
    ```
    上面一段是样例的web.config文件,摘自wfastcgi的官方介绍[页面](https://pypi.python.org/pypi/wfastcgi)，我们需要做一些更改，首先scriptProcessor改为上面我们记录的那段字符串，然后是一些必要的参数，像PYTHONPATH改为网站的根目录的路径，最后就是一些可选的参数，像WSGI_LOG是log文件的路径，还有DJANGO_SETTINGS_MODULE需要改为自己项目的名称，像APPINSIGHTS_INSTRUMENTATIONKEY，WSGI_PTVSD_SECRET,WSGI_PTVSD_ADDRESS这些都是Azure上的参数，我们可以将其删除。

5. 在IIS新建一个站点，指向网站的根目录，注意端口不要重复。

    <p align="center">
      <img src="/img/django_deploy_windows.jpg">
    </p>

6. 打开浏览器进行访问，不出意外已经就OK了。

7. 如果还是有问题，一般是权限问题，检查下Anaconda的目录对于IIS_IUSRS是否有可读、可执行、列出目录权限。

上面的部署过程在IIS 6.0以及7.5及以上都测试通过了，基本没啥大问题。而且通过上面设置的WSGI_RESTART_FILE_REGEX参数，可以实现自动重启，简直不要太爽。

之后是在Linux下的部署，我原本想着会比Windows简单的多，但是自从体验过了DigitalOcean的一键部署的镜像后，我感觉我还是too simple了。提醒大家千万别用DO中Ubuntu16.04的Django的一键镜像，不知道为什么突然就挂了，而且一上来还不能用域名访问。Ubuntu14.04的Django的一键镜像还行，就是Django版本太老了，一升级又挂了。所以我还是从头开始吧，这次主要是使用nginx和uwsgi来进行部署的，过程如下：

1. 遵照nginx官方[教程](http://nginx.org/en/linux_packages.html#stable)进行nginx的安装，一般是下载证书，并添加到本机，刷新库安装软件的缓存，再进行安装这样一个流程。此处以Ubuntu/Debian为例,在bash中敲入：

    ```bash
    wget http://nginx.org/keys/nginx_signing.key
    sudo apt-key add nginx_signing.key
    ```

   再修改/etc/apt/sources.list，添加以下两行

    ```bash
    deb http://nginx.org/packages/system/ codename nginx
    deb-src http://nginx.org/packages/system/ codename nginx
    # system按情况替换为debian或ubuntu，codename可通过lsb_release -a获得，然后继续执行
    apt-get update
    apt-get install nginx
    ```

2. 确保Python及pip已经安装，可以通过以下指令安装。

    ```bash
    apt-get install python-dev python-pip
    ```

3. 安装uwsgi

    ```bash
    pip install uwsgi
    ```

4. 像上面一样，在根目录下面新建一个配置文件uwsgi.ini，填入以下内容。

    ```
    [uwsgi]
    socket = 127.0.0.1:8001
    chdir = /path/to/your/project/
    wsgi-file = project_name/wsgi.py
    master = true
    processes = 1
    vacuum = true
    optimize = 2
    stats = 127.0.0.1:9191
    daemonize = /var/log/uwsgi/myapp.log
    py-autoreload = 1
    ```

    chdir修改为项目的根目录，project_name修改为自己的项目名称，daemonize则是日志文件的路径，socket的端口可以灵活进行调整。可以通过py-autoreload来进行自动重启，但是只有py文件修改才会重启，这个不如之前Windows的正则方便。

5. 执行命令将uwsgi服务器跑起来

    ```bash
    uwsgi uwsgi.ini
    ```

6. 按照nginx的[教程](https://www.nginx.com/resources/admin-guide/gateway-uwsgi-django/)，我们修改nginx的配置文件，新版本的路径一般在/etc/nginx/conf.d/default.conf。

    ```nginx
    upstream django {
        server 127.0.0.1:8001;
    }

    server {
        listen 80;
        server_name myapp.example.com;

        root /var/www/myapp/html;

        location / {
            index index.html;
        }

        location /static/  {
            alias /var/django/projects/myapp/static/;
        }

        location /main {
            include uwsgi_params;
            uwsgi_pass django;

            uwsgi_param Host $host;
            uwsgi_param X-Real-IP $remote_addr;
            uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
            uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;
        }
    }
    ```
    
    请保证上游django路由的端口与之前socket设置的一致，static目录和其他配置等可以灵活进行调整。

7. 如果你现在直接跑的话，可以发现网站可以正常跑，但是静态文件却是有问题的。这还是权限问题，因为nginx默认不是用root用户跑的，他没有访问静态目录的权限。这有两种解决方法，一个是给nginx用户静态目录的访问权限，还有一种就是让nginx以root用户来跑（不推荐）。前者可以通过配置目录的权限或是将静态文件存放在非特权目录中来完成，后者则是将`/etc/nginx/nginx.conf`中的`user nginx`修改为`user root`。在更改配置后再使用`service nginx restart`重启服务即可生效。

8. 如果对项目做了更改，他并不会自动生效，需要先将之前的uwsgi进程kill掉，再次运行步骤5即可。
