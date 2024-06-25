---
title: CentOS7安装Nginx
date: 2024-06-25 16:32:00 +0800
categories: [Linux, CentOS7]
tags: [nginx]

---



### 1、安装方式

Centos7中可通过yum工具以及自定义安装压缩包的方式进行安装Nginx，二者特点如下：

**yum**

安装便捷，但yum源中版本较低

**压缩包**（较为推荐）

安装步骤相对麻烦，但定制化程度较高，可自定义安装位置等 

---



### 2、压缩包安装

#### 2.1：安装流程

**前提**：通过压缩包的方式进行安装，需要提前安装一些nginx依赖库，如下所示：

```shell
# nginx 编译时依赖 gcc 环境
yum -y install gcc gcc-c++ 
# 让 nginx 支持重写功能
yum -y install pcre pcre-devel 
# zlib 库提供了很多压缩和解压缩的方式，nginx 使用 zlib 对 http 包内容进行 gzip 压缩
yum -y install zlib zlib-devel 
# 安全套接字层密码库，用于通信加密
yum -y install openssl openssl-devel
```



(1)：新建目录存放压缩包（可自定义，不必完全参照本文）

```shell
mkdir /usr/local/nginx
cd /usr/local/nginx
```

![image-20231219113745835](https://s2.loli.net/2024/06/25/JtxgeHqSLCGZk1i.png) 

(2)：打开nginx官网

```http
https://nginx.org/en/download.html
```

(3)：选择合适的版本进行下载，这里有两种下载方式

​	3.1：点击链接自行下载后通过SSH等工具上传到服务器目录中

​	3.2：通过**wget**命令进行下载

​		①：右键链接，复制链接地址

![image-20231219113356081](https://s2.loli.net/2024/06/25/hNtoDjqrXQ8g9Mk.png) 

​		②：使用wget命令进行下载

```shell
wget https://nginx.org/download/nginx-1.24.0.tar.gz
```

![image-20231221183136601](https://s2.loli.net/2024/06/25/IKJEmcpfBoY386d.png) 

(4)：查看下载好的压缩包

![image-20231219114226588](https://s2.loli.net/2024/06/25/CO5GKiQ9DnavTx7.png) 

(5)：解压压缩包

```shell
# 这里要替换为你下载的nginx压缩包名
tar -zxvf nginx-1.24.0.tar.gz
```

![image-20231219114541608](https://s2.loli.net/2024/06/25/2XfyEciT4WJoZnY.png) 

(6)：查看并进入解压后的nginx目录

```shell
ll
# 这里要替换为你解压后的目录名
cd nginx-1.24.0/
```

![image-20231219114625588](https://s2.loli.net/2024/06/25/Xwfc7DN2oO5pUSu.png) 

(7)：执行configure命令对nginx作初始化配置

```shell
# --prefix=/usr/local/nginx 指定编译安装目录
./configure --prefix=/usr/local/nginx

# 详尽版命令（正常情况下使用上文中简化命令即可）
./configure 
--prefix=/usr/share/nginx  #指定编译安装目录
--sbin-path=/usr/sbin/nginx  #指定可执行文件路径
--modules-path=/usr/lib64/nginx/modules  #指定模块路径
--conf-path=/etc/nginx/nginx.conf #指定配置文件路径
--error-log-path=/var/log/nginx/error.log  #执行错误日志路径
--http-log-path=/var/log/nginx/access.log  #执行http访问日志路径
--pid-path=/run/nginx.pid  #指定主进程的pid文件路径
--lock-path=/run/lock/subsys/nginx  #指定锁文件的路径
--user=nginx  #指定运行nginx服务的用户名
--group=nginx  #指定运行nginx服务器的用户组
--with-http_stub_status_module  #启用nginx的HTTP-Stub-Status 模块，用于统计
--with-http_ssl_module #启用nginx的https支持，即启用SSL模块
```

执行最终结果 

![image-20231219120952523](https://s2.loli.net/2024/06/25/fvnZwR2NrKpPBGg.png) 

(8)：先后执行make命令进行编译、安装

```shell
make
make install
```

![image-20231219121403581](https://s2.loli.net/2024/06/25/zqeMIRaZvWdUsoi.png) 

![image-20231219121433814](https://s2.loli.net/2024/06/25/scGEVXYW4ke2qj6.png) 

(9)：进入nginx编译安装目录

```shell
# 这里要替换为执行configuration命令时--prefix指定的目录下的/sbin，即你所指定的编译安装目录
cd /usr/local/nginx/sbin
```

![image-20231219121606567](https://s2.loli.net/2024/06/25/frWJliMDz69X3K2.png) 

(10)：启动nginx

```shell
./nginx
```

![image-20231219121846673](https://s2.loli.net/2024/06/25/FCIy2oz5cgYHZwa.png) 

(11)：查看nginx进程

```shell
ps -ef |grep nginx
```

正常来说，出现 auto nginx之外的进程即可

![image-20231219123601261](https://s2.loli.net/2024/06/25/yPWCigDqhVT8Suf.png) 

(12)：防火墙开放80端口，nginx默认代理80端口。

```shell
# 开启防火墙
systemctl start firewalld
# 开放80端口，用于访问nginx
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
# 重新加载防火墙配置
sudo firewall-cmd --reload
```

![image-20240106203601708](https://s2.loli.net/2024/06/25/9OEiAwm8pJyrWG5.png)  

(13)：浏览器中输入服务器ip + 80端口，出现以下页面即代表成功

```http
# 示例
192.168.244.130:80
```

![image-20231219124024097](https://s2.loli.net/2024/06/25/zuY6F8EWmQs7Nqx.png) 

#### 2.2：功能扩展

(1)：如要修改配置，则进入安装目录下的conf目录，对**nginx.conf**进行编辑（具体如何配置本文不再赘述，自行查阅）

![image-20231219124549809](https://s2.loli.net/2024/06/25/DCPpueILyx8hB6H.png)  

(2)：常用命令

```shell
# 注：此命令必须在nginx下的sbin目录执行，否则需要以绝对或相对路径的方式指定位置，例如：/usr/local/nginx/sbin/nginx 启动nginx
./nginx            --启动nginx
./nginx -v		   --查看nginx版本
./nginx -s stop    --强制关闭nginx
./nginx -s quit	   --完成当前任务后停止服务，与stop命令效果相当
./nginx -s reload  --修改配置文件后使用，否则修改不生效（如果停止服务后再启动，则不需要此命令）
```

#### 2.3：卸载nginx

(1)：查看nginx服务是否运行，如果处于运行状态，则执行命令暂停服务。具体命令查看上文

```shell
ps -ef |grep nginx
```

![image-20231221183619900](https://s2.loli.net/2024/06/25/XoNcURCDnyGpJLa.png) 

(2)：删除nginx编译安装目录，至此nginx卸载成功

```shell
# 替换为自己nginx安装编译目录，即执行configure命令时指定的--prefix参数
# 慎用rm -rf 命令，尤其是生产环境，会不作任何提示直接删除！！！！
rm -rf /usr/local/nginx
```

![image-20231221183938089](https://s2.loli.net/2024/06/25/FzbOcW8BoSmLVqj.png) 

---



### 3、yum安装

yum安装方式来自官网，通过这种方式进行安装时，版本一般为最新的稳定版

#### 3.1：安装流程

(1)：安装yum-utils

yum-utils 是一个yum工具集，它提供了一些常用的命令和插件，以便管理和维护yum软件包管理器。

```shell
yum install yum-utils
```

![image-20231219132339602](https://s2.loli.net/2024/06/25/hYTDrUeFSxBbltj.png) 

(2)：设置nginx相关存储库

```shell
# 生成文件
vim /etc/yum.repos.d/nginx.repo
```

![image-20231219132618689](https://s2.loli.net/2024/06/25/uWBi2O5vFDxCJIL.png) 

(3)：按英文**i**进行插入，粘贴以下内容

```shell
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

(4)：退出vim并保存文件 --- 按键盘esc，然后在英文状态下输入:wq后回车

![image-20231219132752562](https://s2.loli.net/2024/06/25/dkxCfE5mvuKUNnw.png) 

(5)：安装nginx

```shell
yum install -y nginx
```

![image-20231219134852880](https://s2.loli.net/2024/06/25/lDFzLQc7u6odUWs.png) 

(6)：查看nginx版本

```shell
nginx -v
```

![image-20231219133151026](https://s2.loli.net/2024/06/25/Y1iCcU7JsFSNqkV.png) 

(7)：启动nginx服务

```shell
# 启动nginx服务
systemctl start nginx
```

![image-20231219134404450](https://s2.loli.net/2024/06/25/xEDno5ch2b7Te8K.png)  

(8)：查看nginx进程，是否启动成功

```shell
ps -ef |grep nginx
```

正常来说，出现 auto nginx之外的进程即可

![image-20231219133623410](https://s2.loli.net/2024/06/25/E5qdIQfCUTezl4L.png) 

(9)：防火墙开放80端口，nginx默认代理80端口。

```shell
# 开启防火墙
systemctl start firewalld
# 开放80端口，用于访问nginx
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
# 重新加载防火墙配置
sudo firewall-cmd --reload
```

![image-20240106203601708](https://s2.loli.net/2024/06/25/9OEiAwm8pJyrWG5.png) 

(10)：浏览器中输入服务器ip + 80端口，出现以下页面即代表成功

```http
# 示例
192.168.244.130:80
```



![](https://s2.loli.net/2024/06/25/JV6KlkHjI4yvwpB.png) 

#### 3.2：功能扩展
(1)：如要修改配置，则进入/etc/nginx目录，修改nginx.conf文件(具体如何配置不作讲解，自行查阅)

```shell
vim /etc/nginx/nginx.conf
```

![image-20231219154215397](https://s2.loli.net/2024/06/25/yW2NzjLoqTghAiM.png) 

(2)：常用命令

```shell
# 开机自启动
systemctl enable nginx
# 启动nginx服务
systemctl start nginx
# 重启nginx服务
systemctl restart nginx
# 重新加载配置(用于修改配置文件后又不想重启的情况)
systemctl reload nginx
```

#### 3.3：卸载nginx

(1)：查看nginx服务是否运行，如果处于运行状态，则执行命令暂停服务。具体命令查看上文

```shell
ps -ef |grep nginx
```

![image-20231221184549777](https://s2.loli.net/2024/06/25/68H1TpY53GSDKUx.png)  

(2)：执行命令卸载nginx

```
yum -y remove nginx
```

![image-20231221184625911](https://s2.loli.net/2024/06/25/A7R2kj5LmFbeZWD.png) 

(3)：查看nginx残余目录

```shell
find / -name nginx
```

![](https://s2.loli.net/2024/06/25/Z31Kc2WaNEACnkx.png) 

(4)：删除nginx残余目录，至此nginx卸载完成

```shell
# 慎用rm -rf 命令，尤其是生产环境，会不作任何提示直接删除
# $() 组合命令
rm -rf $(find / -name nginx)
```

![image-20231221184834683](https://s2.loli.net/2024/06/25/OIZQy87mehsToSv.png) 
