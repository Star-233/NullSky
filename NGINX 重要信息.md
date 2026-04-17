## 常见的重要文件
- `/etc/nginx/nginx.conf`
	- 主配置文件
- `/var/log/nginx/` 
	- `access.log`
		- 记录了**每一个**进入服务器的 HTTP 请求。
	- `error.log`
		- 记录了 NGINX 运行中的错误信息、连接失败、配置语法错误等。
- `/etc/nginx/conf.d/`
	- 这是一个目录，通常包含以 `.conf` 结尾的具体网站配置。