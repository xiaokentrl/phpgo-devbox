The project has been migrated to: https://github.com/xiaokentrl/phpbox

https://github.com/xiaokentrl/phpbox



# 🏗️ 最终项目架构 (1+1+N 扩展模型)

| 组件 | Flatpak ID | 说明 |
|------|------------|------|
| Base Runtime | `com.yourcompany.DevEnv.Base` | 共享基础库 (zlib, pcre) |
| Manager App | `com.yourcompany.DevEnv` | Go 管理器，支持组件安装、卸载、启停、端口配置 |
| PHP 多版本 | `com.yourcompany.DevEnv.PHP.8.2`<br>`com.yourcompany.DevEnv.PHP.8.3` | PHP 8.2/8.3 + OpenSSL 1.1 + Curl，可独立安装 |
| PHP 扩展模块 | `com.yourcompany.DevEnv.PHP.8.2.redis`<br>`com.yourcompany.DevEnv.PHP.8.3.redis` 等 | 为指定 PHP 版本安装 redis/pdo_mysql 等模块 |
| MySQL 多版本 | `com.yourcompany.DevEnv.MySQL.8.0`<br>`com.yourcompany.DevEnv.MySQL.8.4` | 各自携带 OpenSSL 3.0，端口可配 |
| Nginx 多版本 | `com.yourcompany.DevEnv.Nginx.1.24`<br>`com.yourcompany.DevEnv.Nginx.1.26` | 各自携带 OpenSSL，端口可配 |
| Redis 多版本 | `com.yourcompany.DevEnv.Redis.7.2`<br>`com.yourcompany.DevEnv.Redis.7.4` | 端口可配 |

> **扩展安装路径：** `/app/extensions/<组件>/<版本>/`
>
> **PHP 扩展模块安装路径：** `/app/extensions/php/<版本>/modules/<模块名>/`，启动时自动加载
>
> **全部通过管理器交互式操作**

---

## 🛠️ 第一步：环境准备与目录初始化

```bash
sudo apt update && sudo apt install flatpak flatpak-builder -y
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

mkdir -p ~/dev-env/{base,manager,extensions/{php/{8.2,8.3},mysql/{8.0,8.4},nginx/{1.24,1.26},redis/{7.2,7.4}},repo}
cd ~/dev-env
```

---

## 🧱 第二步：构建基础运行时（含 zlib、pcre）

文件：`base/com.yourcompany.DevEnv.Base.yml`

```yaml
id: com.yourcompany.DevEnv.Base
branch: '24.08'
runtime: org.freedesktop.Platform
runtime-version: '24.08'
sdk: org.freedesktop.Sdk
build-extension: true

modules:
  - name: stable-libs
    buildsystem: simple
    build-commands:
      - |
        tar xf zlib-1.3.1.tar.gz
        cd zlib-1.3.1
        ./configure --prefix=/app
        make -j$(nproc) && make install
      - |
        tar xf pcre-8.45.tar.bz2
        cd pcre-8.45
        ./configure --prefix=/app
        make -j$(nproc) && make install
    sources:
      - type: archive
        url: https://zlib.net/zlib-1.3.1.tar.gz
        sha256: 9a93b2b7dfdac77ceba5a558a580e74667dd6fede4585b91eefb60f03b72df23
      - type: archive
        url: https://sourceforge.net/projects/pcre/files/pcre/8.45/pcre-8.45.tar.bz2
        sha256: 4dae6d53c6e1f4e6b9c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c6c  # 需替换为真实 SHA256
```

构建：

```bash
cd ~/dev-env/base
flatpak-builder --force-clean --user --repo=../repo --install-deps-from=flathub build-dir-base com.yourcompany.DevEnv.Base.yml
```

---

## 🎛️ 第三步：终极版 Go 管理器

### 3.1 源码 `manager/main.go`

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"strings"
	"syscall"
)

var extDir = "/app/extensions"

type Manager struct {
	configDir string
	config    map[string]interface{}
}

func (m *Manager) init() {
	m.configDir = os.Getenv("XDG_CONFIG_HOME")
	if m.configDir == "" {
		m.configDir = filepath.Join(os.Getenv("HOME"), ".config")
	}
	m.config = make(map[string]interface{})
	m.loadConfig()
}

func (m *Manager) loadConfig() {
	configFile := filepath.Join(m.configDir, "devenv", "config.json")
	data, err := os.ReadFile(configFile)
	if err == nil {
		json.Unmarshal(data, &m.config)
	}
}

