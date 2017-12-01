---
title: nginx编译
date: 2017-12-01 19:31:39
tags:
 - nginx
---

# 编译
从<http://nginx.org/download/>下载nginx,最简单的用法

``` bash
wget http://nginx.org/download/nginx-1.9.9.tar.gz
tar zvfx nginx-1.9.9.tar.gz
cd nginx-1.9.9.tar.gz
./configure
make
./obj/nginx
```

考虑到nginx有各种配置和各种可选的模块，通常需要定制configure的参数，configure --help查看所有的选项

可以【**nginx -V**】查看现有nginx的编译参数，若configure过程中缺少某些库，自行安装(ubuntu apt-get即可)

例如
``` bash
./configure  --with-cc-opt='-g -O2 -fPIE -fstack-protector-strong \
    -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2' \
    --with-ld-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro \
    -Wl,-z,now' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf \
    --http-log-path=/var/log/nginx/access.log \
    --error-log-path=/var/log/nginx/error.log \
    --lock-path=/var/lock/nginx.lock \
    --pid-path=/run/nginx.pid \
    --http-client-body-temp-path=/var/lib/nginx/body \
    --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
    --http-proxy-temp-path=/var/lib/nginx/proxy \
    --http-scgi-temp-path=/var/lib/nginx/scgi \
    --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
    --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module \
    --with-http_stub_status_module --with-http_realip_module \
    --with-http_auth_request_module --with-http_addition_module \
    --with-http_gunzip_module --with-http_gzip_static_module \
    --with-http_v2_module --with-http_sub_module --with-stream \
    --with-stream_ssl_module --with-threads
```

make之后，sudo ./objs/nginx 即可运行（background）模式

关闭nginx

``` bash
#从容停止Nginx 
sudo kill -QUIT `pidof nginx | sed "s/ /\n/g" | sort |  head -n 1`
#快速停止Nginx  
sudo kill -TERM `pidof nginx | sed "s/ /\n/g" | sort |  head -n 1`
#强制停止Nginx
sudo kill -9 `pidof nginx | sed "s/ /\n/g" | sort |  head -n 1`
```

前台运行
``` bash
nginx -g 'daemon off;'
```

# 编译第三方插件
随便挑一个<https://www.nginx.com/resources/wiki/modules/fancy_index/>

``` bash
# 添加--add-module参数
./configure ** --add-module=/tmp/ngx-fancyindex
git clone https://github.com/aperezdc/ngx-fancyindex.git ngx-fancyindex
```

``` nginx
server {                                                                        
	listen 8800;
	root  /home/king/;
	fancyindex on;
	fancyindex_exact_size off;
	fancyindex_localtime on;
}
```

# OpenResty

安装 <https://openresty.org/cn/installation.html>

可以安装预编译的二进制或者直接从源码编译

> OpenResty以nginx的方式发布，只不过和官方的使用不同的模块。由于两者的配置等各种依赖隔离，只要确保OpenResty和Nginx不端口冲突，就可和谐共生

查看sudo make install 可以看到默认的安装路径
``` bash
test -f '/usr/local/openresty/nginx/conf/mime.types' \
	|| cp conf/mime.types '/usr/local/openresty/nginx/conf'
cp conf/mime.types '/usr/local/openresty/nginx/conf/mime.types.default'
test -f '/usr/local/openresty/nginx/conf/fastcgi_params' \
	|| cp conf/fastcgi_params '/usr/local/openresty/nginx/conf'
cp conf/fastcgi_params \
	'/usr/local/openresty/nginx/conf/fastcgi_params.default'
test -f '/usr/local/openresty/nginx/conf/fastcgi.conf' \
	|| cp conf/fastcgi.conf '/usr/local/openresty/nginx/conf'
cp conf/fastcgi.conf '/usr/local/openresty/nginx/conf/fastcgi.conf.default'
test -f '/usr/local/openresty/nginx/conf/uwsgi_params' \
	|| cp conf/uwsgi_params '/usr/local/openresty/nginx/conf'
cp conf/uwsgi_params \
	'/usr/local/openresty/nginx/conf/uwsgi_params.default'
test -f '/usr/local/openresty/nginx/conf/scgi_params' \
	|| cp conf/scgi_params '/usr/local/openresty/nginx/conf'
cp conf/scgi_params \
	'/usr/local/openresty/nginx/conf/scgi_params.default'
# 主配置
test -f '/usr/local/openresty/nginx/conf/nginx.conf' \
	|| cp conf/nginx.conf '/usr/local/openresty/nginx/conf/nginx.conf'
cp conf/nginx.conf '/usr/local/openresty/nginx/conf/nginx.conf.default'
test -d '/usr/local/openresty/nginx/logs' \
	|| mkdir -p '/usr/local/openresty/nginx/logs'
test -d '/usr/local/openresty/nginx/logs' \
	|| mkdir -p '/usr/local/openresty/nginx/logs'
test -d '/usr/local/openresty/nginx/html' \
	|| cp -R docs/html '/usr/local/openresty/nginx'
test -d '/usr/local/openresty/nginx/logs' \
# 日志
	|| mkdir -p '/usr/local/openresty/nginx/logs'
make[2]: Leaving directory '/tmp/openresty-1.13.6.1/build/nginx-1.13.6'
make[1]: Leaving directory '/tmp/openresty-1.13.6.1/build/nginx-1.13.6'
# 二进制
ln -sf /usr/local/openresty/nginx/sbin/nginx /usr/local/openresty/bin/openresty
```

