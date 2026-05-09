# Java 开发环境搭建清单

> 适用于 Windows 11 的 Java 全栈开发环境配置指南

---

## 一、JDK（多版本管理）

### 推荐方案：使用 SDKMAN (WSL) 或手动安装多版本

#### 方式 A：手动安装（推荐 Windows）

| 版本 | 下载地址 | 安装路径 | 用途 |
|------|----------|----------|------|
| JDK 8 | https://adoptium.net/temurin/releases/?version=8 | `C:\Java\jdk-8` | 老项目维护、兼容性 |
| JDK 17 | https://adoptium.net/temurin/releases/?version=17 | `C:\Java\jdk-17` | Spring Boot 3.x 主力开发 |
| JDK 21 | https://adoptium.net/temurin/releases/?version=21 | `C:\Java\jdk-21` | 新项目、虚拟线程等新特性 |

#### 环境变量配置

1. 新建系统变量 `JAVA_HOME`，指向当前默认版本（如 `C:\Java\jdk-17`）
2. `PATH` 中添加 `%JAVA_HOME%\bin`
3. 切换版本时只需修改 `JAVA_HOME` 即可

#### 快速切换脚本

创建 `jdk8.bat`、`jdk17.bat`、`jdk21.bat` 放在 PATH 目录下：

```bat
@echo off
setx JAVA_HOME "C:\Java\jdk-8" /M
set JAVA_HOME=C:\Java\jdk-8
java -version
```

---

## 二、IDE 工具

### IntelliJ IDEA

- **版本**：Ultimate（推荐）或 Community
- **下载**：https://www.jetbrains.com/idea/download/
- **必装插件**：
  - Lombok
  - MyBatisX
  - Maven Helper
  - RestfulTool / RestfulToolkit
  - Rainbow Brackets
  - Translation（翻译）
  - GitToolBox
  - String Manipulation
  - .env files support
- **配置要点**：
  - 设置 JDK 默认版本
  - 配置 Maven 路径和 settings.xml
  - 调整字体和编码（UTF-8）
  - 开启自动保存（默认已开）

### VS Code

- **下载**：https://code.visualstudio.com/
- **Java 相关插件**：
  - Extension Pack for Java（微软官方）
  - Spring Boot Extension Pack
  - Maven for Java
  - Debugger for Java
- **其他推荐插件**：
  - GitLens
  - Prettier
  - Error Lens
  - Auto Rename Tag

### Claude Code

- **安装**：`npm install -g @anthropic-ai/claude-code`
- **登录**：运行 `claude` 首次会自动跳转认证
- **常用命令**：
  ```
  /help      - 查看帮助
  /clear     - 清空上下文
  /compact   - 压缩上下文
  ```
- **配置**：`~/.claude/CLAUDE.md` 可设置项目级指令

---

## 三、数据库

### MySQL

- **版本**：8.0+（推荐 8.0.39+）
- **安装**：https://dev.mysql.com/downloads/mysql/
- **配置要点**：
  - 字符集设为 `utf8mb4`
  - 端口默认 3306
  - 设置 root 密码
  - 加入环境变量 PATH
- **服务管理**：
  ```powershell
  net start MySQL80   # 启动
  net stop MySQL80    # 停止
  ```

### Navicat

- **版本**：Navicat Premium 或 Navicat for MySQL
- **用途**：数据库图形化管理、数据导入导出、SQL 编辑
- **配置**：新建连接，测试连通性

### Redis

- **Windows 安装**：
  - 方式 A：使用 Memurai（Windows 原生 Redis 替代品）
  - 方式 B：WSL2 中安装原生 Redis
  - 方式 C：Docker 运行 `docker run -d -p 6379:6379 redis:latest`
- **可视化工具**：Redis Desktop Manager / Another Redis Desktop Manager
- **验证**：`redis-cli ping` → `PONG`

---

## 四、版本控制 & 构建

### Git

- **下载**：https://git-scm.com/download/win
- **安装选项**：
  - 使用默认编辑器选 VS Code 或 Notepad++
  - 默认分支名选 `main`
- **基础配置**：
  ```bash
  git config --global user.name "Your Name"
  git config --global user.email "your@email.com"
  git config --global init.defaultBranch main
  git config --global core.autocrlf true
  git config --global core.editor "code --wait"
  ```

### Maven

- **下载**：https://maven.apache.org/download.cgi
- **安装路径**：`C:\Maven\apache-maven-x.x.x`
- **环境变量**：
  - 新建 `MAVEN_HOME`
  - `PATH` 添加 `%MAVEN_HOME%\bin`
- **settings.xml 配置要点**（`~/.m2/settings.xml`）：
  - 本地仓库路径：`<localRepository>D:\.m2\repository</localRepository>`
  - 配置国内镜像源（阿里云）：
    ```xml
    <mirror>
      <id>aliyun</id>
      <mirrorOf>central</mirrorOf>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
    ```
  - 配置默认 JDK 编译版本

---

## 五、运行时环境

### Node.js & npm

- **安装方式**：
  - 直接安装：https://nodejs.org/ （推荐 LTS）
  - 或使用 nvm-windows 管理多版本：https://github.com/coreybutler/nvm-windows
- **切换 npm 镜像**：
  ```bash
  npm config set registry https://registry.npmmirror.com
  ```
- **验证**：`node -v` && `npm -v`

### Python 3

- **版本**：3.12+ 或 3.13
- **下载**：https://www.python.org/downloads/
- **安装注意**：**勾选 "Add Python to PATH"**
- **验证**：`python --version`
- **常用包**：
  ```bash
  pip install requests
  pip install --upgrade pip
  ```

---

## 六、浏览器 & 效率工具

### Chrome

- **下载**：https://www.google.com/chrome/
- **开发者必装扩展**：
  - JSON Viewer / JSON Formatter
  - Postman Interceptor（如需）
  - React Developer Tools（前端需要）
  - Wappalyzer

### Everything

- **下载**：https://www.voidtools.com/
- **用途**：全盘文件秒搜，效率神器
- **技巧**：`Win + S` 设置快捷键快速调出

### Notepad++

- **下载**：https://notepad-plus-plus.org/
- **用途**：快速查看/编辑日志、配置文件
- **设置**：
  - 默认编码 UTF-8
  - 显示行号、状态栏
  - 关联常见配置文件类型

---

## 七、安装顺序建议

```
1. JDK（8/17/21）
2. Maven
3. Git
4. IntelliJ IDEA
5. MySQL
6. Redis
7. Node.js + npm
8. Python 3
9. Chrome
10. VS Code + 插件
11. Claude Code
12. Navicat
13. Everything
14. Notepad++
```

---

## 八、验证清单

安装完成后逐一检查：

| 工具 | 验证命令 | 预期 |
|------|----------|------|
| JDK 8 | `java -version` | 1.8.x |
| JDK 17 | 切换后 `java -version` | 17.x |
| Maven | `mvn -v` | Apache Maven 3.x |
| Git | `git --version` | git 2.x |
| MySQL | `mysql -u root -p` | 连接成功 |
| Redis | `redis-cli ping` | PONG |
| Node.js | `node -v` | v20+ 或 v22+ |
| npm | `npm -v` | 10+ |
| Python | `python --version` | 3.12+ |
| Claude Code | `claude --version` | 正常输出版本号 |