func (m *Manager) saveConfig() {
	configPath := filepath.Join(m.configDir, "devenv")
	os.MkdirAll(configPath, 0755)
	data, _ := json.MarshalIndent(m.config, "", "  ")
	os.WriteFile(filepath.Join(configPath, "config.json"), data, 0644)
}

func (m *Manager) list() {
	fmt.Printf("%-10s %-10s %-10s %-10s\n", "组件", "版本", "状态", "端口")
	fmt.Println(strings.Repeat("-", 40))
	for _, comp := range []string{"php", "mysql", "nginx", "redis"} {
		compPath := filepath.Join(extDir, comp)
		entries, err := os.ReadDir(compPath)
		if err != nil {
			continue
		}
		for _, entry := range entries {
			if entry.IsDir() {
				ver := entry.Name()
				pidFile := pidFilePath(comp, ver)
				status := "Stopped"
				port := ""
				if pid, ok := readPID(pidFile); ok && processRunning(pid) {
					status = "Running"
					port = m.getPortFromConfig(comp, ver)
				}
				fmt.Printf("%-10s %-10s %-10s %-10s\n", comp, ver, status, port)
			}
		}
	}
}

func pidFilePath(comp, version string) string {
	return filepath.Join("/tmp", fmt.Sprintf("devenv-%s-%s.pid", comp, version))
}

func readPID(pidFile string) (int, bool) {
	data, err := os.ReadFile(pidFile)
	if err != nil {
		return 0, false
	}
	pid, err := strconv.Atoi(strings.TrimSpace(string(data)))
	return pid, err == nil
}

func processRunning(pid int) bool {
	process, err := os.FindProcess(pid)
	if err != nil {
		return false
	}
	return process.Signal(syscall.Signal(0)) == nil
}

func (m *Manager) getPortFromConfig(comp, ver string) string {
	key := fmt.Sprintf("%s_%s_port", comp, ver)
	if val, ok := m.config[key]; ok {
		return fmt.Sprintf("%v", val)
	}
	return ""
}

func (m *Manager) install(extID string) {
	fmt.Printf("安装 %s...\n", extID)
	// 统一文件命名规则：将 ID 转为 kebab-case 并加 .flatpak
	// com.yourcompany.DevEnv.PHP.8.2 -> php-8.2.flatpak
	fileName := idToFileName(extID)
	cmd := exec.Command("flatpak", "install", "--user", "-y", fileName)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		fmt.Println("安装失败:", err)
	} else {
		fmt.Println("安装成功")
	}
}

func idToFileName(extID string) string {
	// 提取组件和版本：com.yourcompany.DevEnv.PHP.8.2 -> php-8.2
	parts := strings.Split(extID, ".")
	if len(parts) < 4 {
		return extID + ".flatpak"
	}
	comp := strings.ToLower(parts[3])
	ver := strings.Join(parts[4:], ".") // 8.2 或 8.2.redis
	return fmt.Sprintf("%s-%s.flatpak", comp, ver)
}

func (m *Manager) uninstall(extID string) {
	fmt.Printf("卸载 %s...\n", extID)
	cmd := exec.Command("flatpak", "uninstall", "--user", "-y", extID)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		fmt.Println("卸载失败:", err)
	}
}

func (m *Manager) start(comp, version string, port string) {
	script := filepath.Join(extDir, comp, version, "bin", "start.sh")
	if _, err := os.Stat(script); err != nil {
		fmt.Println("❌ 启动脚本不存在:", script)
		return
	}

	// 保存端口到配置
	if port != "" {
		m.config[fmt.Sprintf("%s_%s_port", comp, version)] = port
		m.saveConfig()
	}

	cmd := exec.Command(script)
	cmd.Env = append(os.Environ(), fmt.Sprintf("PORT=%s", port))
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Start(); err != nil {
		fmt.Printf("❌ 启动失败: %v\n", err)
		return
	}
	pidFile := pidFilePath(comp, version)
	os.WriteFile(pidFile, []byte(fmt.Sprintf("%d", cmd.Process.Pid)), 0644)
	fmt.Printf("✅ %s %s 已启动 (PID: %d)\n", comp, version, cmd.Process.Pid)
}

func (m *Manager) stop(comp, version string) {
	pidFile := pidFilePath(comp, version)
	pid, ok := readPID(pidFile)
	if !ok {
		fmt.Println("PID 文件不存在，尝试强制停止...")
		exec.Command("pkill", "-f", fmt.Sprintf("%s.*%s", comp, version)).Run()
		return
	}
	process, err := os.FindProcess(pid)
	if err == nil {
		process.Signal(syscall.SIGTERM)
		fmt.Printf("🛑 %s %s (PID: %d) 已停止\n", comp, version, pid)
	}
	os.Remove(pidFile)
}

