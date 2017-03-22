## Mac OS下安装及配置nginx

### 安装包
- 下载 Nginx 源码包

    下载页: http://nginx.org/en/download.html

    当前稳定版本: http://nginx.org/download/nginx-1.8.1.tar.gz



- zlib

    下载页: http://zlib.net/
    
    当前稳定版本: http://zlib.net/zlib-1.2.8.tar.gz

    注: Nginx 参考文档中提到需要 1.1.3 以上版本的 zlib

- pcre

    下载页: http://www.pcre.org/

    当前稳定版本: ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.38.tar.gz

    注: Nginx 参考文档中提到需要 4.4 以上版本的 pcre

### 配置及安装

- __First, 进入到包所在的文件夹内__
```
tar zxvf zlib-1.2.8.tar.gz  # 得到 zlib-1.2.8 目录
tar zxvf pcre-8.36.tar.gz  # 得到 pcre-8.36 目录
```
- 编译安装 Nginx

    这里会将各依赖的源码编译进 Nginx, 所以 --with-zlib 和 --with-pcre 后为对应的依赖源码目录路径。此外, 编译选项中还开启了 HTTPS 的协议支持--with-http_ssl_module, 若不需要 HTTPS, 可取消该选项。

    ```
    tar zxvf nginx-1.8.0.tar.gz
    cd nginx-1.8.0
    ./configure --prefix=/usr/local/nginx --with-zlib=../zlib-1.2.8 --with-pcre=../pcre-8.36
    make
    sudo make install
    ```
- 其他安装指令（可做了解）
    ```
    nginx大部分常用模块，编译时./configure --help以--without开头的都默认安装。
    
    --prefix=PATH ： 指定nginx的安装目录。默认 /usr/local/nginx
    --conf-path=PATH ： 设置nginx.conf配置文件的路径。nginx允许使用不同的配置文件启动，通过命令行中的-c选项。默认为prefix/conf/nginx.conf
    --user=name： 设置nginx工作进程的用户。安装完成后，可以随时在nginx.conf配置文件更改user指令。默认的用户名是nobody。--group=name类似
    --with-pcre ： 设置PCRE库的源码路径，如果已通过yum方式安装，使用--with-pcre自动找到库文件。使用--with-pcre=PATH时，需要从PCRE网站下载pcre库的源码（版本4.4 - 8.30）并解压，剩下的就交给Nginx的./configure和make来完成。perl正则表达式使用在location指令和 ngx_http_rewrite_module模块中。
    --with-zlib=PATH ： 指定 zlib（版本1.1.3 - 1.2.5）的源码解压目录。在默认就启用的网络传输压缩模块ngx_http_gzip_module时需要使用zlib 。
    --with-http_ssl_module ： 使用https协议模块。默认情况下，该模块没有被构建。前提是openssl与openssl-devel已安装
    --with-http_stub_status_module ： 用来监控 Nginx 的当前状态
    --with-http_realip_module ： 通过这个模块允许我们改变客户端请求头中客户端IP地址值(例如X-Real-IP 或 X-Forwarded-For)，意义在于能够使得后台服务器记录原始客户端的IP地址
    --add-module=PATH ： 添加第三方外部模块，如nginx-sticky-module-ng或缓存模块。每次添加新的模块都要重新编译（Tengine可以在新加入module时无需重新编译）
    ```
    比如：
    ```
    ./configure \
    > --prefix=/usr \
    > --sbin-path=/usr/sbin/nginx \
    > --conf-path=/etc/nginx/nginx.conf \
    > --error-log-path=/var/log/nginx/error.log \
    > --http-log-path=/var/log/nginx/access.log \
    > --pid-path=/var/run/nginx/nginx.pid  \
       > --lock-path=/var/lock/nginx.lock \   
    > --user=nginx \
    > --group=nginx \
    > --with-http_ssl_module \
    > --with-http_stub_status_module \
    > --with-http_gzip_static_module \
    > --http-client-body-temp-path=/var/tmp/nginx/client/ \
    > --http-proxy-temp-path=/var/tmp/nginx/proxy/ \
    > --http-fastcgi-temp-path=/var/tmp/nginx/fcgi/ \
    > --http-uwsgi-temp-path=/var/tmp/nginx/uwsgi \
    > --with-pcre=../pcre-7.8
    > --with-zlib=../zlib-1.2.3
    ```
    
### 设置监听

进入配置文件/usr/local/nginx/conf/nginx.conf ，修改listen下参数：

```
server {
        listen       85;            #监听端口
        server_name  localhost;     #监听服务器

        #charset koi8-r;
        #access_log  logs/host.access.log  main;
        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```
### 启动nginx

```
cd /usr/local/nginx

#启动
sudo sbin/nginx     #浏览器访问 127.0.0.1 测试是否成功启动

#重启
sudo sbin/nginx -s reload

#停止
sudo sbin/nginx -s stop
```

### 修改返回值

以bandwidth请求：

http://127.0.0.1:8001/v2/bandwidth

按照请求的格式创建目录为：/usr/local/nginx/html/v2/   
（/usr/local/nginx/html/为默认目录,/v2/为根据请求格式新建的目录）

新建bandwidth文件，则文件路径为/usr/local/nginx/html/v2/bandwidth，写入返回值，则构造指定返回报文成功，可按照自己需求进行编辑；

然后在浏览器中请求这个接口，返回自定义的内容则配置成功。
