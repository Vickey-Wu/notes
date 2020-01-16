#### [编译安装](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/#downloading-the-sources)
- ubuntu update
```
sudo apt-get update
sudo apt-get install -y libpcre3 libpcre3-dev build-essential libtool zlib1g-dev openssl*
```
- centos update(alternative)

```
sudo yum update
sudo yum -y install gcc gcc-c++ automake pcre pcre-devel zlib zlib-devel openssl openssl-devel
sudo yum update
```

- add group & user

```
groupadd nginx && useradd nginx -g nginx -s /sbin/nologin -M
```

- compile & install

```
wget -P /usr/src https://nginx.org/download/nginx-1.17.6.tar.gz
tar -xvf /usr/src/nginx-1.17.6.tar.gz

# --add-module use for adding 3party_module, bin file default dir: /usr/local/nginx
cd /usr/src/nginx-1.17.6
./configure \
  --user=nginx \
  --group=nginx \
  --with-stream \
  --with-stream_ssl_module \
  --add-module=/{external_module_dir}/nginx-rtmp-module \
  --add-dynamic-module=/{external_module_dir}/{3party_module}

# if no make `apt install make` or `yum install make`
sudo make && sudo make install
```

- [config arg explanation](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/#configuring-nginx-paths)

```
--prefix=<nginx_install_path>       nginx install path default: /usr/local/nginx
--sbin-path=<nginx_install_path>    nginx exec file path default: <prefix>/sbin/nginx
--conf-path=<nginx_install_path>    nginx config file path default: <prefix>/conf/nginx
--pid-path=<nginx_install_path>     nginx pid file path default: <prefix>/logs/nginx.pid
--error-path=<nginx_install_path>   error log default: <prefix>/logs/error.log
--http-path=<nginx_install_path>    access log default: <prefix>/logs/access.log
--user=<NAME>                       nginx worker uid name default: nobody
```

- add env at bottom of /etc/profile

```
vim /etc/profile
export PATH=$PATH:/usr/local/nginx/sbin
source /etc/profile
```

- start

```
# use spec conf to start nginx process
# nginx -c /dir/nginx.conf   : set configuration file (default: /etc/nginx/nginx.conf)
sudo ln -s /usr/local/nginx/sbin/nginx /usr/bin/nginx
sudo /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

#### nignx.conf配置文件实例

```
user nginx nginx;                               # 设置运行nginx的用户和用户组
worker_processes 4;                         # 设置nginx的worker进程数，不包括缓存的worker进程，设置为cpu核数，设置超过8，性能没多大变化反而会引起性能不稳定

# nginx默认没有开启利用多核CPU, 可以用worker_cpu_affinity 配置来充分利用多核CPU，以下是8核的配置例子。
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000 00100000 01000000 10000000;
pid /var/run/nginx.pid;                     # 存放nginx进程pid的文件
worker_rlimit_nofile 65535;                 # 设置nginx worker进程最大打开文件数目

# events块涉及的指令主要影响Nginx服务器与用户的网络连接
events {
    use epoll;                              # 选取哪种事件驱动模型处理连接请求
    worker_connections 65535;               # 每个worker可以同时支持的最大连接数等
    # multi_accept on;                      # 是否允许同时接收多个网络连接
}

# 代理、缓存和日志定义等绝大多数的功能和第三方模块的配置都可以放在http模块中
http {
    #### http模块全局配置
    sendfile on;                            # 启用高效传输文件功能
    tcp_nopush on;                          # 用于防止网络拥塞，tcp_nopush当sendfile on时才能用
    tcp_nodelay on;                         # 用于防止网络拥塞
    
    
    #### 缓存设置
    server_names_hash_bucket_size 64;       # 多个域名时设置存放域名的hash表大小为64kb
    large_client_header_buffers 4 64k;      # 设定请求缓存大小
    client_header_buffer_size 32k;          # 为请求头分配一个缓冲区，如超过这个值就用large_client_header_buffers分配更大的缓冲区，如还超过large_client_header_buffers大小返回414
    client_max_body_size 50m;               # 为请求体分配一个缓冲区
    
    open_file_cache max=204800 inactive=30s;  # 设置缓存数量，30s不访问则删除该缓存
    open_file_cache_valid 30s;              # 30s检查一次缓存修改时间等元数据是否有更新，有则更新，无则继续使用缓存
    open_file_cache_min_uses 6;             # inactive时间内文件最少使用次数，超过则使用缓存，没有则从缓存删除
    open_file_cache_errors on;              # 缓存在文件访问期间发生的错误
    
    
    #### 客户端连接设置，单位秒
    keepalive_timeout 65;                   # 客户端连接超时时间
    keepalive_requests 100000;              # 在tcp长连接上接收请求最大数为100000个，超过则关闭连接
    client_header_timeout 10;               # 客户端请求头的超时时间
    client_body_timeout 10;                 # 客户端请求主体超时时间
    types_hash_max_size 2048;               # 默认为1024kb，值越大消耗内存越大，但检索速度更快
    reset_timedout_connection on;           # 关闭不响应的客户端连接，释放客户端所占有的内存空间
    send_timeout 10;                        # 客户端响应超时时间，在两次客户端读取操作之间。如果在这段时间内，客户端没有读取任何数据，nginx就会关闭连接
    
    
    #### FastCGI配置，单位秒。可改善网站的性能，减少资源占用，提高访问速度。
    fastcgi_connect_timeout 300;
    fastcgi_send_timeout 300;
    fastcgi_read_timeout 300;
    fastcgi_buffer_size 64k;
    fastcgi_buffers 4 64k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;


    #### 日志文件格式配置
    log_format access '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" $http_x_forwarded_for '
        '"$request_time $upstream_response_time $pipe" '
        '"$gzip_ratio" "$http_cookie" "$http_accept" "$http_accept_encoding" "$http_platform"';
    
    # 以下为日志格式注释
    # 访问日志格式：log_format 格式名access 客户端ip 基本身份验证提供的用户名 服务器本地时间 完整原始请求，包括请求类型GET及请求uri及http版本
    # log_format access '$remote_addr - $remote_user [$time_local] "$request" '
    # 响应状态200 发送到客户端的响应体字节数 http跳转来源 
    # '$status $body_bytes_sent "$http_referer" '
    # 访问者操作系统类型及浏览器版本 如果客户端没有通过代理访问则不显示 
    # '"$http_user_agent" $http_x_forwarded_for '
    # 处理请求花费的总时间 upstream返回响应时间 p或. 
    # '"$request_time $upstream_response_time $pipe" '
    # 压缩率 cookie内容 接收资源类型 
    # '"$gzip_ratio" "$http_cookie" "$http_accept"
    # 浏览器支持的压缩编码列表 平台来源，是移动端还是pc端
    # "$http_accept_encoding" "$http_platform"';

    # access_log 路径/日志名 日志格式名 缓存大小；如没定义日志格式，则用默认的格式combined；
    access_log /var/log/nginx/access.out access buffer=32k;
    # 错误日志，默认作用于全局
    error_log /var/log/nginx/error.log;


    #### Gzip配置
    gzip on;                                # Nginx默认只对text/html进行压缩,启用压缩对其他类型文件压缩 
    # gzip_vary on;                         # 有的浏览器不支持压缩，在http头部加vary头决定是否需要压缩
    # gzip_proxied any;                     # any任何资源都压缩
    gzip_min_length 100;                    # 设置将被gzip压缩的的最小长度    
    gzip_buffers 4 16k;                     # 压缩缓冲区个数和大小
    gzip_comp_level 6;                      # gzip压缩比，1为最小，处理最快；9为压缩比最大，处理最慢，传输速度最快，也最消耗 CPU；
    gzip_http_version 1.1;                  # 压缩版本，默认1.1
    # 用正则表达式匹配命中的资源不进行压缩
    gzip_disable "xxx_Android\/\b([1-9]|9[0-9])\b";
    # 需要压缩的资源类型
    gzip_types text/plain application/x-javascript text/css text/htm application/xml application/json text/javascript text/xml;


    #### 代理设置，即nginx和后端服务器间的通讯设置
    proxy_hide_header X-Application-Context;# 隐藏服务器响应的某些头部信息，这里隐藏X-Application-Context
    proxy_connect_timeout 90;               # nginx跟后端服务器连接超时时间（代理连接超时）
    proxy_send_timeout 90;                  # 后端服务器数据回传时间（代理发送超时）
    proxy_read_timeout 90;                  # 连接成功后，后端服务器响应时间（代理接收超时）
    proxy_buffering on;                     # 开启缓存，缓存被代理服务器的响应内容，此参数开启后proxy_buffers和proxy_busy_buffers_size参数才会起作用
    proxy_buffer_size 4k;                   # 设置代理服务器（nginx）保存用户头信息的缓冲区大小
    proxy_buffers 4 32k;                    # proxy_buffers 缓冲区，网页平均在32k以下的设置
    proxy_busy_buffers_size 64k;            # 高负荷下缓存大小（proxy_buffers*2）
    proxy_max_temp_file_size 2048m;         # 默认1024m, 该指令用于设置当网页内容大于proxy_buffers时，临时文件大小的最大值。如果文件大于这个值，它将从upstream服务器同步地传递请求，而不是缓存到磁盘
    proxy_temp_file_write_size 96k;         # 当被代理服务器的响应过大时，nginx一次性写入临时文件的数据量。
    
    # proxy_temp_path和proxy_cache_path需要在同一个分区中，不一定同路径
    # 必须要先手动创建此目录，当上游服务器的响应过大不能存储到配置的缓冲区域时，Nginx存储临时文件硬盘路径
    proxy_temp_path /usr/local/nginx/temp_dir;
    # 创建代理缓存目录保存来自被代理服务器返回的数据
    # 格式：proxy_cache_path 缓存路径 在缓存路径下使用2级目录存放缓存资源 key_zone=名:大小，1m可存8000个key 内存中的缓存过期检查周期30m 最大缓存大小6G，没指定会耗掉所有磁盘空间
    proxy_cache_path /usr/local/nginx/cache levels=1:2 keys_zone=my_cache_one:512m inactive=30m max_size=6g;
    # proxy_cache_path定义之后就可以在需要缓存的地方引用了，同时也可加多几个其他的配置，如下缓存静态文件
    # location ~ .*\.(gif|jpg|png|css|js)(.*)  {
    #  proxy_cache my_cache_one;
    #  proxy_cache_revalidate on;
    #  proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    #  proxy_cache_lock on;
    #  proxy_pass http://my_upstream;
    #}
    
    #### 访问限制设置
    # limit_conn_zone 客户端ip zone=连接限制区内存名字:大小（定义了之后就可以在location使用连接限制区内存块来限制具体的uri连接个数了。如在location里limit_conn conn 10;表示每个客户端IP最多有10个连接。） 
    limit_conn_zone $binary_remote_addr zone=conn:20m;
    # limit_req_zone 客户端ip zone=请求限制区内存名字:大小 请求速率，1分钟30次（同上，如在location里limit_req zone=one burst=5;表示使用上面定义的请求限制区内存块定义的规则限制请求速率，即1分钟30次，最高突发5次每秒）
    limit_req_zone $binary_remote_addr zone=one:20m rate=30r/m;


    #### 引入其他配置文件
    include /usr/local/nginx/conf/mime.types;           # 引入所有元数据后缀名
    default_type application/octet-stream;              # 默认类型为二进制
    include /usr/local/nginx/conf/vhosts/*.conf;        # 引入自建vhosts目录下所有配置文件
    server_tokens off;
    # include /etc/nginx/sites-enabled/default;
}
```

#### [upstream支持的负载均衡算法](https://moonbingbing.gitbooks.io/openresty-best-practices/content/ngx/balancer.html)

```
upstream vickey.com {
    # 切流量各90%, 10%, 10%
    #ip_hash;
    server 10.x.x.x:port weight=6 max_fails=2 fail_timeout=30s;
    server 10.x.x.x:port weight=4 max_fails=2 fail_timeout=30s
}

server{
    location /api/xxx {
        # 上面http里配置的访问限制conn, 最多30个连接, 所有ip请求突发最高每秒5个请求
        limit_conn conn 30;
        limit_req zone=allips burst=5;
        # 转发到upstream定义的server
        proxy_pass http://vickey.com;
    }
    
    location /xxx {...}
}
```

- 轮询（默认）：每个请求按时间顺序逐一分配到不同的后端服务器，如果后端某台服务器宕机，故障系统被自动剔除，使用户访问不受影响。Weight 指定轮询权值，Weight 值越大，分配到的访问机率越高，主要用于后端每个服务器性能不均的情况下。
- ip_hash：每个请求按访问 IP 的 hash 结果分配，这样来自同一个 IP 的访客固定访问一个后端服务器，有效解决了动态网页存在的 session 共享问题。
- fair：这是比上面两个更加智能的负载均衡算法。此种算法可以依据页面大小和加载时间长短智能地进行负载均衡，也就是根据后端服务器的响应时间来分配请求，响应时间短的优先分配。Nginx 本身是不支持 fair 的，如果需要使用这种调度算法，必须下载 Nginx 的 upstream_fair 模块。
- url_hash：此方法按访问 url 的 hash 结果来分配请求，使每个 url 定向到同一个后端服务器，可以进一步提高后端缓存服务器的效率。Nginx 本身是不支持 url_hash 的，如果需要使用这种调度算法，必须安装 Nginx 的 hash 软件包。
- least_conn：最少连接负载均衡算法，简单来说就是每次选择的后端都是当前最少连接的一个 server(这个最少连接不是共享的，是每个 worker 都有自己的一个数组进行记录后端 server 的连接数)。
- hash：这个 hash 模块又支持两种模式 hash, 一种是普通的 hash, 另一种是一致性 hash(consistent)。

#### [封ip](https://amos-x.com/index.php/amos/archives/centos7-nginx-add-module/)

- 进入之前安装时的源码包
- 需要先的下载好第三方安装包，然后通过--add-moudle来指定目录
- 使用`./configure --add-module=module_name`重新编译
- 编译安装后原有的配置文件等都会保留，只会有nginx的执行文件发生改变, `nginx -s reload`重启nginx即可

#### nginx状态监控

配置路由`/etc/nginx/conf.d/default.conf`

```
location /nginxstatus{
    stub_status on;
    auth_basic "nginxstatus";
}
```
访问浏览器`http://{host}:{port}/nginxstatus`
```
Active connections: 2 
server accepts handled requests
 7 7 17 
Reading: 0 Writing: 1 Waiting: 1 
```

#### 参考文章
- http://nginx.org/en/docs/
- http://nginx.org/en/docs/varindex.html
- http://nginx.org/en/docs/http/ngx_http_core_module.html#var_http_
- https://juejin.im/post/5d81906c518825300a3ec7ca
- https://www.linuxidc.com/Linux/2013-07/86961.htm
- https://blog.csdn.net/lgq421033770/article/details/51787273
- http://tool.oschina.net/commons/
- https://www.cnblogs.com/EasonJim/p/7806879.html
- https://blog.csdn.net/angus_01/article/details/81912031
- https://www.jb51.net/article/136921.htm
- https://amos-x.com/index.php/amos/archives/centos7-nginx/