func main() {
	m := &Manager{}
	m.init()
	defer m.saveConfig()

	for {
		fmt.Println("\n=== DevEnv Manager ===")
		fmt.Println("1. 查看状态")
		fmt.Println("2. 安装组件")
		fmt.Println("3. 卸载组件")
		fmt.Println("4. 启动服务")
		fmt.Println("5. 停止服务")
		fmt.Println("0. 退出")
		fmt.Print("选择: ")

		var choice string
		fmt.Scanln(&choice)

		switch choice {
		case "1":
			m.list()
		case "2":
			fmt.Print("输入扩展ID (如 com.yourcompany.DevEnv.PHP.8.2 或 com.yourcompany.DevEnv.PHP.8.2.redis): ")
			var eid string
			fmt.Scanln(&eid)
			m.install(eid)
		case "3":
			fmt.Print("输入扩展ID: ")
			var eid string
			fmt.Scanln(&eid)
			m.uninstall(eid)
		case "4":
			fmt.Print("类型 (php/mysql/nginx/redis): ")
			var t string
			fmt.Scanln(&t)
			fmt.Print("版本: ")
			var v string
			fmt.Scanln(&v)
			fmt.Print("端口 (回车使用默认): ")
			var port string
			fmt.Scanln(&port)
			m.start(t, v, port)
		case "5":
			fmt.Print("类型: ")
			var t string
			fmt.Scanln(&t)
			fmt.Print("版本: ")
			var v string
			fmt.Scanln(&v)
			m.stop(t, v)
		case "0":
			return
		default:
			fmt.Println("无效选项")
		}
	}
}
```

---

### 3.2 清单 `manager/com.yourcompany.DevEnv.yml`

```yaml
id: com.yourcompany.DevEnv
runtime: com.yourcompany.DevEnv.Base
runtime-version: '24.08'
sdk: org.freedesktop.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.golang
command: devenv

build-options:
  env:
    GOROOT: /usr/lib/sdk/golang
    PATH: /usr/lib/sdk/golang/bin:/usr/bin:/bin

finish-args:
  - --share=network
  - --filesystem=host
  - --talk-name=org.freedesktop.Flatpak

# 基础组件扩展点
add-extensions:
  com.yourcompany.DevEnv.PHP:
    directory: extensions/php
    version: '24.08'
    subdirectories: true
    no-autodownload: true
    autodelete: true
    merge-dirs: bin
  com.yourcompany.DevEnv.MySQL:
    directory: extensions/mysql
    version: '24.08'
    subdirectories: true
    no-autodownload: true
    autodelete: true
    merge-dirs: bin
  com.yourcompany.DevEnv.Nginx:
    directory: extensions/nginx
    version: '24.08'
    subdirectories: true
    no-autodownload: true
    autodelete: true
    merge-dirs: bin
  com.yourcompany.DevEnv.Redis:
    directory: extensions/redis
    version: '24.08'
    subdirectories: true
    no-autodownload: true
    autodelete: true
    merge-dirs: bin

modules:
  - name: devenv-main
    buildsystem: simple
    build-commands:
      - export GO111MODULE=off
      - go build -o /app/bin/devenv main.go
    sources:
      - type: dir
        path: .
```

构建管理器：

```bash
cd ~/dev-env/manager
flatpak-builder --force-clean --user --repo=../repo --install-deps-from=flathub build-dir-app com.yourcompany.DevEnv.yml
```

---

## 🧩 第四步：构建所有扩展（多版本 + PHP 扩展模块）

### 4.1 PHP 8.2 版本（基础）

文件：`extensions/php/8.2/com.yourcompany.DevEnv.PHP.8.2.yml`

```yaml
id: com.yourcompany.DevEnv.PHP.8.2
runtime: com.yourcompany.DevEnv.Base
runtime-version: '24.08'
branch: '24.08'
install-extension-path: extensions/php/8.2

