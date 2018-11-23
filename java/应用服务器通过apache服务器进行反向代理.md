# 应用服务器通过apache服务器进行反向代理


### 配置：

```

重写web请求
RewriteEngine On
RewriteCond %{HTTP:Connection} Upgrade [NC]
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteRule /(.*) ws://[tagetIp]:18888/$1 [P,L]
```

#### 注意！以上的写法需要将 (2.4版本之后)

```
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
```
#### 模块放开，否则报500 错误

遇到端口号

```

httpd.conf 中：
Listen 8888

```

```
<VirtualHost *:8888>
	#   General setup for the virtual host
	<IfModule dir_module>
    	DirectoryIndex index.html
	</IfModule>
	DocumentRoot "D:/iVMS_Platform/bin/apps/apache/htdocs"
	ServerName pc-ivms
	ServerAdmin admin@example.com
	#ErrorLog "logs/error.log"
	ErrorLog "|bin/rotatelogs.exe -l logs/error-%Y-%m-%d.log 20M"
	#ErrorLog "|bin/rotatelogs.exe -l logs/error-%Y-%m-%d.log 86400"

	Include conf/proxy.conf - > 代理的具体配置
</VirtualHost>

```

### proxy.conf 中：

```

RewriteEngine On
RewriteCond %{HTTP:Connection} Upgrade [NC]
RewriteCond %{HTTP:Upgrade} websocket [NC]
RewriteRule /(.*) ws://127.0.0.1:18888/$1 [P,L]

ProxyPass "/"  "http://10.11.165.101:8000/"

```

