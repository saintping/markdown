在上一篇中讲了如何搭建一个基本的WordPress网站。要想达到比较好的传播效果，这些肯定是还不够的，首先需要支持域名访问和HTTPS。

### 准备工作
在此之前，有一些准备工作需要先完成：购买域名，将域名解析到公网IP，向CA申请证书，网站ICP备案（主机部署在国内需要）等。

### Nginx反向代理
我这里采用Nginx做反向代理，来支持HTTPS以及后续搜索引擎SEO需要的rewrite。
```bash
yum install -y gcc-c++ pcre-devel zlib-devel openssl-devel
wget https://nginx.org/download/nginx-1.24.0.tar.gz
configure --with-http_ssl_module; make; make install
```

将Apache的监听端口改成8080（因为Nginx要用80、443对外提供服务），并且将网站反向代理到Nginx上。
```bash
/usr/local/nginx/conf/nginx.conf
    server {
        listen 443 ssl;
        server_name ping666.com;
        ssl_certificate /root/cert/ping666.com.pem;
        ssl_certificate_key /root/cert/ping666.com.key;
        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header HOST $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
    server {
        listen 80;
        server_name ping666.com;
        rewrite ^(.*)$ https://$host$1 permanent;
    }
```

/root/cert/是证书存放路径，ping666.com是我的域名。

将Nginx配置成系统服务/etc/systemd/system/nginx.service，方便开机启动。
```bash
systemctl daemon-reload
systemctl start nginx
systemctl enable nginx
```

在云控制台上打开80、443的外网访问，HTTPS现在可以正常访问了https://ping666.com
http的流量也会跳转到https上来。

### 全站切到HTTPS
迎面扑来一个大问题，HTTPS加载到的页面是杂乱无章的，显然是JS、CSS等文件没加载到。
有两种方法解决这个问题，是不是用插件看个人喜欢：

1. 通过插件解决
   really-simple-ssl、better-search-replace两个插件一顿操作就可以了。我使用的是下面第2种方案。
2. 通过修改代码和配置解决
   - 第一步：将网站链接设置成HTTPS
     【设置|常规】https://ping666.com
   - 第二步：文章内的超链接也需要全部替换为HTTPS
     通过插件better-search-replace
   - 第三步：将JS、CSS全部改为HTTPS输出
```bash
# 在文件data/wordpress/wp-includes/functions.php的require ABSPATH这一行后面加上以下几行
add_filter('script_loader_src', 'agnostic_script_loader_src', 20, 2);
function agnostic_script_loader_src($src, $handle) {
    return preg_replace('/^(http|https):/', '', $src);
}
add_filter('style_loader_src', 'agnostic_style_loader_src', 20, 2);
function agnostic_style_loader_src($src, $handle) {
    return preg_replace('/^(http|https):/', '', $src);
}
# 在data/wordpress/wp-config.php文件开始加上这几个配置
$_SERVER['HTTPS'] = 'on';
define('FORCE_SSL_LOGIN', true);
define('FORCE_SSL_ADMIN', true);
```

通过以上操作后，网站就完全支持HTTPS访问了。
但是性能嘛，请看下一篇。