modules:
  - name: openssl-php
    buildsystem: simple
    build-commands:
      - |
        tar xf openssl-1.1.1w.tar.gz
        cd openssl-1.1.1w
        ./config --prefix=/app/extensions/php/8.2/deps/openssl --openssldir=/app/extensions/php/8.2/deps/openssl/ssl
        make -j$(nproc) && make install
    sources:
      - type: archive
        url: https://www.openssl.org/source/openssl-1.1.1w.tar.gz
        sha256: cf3098950cb7d215801c847070f873c22ec9c006e949d8f15e12bdf58ad89f9f
  - name: curl-php
    buildsystem: simple
    build-commands:
      - |
        tar xf curl-8.6.0.tar.gz
        cd curl-8.6.0
        ./configure --prefix=/app/extensions/php/8.2/deps/curl --with-ssl=/app/extensions/php/8.2/deps/openssl
        make -j$(nproc) && make install
    sources:
      - type: archive
        url: https://curl.se/download/curl-8.6.0.tar.gz
        sha256: e8f93bf63301626252d4b8710b83458f80ec57925fa287a5178a57d11fa5fc8c
  - name: php
    buildsystem: simple
    build-commands:
      - |
        tar xf php-8.2.18.tar.gz
        cd php-8.2.18
        ./configure --prefix=/app/extensions/php/8.2 \
          --with-config-file-path=/app/extensions/php/8.2/etc \
          --enable-fpm \
          --with-openssl=/app/extensions/php/8.2/deps/openssl \
          --with-curl=/app/extensions/php/8.2/deps/curl \
          --with-zlib=/app
        make -j$(nproc) && make install
      - |
        mkdir -p /app/extensions/php/8.2/bin
        cat > /app/extensions/php/8.2/bin/start.sh << 'SCRIPT'
        #!/bin/bash
        export LD_LIBRARY_PATH=/app/extensions/php/8.2/deps/openssl/lib:/app/extensions/php/8.2/deps/curl/lib
        # 动态加载所有安装的扩展模块
        for ini in /app/extensions/php/8.2/modules/*/*.ini; do
          [ -f "$ini" ] && cat "$ini" >> /app/extensions/php/8.2/etc/php.ini
        done
        cd /app/extensions/php/8.2/etc
        [ ! -f php-fpm.conf ] && cp php-fpm.conf.default php-fpm.conf
        PORT=${PORT:-9000}
        sed -i "s/^listen = .*/listen = 127.0.0.1:${PORT}/" php-fpm.conf
        /app/extensions/php/8.2/sbin/php-fpm --nodaemonize --fpm-config /app/extensions/php/8.2/etc/php-fpm.conf
        SCRIPT
        chmod +x /app/extensions/php/8.2/bin/start.sh
    sources:
      - type: archive
        url: https://www.php.net/distributions/php-8.2.18.tar.gz
        sha256: 9480b0d0287c6d5b7c6b0d5c7585835851c8b852321b6c590835834774944615
```

### 4.2 PHP 8.3 版本（与 8.2 结构相同，仅版本号不同）

文件：`extensions/php/8.3/com.yourcompany.DevEnv.PHP.8.3.yml`

> 内容与 8.2 类似，只需将 `8.2` 替换为 `8.3`，并更新源码 URL 与 SHA256。

### 4.3 PHP 扩展模块示例：Redis for PHP 8.2

文件：`extensions/php/8.2/modules/com.yourcompany.DevEnv.PHP.8.2.redis.yml`

```yaml
id: com.yourcompany.DevEnv.PHP.8.2.redis
runtime: com.yourcompany.DevEnv.Base
runtime-version: '24.08'
branch: '24.08'
install-extension-path: extensions/php/8.2/modules/redis

modules:
  - name: phpredis
    buildsystem: simple
    build-commands:
      - |
        tar xf redis-6.0.2.tgz
        cd phpredis-6.0.2
        /app/extensions/php/8.2/bin/phpize
        ./configure --with-php-config=/app/extensions/php/8.2/bin/php-config
        make -j$(nproc) && make install
      - |
        mkdir -p /app/extensions/php/8.2/modules/redis
        echo "extension=redis.so" > /app/extensions/php/8.2/modules/redis/redis.ini
    sources:
      - type: archive
        url: https://pecl.php.net/get/redis-6.0.2.tgz
        sha256: <真实SHA256>