## [网站统计中的数据收集原理及实现](http://blog.codinglabs.org/articles/how-web-analytics-data-collection-system-work.html)

配置

``` nginx
http {
	# 创建名称为tick的日志格式（后面会引用）
	log_format tick "$msec^A$remote_addr^A$u_domain^A$u_url^A$u_title^A$u_referrer^A$u_sh^A$u_sw^A$u_cd^A$u_lang^A$http_user_agent^A$u_utrace^A$u_account";

	server {
		listen 9088;
		index index.html;

        location / {
            root   html;
            index  index.html index.htm;
        }

		location /1.gif {
			#伪装成gif文件
			default_type image/gif;    
			#本身关闭access_log，通过subrequest记录log
			access_log off;

			access_by_lua "
				-- 用户跟踪cookie名为__utrace
				local uid = ngx.var.cookie___utrace        
				if not uid then
					-- 如果没有则生成一个跟踪cookie，算法为md5(时间戳+IP+客户端信息)
					uid = ngx.md5(ngx.now() .. ngx.var.remote_addr .. ngx.var.http_user_agent)
				end 
				ngx.header['Set-Cookie'] = {'__utrace=' .. uid .. '; path=/'}
				if ngx.var.arg_domain then
					-- 通过subrequest到/i-log记录日志，将参数和用户跟踪cookie带过去
					ngx.location.capture('/i-log?' .. ngx.var.args .. '&utrace=' .. uid)
				end 
			";  

			#此请求不缓存
			add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";
			add_header Pragma "no-cache";
			add_header Cache-Control "no-cache, max-age=0, must-revalidate";
			#返回一个1×1的空gif图片
			empty_gif;
		}   

		location /i-log {
			internal;

			set_unescape_uri $u_domain $arg_domain;
			set_unescape_uri $u_url $arg_url;
			set_unescape_uri $u_title $arg_title;
			set_unescape_uri $u_referrer $arg_referrer;
			set_unescape_uri $u_sh $arg_sh;
			set_unescape_uri $u_sw $arg_sw;
			set_unescape_uri $u_cd $arg_cd;
			set_unescape_uri $u_lang $arg_lang;
			set_unescape_uri $u_utrace $arg_utrace;
			set_unescape_uri $u_account $arg_account;

			log_subrequest on;
			#记录日志到ma.log，实际应用中最好加buffer，格式为tick
			access_log /tmp/ma.log tick;

			echo '';
		}
	}
}
```

前端代码
cat /usr/local/openresty/nginx/html/ma.js
``` js
(function () {
    var params = {};
    //Document对象数据
    if(document) {
        params.domain = document.domain || ''; 
        params.url = document.URL || ''; 
        params.title = document.title || ''; 
        params.referrer = document.referrer || ''; 
    }   
    //Window对象数据
    if(window && window.screen) {
        params.sh = window.screen.height || 0;
        params.sw = window.screen.width || 0;
        params.cd = window.screen.colorDepth || 0;
    }   
    //navigator对象数据
    if(navigator) {
        params.lang = navigator.language || ''; 
    }   
    //解析_maq配置
    if(_maq) {
        for(var i in _maq) {
            switch(_maq[i][0]) {
                case '_setAccount':
                    params.account = _maq[i][1];
                    break;
                default:
                    break;
            }   
        }   
    }   
    //拼接参数串
    var args = ''; 
    for(var i in params) {
        if(args != '') {
            args += '&';
        }   
        args += i + '=' + encodeURIComponent(params[i]);
    }   
 
    //通过Image对象请求后端脚本
    var img = new Image(1, 1); 
    img.src = '/1.gif?' + args;
})();
```

cat /usr/local/openresty/nginx/html/index.html 
``` html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to OpenResty!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<script type="text/javascript">
var _maq = _maq || [];
_maq.push(['_setAccount', '网站标识']);
 
(function() {
    var ma = document.createElement('script'); ma.type = 'text/javascript'; ma.async = true;
    ma.src = '/ma.js';
    var s = document.getElementsByTagName('script')[0]; s.parentNode.insertBefore(ma, s);
})();
</script>
<h1>Welcome to OpenResty!</h1>
<p>If you see this page, the OpenResty web platform is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="https://openresty.org/">openresty.org</a>.<br/></p>

<p><em>Thank you for flying OpenResty.</em></p>
</body>
</html>
```

# 参考
1. <http://blog.codinglabs.org/articles/how-web-analytics-data-collection-system-work.html>
