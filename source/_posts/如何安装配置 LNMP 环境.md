title: 如何安装配置 LNMP 环境
category: Technologies
date: 2018-03-03
tags:
- Linux
- LNMP
- 环境搭建
thumbnail: https://blog-1253392534.cossh.myqcloud.com/img/lnmp.jpg
lede: "Nginx, PHP, MySQL 之间究竟是怎样联系的？"
featured: true

---

# 介绍

LNMP 由以下四个部分构成：

* [Linux](https://zh.wikipedia.org/wiki/Linux) 是运行下面三个软件的操作系统。
* [Nginx](https://zh.wikipedia.org/wiki/Nginx) 是一款面向性能设计的 HTTP 服务器，相较于 Apache、lighttpd 具有占有内存少，稳定性高等优势。
* [MySQL](https://zh.wikipedia.org/wiki/MySQL) 是一款性能高、成本低、可靠性好的数据库。
* [PHP](https://zh.wikipedia.org/wiki/PHP) 是目前常用于编写网页的脚本语言。

本文将一步一步搭建 LNMP 服务。

<!-- more -->

# Step 0：准备工作



你需要一台16.04及以上版本的 Ubuntu 主机，并执行以下命令确保系统及软件源为最新

```bash
sudo apt update
sudo apt upgrade
```

一旦完成升级，就可以开始进行 LNMP 环境的搭建了。

# Step 1：安装Nginx



为了让来访者能看见我们的网页，我们要先安装 Nginx。

在此过程中我们安装的软件都将来自 Ubuntu 官方的软件源，这意味着我们可以使用`apt`软件包管理器进行安装。

安装 Nginx：

```bash
sudo apt install nginx
```

在 Ubuntu16.04 上，Nginx 安装完成后会自动运行。

安装完成后，在浏览器上访问你的IP，不出意外的话应该会显示类似这样的欢迎页面：

![](https://blog-1253392534.cossh.myqcloud.com/img/welcome-to-nginx.png)


如果你看到了这个页面，说明 Nginx 已经安装好并能正常工作了。

# Step 2：安装 MySQL



现在我们有了一个简易的 web 服务器，我们还需要一个数据库来保存我们的数据。

通过执行以下命令安装 MySQL：

```bash
sudo apt install mysql-server
```

安装过程中 MySQL 会要求你设置数据库 root 密码。

# Step 3：安装 PHP



我们现在已经安装了 Nginx 来为我们提供 web 服务，并安装了 MySQL 来存储和管理我们的数据。但是我们还没有任何可以生成动态内容的东西。 我们使用 PHP 来实现这一目的。

由于 Nginx 不像其他 web 服务器那样包含原生的 PHP 处理，所以我们需要安装 `php-fpm`。我们会告诉 Nginx 将 PHP 请求传递给这个软件进行处理。

我们还将安装`php-mysql`，这将允许 PHP 与我们的数据库进行通信。安装将会拉取必要的 PHP 核心文件。

通过执行以下命令安装：

```bash
sudo apt install php-fpm php-mysql
```

现在 PHP 组件已经安装好了，为了安全考虑，我们需要改动一个小设置。

打开`php-fpm`的配置文件：

```bash
sudo nano /etc/php/7.0/fpm/php.ini
```

我们要找的参数是`cgi.fix_pathinfo`，这个参数设置默认是以分号（;）注释掉的，并且默认值为1，我们去掉注释，并把值改为0：`cgi.fix_pathinfo=0`，保存并退出。

这是一个非常不安全的设置，因为它告诉 PHP 如果找不到请求的 PHP 文件，就尝试执行它可以找到的最接近的文件。这将允许用户执行本不应该或不允许被执行的脚本。

现在，我们需要重启下 PHP 解析器：

```bash
sudo systemctl restart php7.0-fpm
```

这个改变就会即时生效。

# Step 4：配置 Nginx 解析 PHP



现在我们所有需要的组件都已经安装好了，唯一还要更改的配置就是告诉 Nginx 使用我们的 PHP 解析器来处理动态内容。

打开默认的 Nginx 配置文件：

```bash
sudo nano /etc/nginx/sites-available/default
```

不出意外的话，你的配置文件去掉注释后应该长这样：

```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		try_files $uri $uri/ =404;
	}
}
```

我们需要做一些改动。

* 首先，我们需要将`index.php`添加为`index`指令的第一个值，以便默认请求`index.php`。
* 我们可以修改`server_name`指令来指向我们的服务器域名或 IP 地址。
* 对于实际的PHP处理，我们只需要通过从每行前面删除井号（#）来取消注释处理 PHP 请求的配置。这个代码块位于`location ~\.php$`，包含`fastcgi-php.conf`的代码片和与`php-fpm`关联的套接字。
* 我们还将使用相同的方法取消处理`.htaccess`文件的代码块。 Nginx不处理这些文件。 如果这些文件中的任何一个文件碰巧在文档根目录中，则不应将其提供给访问者。

做完改动后应该是这个样子：

```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /var/www/html;
	index index.php index.html index.htm index.nginx-debian.html;

	server_name 服务器域名或IP;

	location / {
		try_files $uri $uri/ =404;
	}

	location ~ \.php$ {
		include snippets/fastcgi-php.conf;
		fastcgi_pass unix:/run/php/php7.0-fpm.sock;
	}

	location ~ /\.ht {
		deny all;
	}
}
```

做完改动后就可以保存并关闭文件了。

通过以下命令测试配置文件是否正确：

```bash
sudo nginx -t
```

如果没有问题的话，应该返回如下结果：

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

如果报错了的话，请在下一步前检查你的配置。

通过以下命令重启 Nginx 服务：

```bash
sudo systemctl reload nginx
```

# Step 5：创建一个测试页面



此时你的 LNMP 环境就完全搭建好了，我们可以测试一下 Nginx 能否正确通过 PHP 解析器解析`.php`文件。

在网站根目录新建一个测试 PHP 文件：

```bash
sudo nano /var/www/html/info.php
```

填入以下代码：

```php
<?php
phpinfo();
```

保存并关闭文件。

现在，你可以通过域名或 IP 来访问你的网页了：

```
http://服务器域名或IP/info.php
```

你应该能看见一个由 PHP 生成的服务器信息页面：

![](https://blog-1253392534.cossh.myqcloud.com/img/phpinfo.png)

至此，LNMP环境搭建就已经完成了。

——参考：[DigitalOcean: How To Install LEMP stack in Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-in-ubuntu-16-04#step-2-install-mysql-to-manage-site-data)