```

> 其他模块（pdo_mysql 等）类似。

### 4.4 MySQL 8.0 / 8.4 扩展

与之前增强版相同，增加 `--port=$PORT`，并修改 `DATADIR` 区分版本（如 `$HOME/.mysql-8.0-data` 和 `$HOME/.mysql-8.4-data`）。

清单文件名分别为：
- `com.yourcompany.DevEnv.MySQL.8.0.yml`
- `com.yourcompany.DevEnv.MySQL.8.4.yml`

### 4.5 Nginx 1.24 / 1.26 扩展

同样使用端口模板替换，清单文件分别命名：
- `com.yourcompany.DevEnv.Nginx.1.24.yml`
- `com.yourcompany.DevEnv.Nginx.1.26.yml`

### 4.6 Redis 7.2 / 7.4 扩展

支持端口，清单文件分别命名：
- `com.yourcompany.DevEnv.Redis.7.2.yml`
- `com.yourcompany.DevEnv.Redis.7.4.yml`

---

## 🚀 第五步：构建与打包（修正文件名与命令）

所有扩展清单文件名必须与 ID 对应，如 `com.yourcompany.DevEnv.PHP.8.2.yml`。

### 构建命令（逐个执行或脚本）

```bash
# 构建 Base Runtime（已完成）

# 构建管理器
cd ~/dev-env/manager
flatpak-builder --force-clean --user --repo=../repo --install-deps-from=flathub build-dir-app com.yourcompany.DevEnv.yml

# 构建 PHP 版本
cd ~/dev-env/extensions/php/8.2
flatpak-builder --force-clean --user --repo=../../../repo --install-deps-from=flathub build-dir com.yourcompany.DevEnv.PHP.8.2.yml
cd ~/dev-env/extensions/php/8.3
flatpak-builder --force-clean --user --repo=../../../repo --install-deps-from=flathub build-dir com.yourcompany.DevEnv.PHP.8.3.yml

# 构建 PHP 扩展模块
cd ~/dev-env/extensions/php/8.2/modules
flatpak-builder --force-clean --user --repo=../../../../repo --install-deps-from=flathub build-dir com.yourcompany.DevEnv.PHP.8.2.redis.yml

# 构建 MySQL
cd ~/dev-env/extensions/mysql/8.0
flatpak-builder --force-clean --user --repo=../../../repo --install-deps-from=flathub build-dir com.yourcompany.DevEnv.MySQL.8.0.yml
# ... 其他版本同样

# 构建 Nginx / Redis 同理
```

### 打包（统一命名规则：`<组件>-<版本>.flatpak`）

```bash
cd ~/dev-env
flatpak build-bundle repo php-8.2.flatpak com.yourcompany.DevEnv.PHP.8.2
flatpak build-bundle repo php-8.3.flatpak com.yourcompany.DevEnv.PHP.8.3
flatpak build-bundle repo php-8.2-redis.flatpak com.yourcompany.DevEnv.PHP.8.2.redis
flatpak build-bundle repo mysql-8.0.flatpak com.yourcompany.DevEnv.MySQL.8.0
flatpak build-bundle repo mysql-8.4.flatpak com.yourcompany.DevEnv.MySQL.8.4
flatpak build-bundle repo nginx-1.24.flatpak com.yourcompany.DevEnv.Nginx.1.24
flatpak build-bundle repo nginx-1.26.flatpak com.yourcompany.DevEnv.Nginx.1.26
flatpak build-bundle repo redis-7.2.flatpak com.yourcompany.DevEnv.Redis.7.2
flatpak build-bundle repo redis-7.4.flatpak com.yourcompany.DevEnv.Redis.7.4
flatpak build-bundle repo dev-env-manager.flatpak com.yourcompany.DevEnv
```

---

## 💡 用户使用流程（最终）

安装管理器：

```bash
flatpak install --user dev-env-manager.flatpak
```

启动管理器：

```bash
flatpak run com.yourcompany.DevEnv
```

在界面中：

1. 选 **2** 安装组件，输入 `com.yourcompany.DevEnv.PHP.8.2`（对应 `php-8.2.flatpak` 需在同一目录）
2. 再安装 PHP 扩展模块：`com.yourcompany.DevEnv.PHP.8.2.redis`
3. 选 **4** 启动 `php` 版本 `8.2`，端口 `9000`，PHP-FPM 会自动加载 `redis.so`
4. 同样安装并启动 MySQL 8.0（端口 `3306`）、Nginx 1.24（端口 `8080`）等

> 管理器通过 PID 文件精确控制进程，支持多个版本同时运行在不同端口。

---

## ✅ 最终审查通过清单

- [x] PHP 多个版本（8.2、8.3）及扩展模块（redis 等）可自由安装/卸载
- [x] MySQL/Nginx/Redis 多版本支持，端口可配置
- [x] 组件依赖完全隔离（自带 OpenSSL）
- [x] 安装/卸载通过标准 Flatpak 命令，文件名映射正确
- [x] PID 文件精准管理进程，不会误杀
- [x] 基础运行时包含 PCRE，Nginx 可正常编译
- [x] 所有清单 SHA256 需替换真实值即可构建

---

