---
title: 在Apache的部署Django项目
category: Django
---

## 说明
Django是一个Python web应用，所以可以用Python的[WSGI](http://wsgi.readthedocs.org/en/latest/)部署在常用的Web服务器上，如Apache。
Apache的mod_wsgi实现了Python的WSGI，所以在Apache上部署Django项目时还需安装mod_wsgi模块。

## 安装Aapache和mod_wsgi模块

目前我用Ubuntu，在Ubuntu上安装是非常简单的。输入命令：

```bash
sudo apt-get install apache2 libapache2-mod-wsgi   #如果用Python3需要安装libapache2-mod-wsgi-py3
```

安装完后可进入`/etc/apache2/`目录查看apache的配置，现Ubuntu版的Apache配置结构如下所示：

```bash
/etc/apache2/

|--apache2.conf
|--conf-available/
|	--apache2-doc.conf配置
|	--charset.conf
|	--.....
|--conf-enabled/
|	--*.conf #link file
|--envvars
|--magic
|--mods-available/
|	--*.load
|	--*.conf
|--mods-enabled/enter link description here
|	--*.load	#link file
|	--*.conf	#link file
|--prots.conf
|--sites-available/
|	--000-default.conf
|--sites-enabled/
|	--000-default	#link file
```

逐一说明下：
1. **apache2.conf**: Apache2 的主要配置文件。 包含了 Apache2 的全局的配置。
2. **httpd.conf**:  (不一定有，老的Apache2配置)historically the main Apache2 configuration file, named after the httpd daemon. Now the file does not exist. In older versions of Ubuntu the file might be present, but empty, as all configuration options have been moved to the below referenced directories.
3. **conf-available**: 这个目录包含可以使用的配置文件。以前版本使用的/etc/apache2/conf.d目录中所有配置都移到此目录下了。
4. **conf-enabled**:  里面都是符号文件，指向/etc/apache2/conf-available。如果这个符号文件指向conf-available/目录下的某个配置文件，代表这个配置文件激活。
5. **envvars**： Apache2 环境变量设置。
6. **magic**： instructions for determining MIME type based on the first few bytes of a file.
7. **mods-available**： 该目录包含的配置文件都装载 模块 和设置它们。不管怎样并非所有模块都会有具体的配置文件。
8. **mods-enabled**：保持符号链接文件在 /etc/apache2/mods-available。当一模块配置文件被设为符号连接后会在下一次apache2重启时激活。
9. **ports.conf**：指示以确定 Apache2 正在监听哪些 TCP 端口。
10. **sites-available**: 这个目录下有 Apache2 虚拟主机 的配置文件。虚拟主机使 Apache2 能够配置多个站点，这些站点有各自不同的配置。
11. **sites-enabled:** 跟mods-enabled, sites-enabled一样。在这里简单的设置一个链接文件链接到/etc/apache2/sites-available的网站配置，这个网站就会在下一次Apache2启动时激活。

## 配置Apache2的mod_wsgi模块

使用`sudo apt-get install libapache2-mod-wsgi`的话mod_wsgi应该在Apache2配置好了。如果是手动编译和安装，需要在mods-available配置文件设置Apache2装载mod_wsgi模块，并在mods-enable中链接该配置。

设置好的配置类似这样：
```bash
# /etc/apache2/mods-available/wsgi.load
 LoadModule wsgi_module /usr/lib/apache2/modules/mod_wsgi.so

# /etc/apache2/mods-enable/wsgi.load -> ../mods-available/wsgi.load

```

也使用命令`a2enmod wsgi`激活该模块，如果提示未找到模块，那就要查看模块是否安装以及配置好。


## 配置Apache2网站

在`/ect/apache2/site-available/`新建一个**sitename.conf**文件，其中sitename是网站名字，可以随意取名。

编辑**sitename.conf**文件：

```bash
WSGIPythonPath /path/to/mysite.com
<VirtualHost *:80>
    WSGIScriptAlias / /path/to/mysite.com/mysite/wsgi.py
    <Directory /path/to/mysite.com/mysite>
    <Files wsgi.py>
        Require all granted
    </Files>
    </Directory>
</VirtualHost>
```

通过设置链接文件激活该网站，或者使用命令`a2ensite sitename.conf`，然后重启Apache2，`sudo apache2ctl restart`，在浏览器打开127.0.0.1即可看到Django应用。

## 更多配置

上述配置仅测试Django是否能在Apache2上部署成功，想要Django应用运行得好还要作许多配置，如静态文件读取，文件写入权限，多个Django应用部署等等。

待续。。

## 参考
1. [How to use Django with Apache and mod_wsgi](https://docs.djangoproject.com/en/1.8/howto/deployment/wsgi/modwsgi/)
2. [mod_wsgi document](http://modwsgi.readthedocs.org/en/develop/index.html)
3. [HTTPD - Apache2 Web 服务器](https://help.ubuntu.com/lts/serverguide/httpd.html)
