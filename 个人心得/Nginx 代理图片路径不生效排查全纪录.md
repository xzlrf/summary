# Nginx 代理图片路径不生效排查全纪录

## 1. 功能目标

在前端页面中正确展示后端报警系统产生的图片。图片存储在另一台图片服务器（192.168.110.231）上，需要通过前端服务器（192.168.110.141）的 Nginx 进行反向代理，实现图片的统一访问。

## 2. 问题背景

- **架构**：前端 Vue 项目部署在 141 服务器（Nginx 8081端口），图片存储在 231 服务器（Nginx 8080端口）。
- **初始状态**：后端接口返回的图片字段是服务器物理磁盘路径，例如：`E:\smart_engine_no_wvp\app\alerts\images\alert_1_101_1779344651.jpg`。
- **期望状态**：前端通过 HTTP 路径 `http://localhost:8081/images/alert_1_101_1779344651.jpg` 即可访问到 231 服务器上的对应图片。

## 3. 踩坑与排查过程

### 坑一：前端强行处理物理路径，导致架构混乱

**现象**：最初后端直接把数据库存储的图片原始路径 `E:\smart_engine_no_wvp\app\alerts\images\alert_1_101_1779344651.jpg` 返给前端，前端试图在前端代码里做字符串替换拼装成 URL。
**分析**：这属于架构层面的反模式。暴露服务器物理路径不安全，且前端需要写死大量恶心的字符串处理逻辑，一旦后端目录变更，前端同步崩溃。
**解决**：**在后端接口层直接转换路径**。后端将 `E:\...\images\xxx.jpg` 转换为 `/images/xxx.jpg` 返给前端。前端只需直接拼接域名即可，保持代码纯净。

### 坑二：Nginx 静态资源代理配置细节（alias vs root，斜杠问题）

**现象**：后端路径转换后，浏览器请求 `http://192.168.110.231:8080/images/xxx.jpg` 依然 404。
**分析**：

1. Windows 下 Nginx 识别盘符路径必须使用正斜杠 `/`，反斜杠 `\` 易出错。
2. 使用 `root` 指令会导致路径拼接错误（如 `root E:/app/alerts/;` 会拼成 `E:/app/alerts/images/xxx.jpg`），与实际物理路径不符。
   **解决**：在 231 图片服务器的 Nginx 中使用 `alias` 并确保斜杠闭合：

```
location /images/ {
    alias "E:/smart_engine_no_wvp/app/alerts/images/";
    autoindex off;
}
```

此时在 141 服务器上使用 `curl -I http://192.168.110.231:8080/images/xxx.jpg` 测试，返回 `Content-Type: image/jpeg`，证明图片服务器本身没问题，网络也是通的。

### 坑三：请求被 `location /` 劫持，代理配置形同虚设

**现象**：在浏览器访问 `http://localhost:8081/images/xxx.jpg`，返回的状态码是 200，但 `Content-Type` 竟然是 `text/html`，内容是 Vue 的首页代码。
**分析**：请求根本没有走 141 服务器的 `location /images/` 代理，而是被 `location /` 里的 `try_files $uri$uri/ /index.html;` 兜底，重写到了前端首页。常见原因有两个：

1. `location /images/` 内部混用了 `root` 或 `try_files`，导致 Nginx 优先查找本地文件。
2. 前端打包的 `dist` 目录下刚好也存在一个同名的 `images` 文件夹及图片，Nginx 直接返回了本地旧文件，没有去请求 231 代理。
   **解决**：
3. 保持 `location /images/` 极度纯净，**只放 `proxy_pass` 相关指令，绝不放 `root` 或 `try_files`**。
4. 为了验证请求到底走没走到这个 location，加入调试响应头：

```
location /images/ {
    proxy_pass http://192.168.110.231:8080/images/ ;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    # 调试头：如果响应里有这个头，说明请求确实走到了这里
    add_header X-Debug-Proxy "images"; 
}
```

### 坑四：幽灵 Nginx 进程（最终 Boss，导致配置永远不生效）

**现象**：配置文件明明已经改对了，也执行了 `nginx -s reload`，但浏览器死活不返回 `X-Debug-Proxy` 头，依然返回 `text/html`。
**分析**：执行 `tasklist | findstr nginx` 发现竟然有 **4 个 nginx.exe 进程**！这说明机器上运行了多个 Nginx 实例，或者实例被异常重复拉起。当我们修改了 A 目录的配置并执行 reload 时，实际上重启的是 B 目录的 Nginx，而真正监听 8081 端口的 C 目录的 Nginx 根本没读到新配置。
**解决**：

1. **寻找真凶**：通过 `netstat -ano | findstr :8081` 找到真正占用 8081 端口的进程 PID（例如 15828）。*(注意：忽略 CLOSE_WAIT 等状态的连接，只看 LISTENING 状态的那一行)*。
2. **顺藤摸瓜**：执行 `wmic process where "ProcessId=15828" get ExecutablePath` 找到这个真实进程的启动路径。
3. **赶尽杀绝**：执行 `taskkill /f /im nginx.exe` 将所有 Nginx 进程全部杀掉。
4. **重新启航**：`cd /d` 进入刚才查到的真实 Nginx 目录，执行 `start nginx` 重新启动。
5. **大功告成**：再次访问图片，响应头终于出现了 `X-Debug-Proxy: images` 和 `Content-Type: image/jpeg`，图片完美显示！

## 4. 最终正确配置参考

### 图片服务器 (192.168.110.231:8080) Nginx 配置

```
server {
    listen 8080;
    server_name 192.168.110.231;

    location /images/ {
        # 使用 alias 映射物理路径，注意末尾斜杠
        alias "E:/smart_engine_no_wvp/app/alerts/images/";
        autoindex off;
    }
}
```

### 前端服务器 (192.168.110.141:8081) Nginx 配置

```
server {
    listen       8081;
    server_name  localhost;

    # 1. 前端页面
    location / {
        root   D:\work\AI_video_analyssis_vue\ai_video_analysis_ui\dist;
        index  index.html index.htm;
		try_files $uri$uri/ /index.html;
        
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
    }

    # 2. 图片服务器反向代理 (极其纯净，不掺杂 root/try_files)
    location /images/ {
        proxy_pass http://192.168.110.231:8080/images/ ;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # ... 其他代理配置 ...
}
```

## 5. 经验总结

1. **前后端职责分离**：物理路径转 HTTP 路径的工作必须由后端在接口层完成，前端只负责消费 URL。
2. **Nginx 代理要纯粹**：作为纯粹反向代理的 `location`，里面不要放 `root`、`alias` 或 `try_files`，否则容易引发本地文件拦截。
3. **善用调试头**：在 Nginx 中使用 `add_header X-Debug-xxx "xxx";` 是判断请求到底命中了哪个 `location` 块的最快方法。
4. **重载 Nginx 需谨慎**：Windows 环境下极易产生多个 Nginx 进程。**每次操作前，务必先 `cd /d` 进入真实的 Nginx 安装目录**，再执行 `nginx -s reload`。若遇配置不生效灵异事件，先杀进程（`taskkill /f /im nginx.exe`）再重启，绝对能解决 90% 的玄学问题。
5. **涉及浏览器的东西一定记得及时清理浏览器缓存！！！！**