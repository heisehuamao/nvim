# WSL 下使用 Neovim 查看 Linux Kernel 源码

本文整理在 WSL2 + Ubuntu 环境中使用 Neovim / LazyVim 阅读 Linux Kernel 源码时的环境准备步骤，以及网络、证书、编译器、Node.js/NVM 和 Windows PATH 混入等常见问题的解决方法。

> 说明：文档中的代理端口 `7890` 仅为示例，请根据实际代理软件配置修改。

---

## 1. WSL 基本操作

### 1.1 查看已安装的 Linux 发行版及状态

在 Windows PowerShell 中：

```powershell
wsl -l -v
```

### 1.2 运行指定发行版

例如运行 Ubuntu：

```powershell
wsl -d Ubuntu
```

### 1.3 停止/关闭 WSL

```powershell
wsl --shutdown
```

修改 `/etc/wsl.conf` 后通常需要执行此命令，使配置重新生效。

---

# 2. 安装 Ubuntu

## 2.1 使用 Web Direct 下载

如果 Microsoft Store 网络访问受限，可以使用 `--web-download`，直接通过 Web 下载。

在管理员 PowerShell 中：

```powershell
wsl --install -d Ubuntu --web-download
```

---

## 2.2 PowerShell 配置代理

如果 Web 下载仍然无法正常访问，可以为当前 PowerShell 窗口设置 HTTP/HTTPS 代理。

例如代理端口为 `7890`：

```powershell
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"
```

然后重新执行：

```powershell
wsl --install -d Ubuntu --web-download
```

> `127.0.0.1:7890` 只是示例，需要根据实际代理软件的端口修改。

---

# 3. WSL 配置网络代理

WSL2 中的 Linux 与 Windows 主机处于不同的网络环境，因此在 WSL 中通常需要使用 Windows 主机的网关 IP，而不是直接使用 Linux 内的 `127.0.0.1`。

## 3.1 获取 Windows 主机 IP

推荐使用 Linux 自带的 `ip` 命令：

```bash
ip addr
```

也可以直接获取默认网关：

```bash
host_ip=$(ip route show | grep -i default | awk '{print $3}')
```

不需要为了查看 IP 专门安装 `net-tools`。

如果确实需要 `ifconfig` 等旧工具，可以安装：

```bash
sudo apt install -y net-tools
```

---

## 3.2 设置 HTTP/HTTPS 代理

```bash
export http_proxy="http://${host_ip}:7890"
export https_proxy="http://${host_ip}:7890"
```

也可以同时设置大写变量：

```bash
export HTTP_PROXY="http://${host_ip}:7890"
export HTTPS_PROXY="http://${host_ip}:7890"
```

---

## 3.3 测试网络代理

例如：

```bash
curl -I https://www.google.com
```

如果能够正常获得 HTTP 响应，说明代理基本可用。

---

# 4. 配置 apt 使用代理

如果 `apt update` 或安装软件包时经常无法连接，可以给 apt 单独配置代理。

## 4.1 写入 apt 代理配置

```bash
host_ip=$(ip route show | grep -i default | awk '{print $3}')

echo "Acquire::http::Proxy \"http://${host_ip}:7890\";" | sudo tee /etc/apt/apt.conf.d/99proxy

echo "Acquire::https::Proxy \"http://${host_ip}:7890\";" | sudo tee -a /etc/apt/apt.conf.d/99proxy
```

然后：

```bash
sudo apt update
```

## 4.2 强制 apt 使用 IPv4

如果遇到 IPv6 无法连接或连接超时，可以：

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

然后：

```bash
sudo apt update
```

---

# 5. Windows 资源管理器访问 WSL 文件

WSL2 提供了 Windows 资源管理器访问 Linux 文件系统的方式。

## 5.1 打开 WSL 文件系统

按：

```text
Win + E
```

在地址栏输入：

```text
\\wsl$
```

或者针对指定发行版：

```text
\\wsl.localhost\Ubuntu
```

回车后即可看到 Linux 文件系统，例如：

```text
/home/your_username
```

可以像操作 Windows 文件一样进行复制、粘贴和拖拽。

---

# 6. Neovim 配置目录

WSL / Linux 环境下，Neovim 配置目录通常为：

```bash
~/.config/nvim
```

进入：

```bash
cd ~/.config/nvim
```

---

# 7. LazyVim 推荐目录结构

LazyVim 常见的配置结构：

```text
nvim/
├── lua/
│   ├── config/
│   │   ├── autocmds.lua    # 自动命令（Autocommands）
│   │   ├── keymaps.lua     # 快捷键映射
│   │   ├── lazy.lua        # LazyVim 核心初始化
│   │   └── options.lua     # Neovim 基础选项（vim.opt）
│   │
│   └── plugins/
│       ├── example.lua     # 插件配置示例
│       └── spec.lua        # 自定义/覆盖插件配置
│
└── init.lua                # Neovim 启动入口
```

---

# 8. WSL 中的 CA 根证书问题

在安装 LazyVim、插件或从 GitHub 下载代码时，可能遇到：

- `SSL certificate problem`
- `certificate verify failed`
- GitHub HTTPS 连接失败
- Git clone 失败
- curl HTTPS 请求失败

首先推荐修复系统 CA 根证书。

## 8.1 更新/重新安装 CA 证书

```bash
sudo apt update
sudo apt install --reinstall -y ca-certificates
sudo update-ca-certificates
```

然后重新尝试 Git、curl 或插件安装。

---

# 9. Git SSL 证书验证问题

如果只是为了快速让 Git 和 lazy.nvim 拉取 GitHub 代码，可以临时关闭 SSL 验证。

## 9.1 仅对 GitHub 关闭验证

相对更推荐：

```bash
git config --global http.https://github.com/.sslVerify false
```

## 9.2 全局关闭 Git SSL 验证

如果上一种方式仍然不能解决：

```bash
git config --global http.sslVerify false
```

> **安全提醒：** 关闭 SSL 验证会降低 HTTPS 安全性。正常情况下，应优先修复 CA 证书和代理配置。只建议把关闭验证作为排查网络问题或临时使用的方案。

---

# 10. curl SSL 证书问题

如果 curl 因为证书错误无法访问，可以配置 `~/.curlrc`。

## 10.1 全局忽略 curl SSL 证书错误

```bash
echo "insecure" >> ~/.curlrc
```

测试：

```bash
curl -I https://github.com
```

> **安全提醒：** 这会让当前用户的 curl 默认跳过证书验证，存在安全风险。长期使用更推荐修复 CA 证书。

---

# 11. wget SSL 证书问题

## 11.1 临时跳过证书检查

例如下载 Linux Kernel 源码：

```bash
wget --no-check-certificate https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.8.12.tar.xz
```

## 11.2 全局配置 wget

如果经常使用 wget，可以：

```bash
echo "check_certificate = off" >> ~/.wgetrc
```

这样 wget 默认关闭证书检查。

> 同样不建议长期关闭 HTTPS 证书验证。

---

# 12. nvim-treesitter 缺少 C Compiler

安装或更新 `nvim-treesitter` 时可能出现：

```text
Unmet requirements for nvim-treesitter main:

- ❌ C compiler
```

这是因为 Treesitter parser 需要进行本地编译，而基础 Ubuntu 环境可能没有完整的 C/C++ 编译工具链。

## 12.1 安装 build-essential

```bash
sudo apt update
sudo apt install -y build-essential
```

`build-essential` 会安装常用的基础编译工具，例如 GCC、G++、make 等。

安装完成后重新执行 Neovim/LazyVim 中的插件安装或更新操作。

---

# 13. 在 WSL 中安装 NVM 和 Node.js

部分 Neovim 插件和工具链依赖 Node.js。

推荐使用 NVM 管理 Node.js，而不是直接依赖系统版本。

## 13.1 安装 NVM

使用官方安装脚本：

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

如果 GitHub 网络受阻，可以使用镜像脚本：

```bash
curl -o- https://gitee.com/mirrors/nvm/raw/v0.39.7/install.sh | bash
```

## 13.2 加载 NVM 配置

安装完成后：

```bash
source ~/.bashrc
```

验证：

```bash
nvm --version
```

如果能够显示版本号，例如：

```text
0.39.7
```

说明 NVM 已基本安装成功。

## 13.3 安装并切换到 Node.js LTS

```bash
nvm install --lts
nvm use --lts
```

## 13.4 验证 Node.js/npm 路径

```bash
which node
which npm
```

应确认路径指向 WSL/Linux 环境，而不是 Windows 的 Node/npm。

---

# 14. 禁止 WSL 自动混入 Windows PATH

默认情况下，WSL 会自动把 Windows 的 PATH 加入 Linux 的 `$PATH`。

这可能导致 Linux 环境中意外调用 Windows 程序，例如：

```text
C:\...
```

从而出现：

- Linux 和 Windows 两套 Node.js 混用
- `npm` 指向 Windows
- Neovim 插件调用 Windows 工具
- Node/npm 版本混乱
- Linux 工具链和 Windows 工具链互相干扰

如果希望 WSL 更接近一个纯 Linux 环境，可以关闭 Windows PATH 自动注入。

## 14.1 修改 `/etc/wsl.conf`

编辑：

```bash
sudo nano /etc/wsl.conf
```

写入或修改：

```ini
[interop]
appendWindowsPath = false
```

保存后，在 Windows PowerShell 中执行：

```powershell
wsl --shutdown
```

然后重新启动 Ubuntu。

## 14.2 检查 PATH

重新进入 WSL 后：

```bash
echo $PATH
```

检查 Node/npm：

```bash
which node
which npm
```

确保使用的是 Linux/WSL 中安装的版本。

---

# 15. 推荐的完整环境准备流程

如果最终目标是：

> 在 WSL 中使用 Neovim + LazyVim 阅读 Linux Kernel 源码

可以按照下面的顺序配置。

## Step 1：安装并启动 WSL

Windows PowerShell：

```powershell
wsl --install -d Ubuntu --web-download
```

查看状态：

```powershell
wsl -l -v
```

启动：

```powershell
wsl -d Ubuntu
```

## Step 2：配置网络

获取 Windows 主机 IP：

```bash
host_ip=$(ip route show | grep -i default | awk '{print $3}')
```

设置代理：

```bash
export http_proxy="http://${host_ip}:7890"
export https_proxy="http://${host_ip}:7890"
```

测试：

```bash
curl -I https://www.google.com
```

## Step 3：配置 apt 代理

```bash
host_ip=$(ip route show | grep -i default | awk '{print $3}')

echo "Acquire::http::Proxy \"http://${host_ip}:7890\";" | sudo tee /etc/apt/apt.conf.d/99proxy
echo "Acquire::https::Proxy \"http://${host_ip}:7890\";" | sudo tee -a /etc/apt/apt.conf.d/99proxy
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

然后：

```bash
sudo apt update
```

## Step 4：安装基础开发工具

```bash
sudo apt install -y build-essential git curl wget
```

如果确实需要 `ifconfig` 等工具：

```bash
sudo apt install -y net-tools
```

## Step 5：修复 CA 证书

```bash
sudo apt install --reinstall -y ca-certificates
sudo update-ca-certificates
```

## Step 6：安装 NVM 和 Node.js

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
```

验证：

```bash
nvm --version
```

安装 LTS：

```bash
nvm install --lts
nvm use --lts
```

检查：

```bash
which node
which npm
```

## Step 7：配置 WSL PATH

如果希望禁止 Windows PATH 自动注入：

```bash
sudo nano /etc/wsl.conf
```

加入：

```ini
[interop]
appendWindowsPath = false
```

然后在 Windows PowerShell：

```powershell
wsl --shutdown
```

重新启动 Ubuntu。

## Step 8：配置 Neovim

进入：

```bash
cd ~/.config/nvim
```

LazyVim 配置可以按照：

```text
~/.config/nvim/
├── init.lua
└── lua/
    ├── config/
    │   ├── autocmds.lua
    │   ├── keymaps.lua
    │   ├── lazy.lua
    │   └── options.lua
    │
    └── plugins/
        ├── example.lua
        └── spec.lua
```

---

# 16. 常见问题速查表

| 问题 | 解决方法 |
|---|---|
| 查看 WSL 发行版 | `wsl -l -v` |
| 启动 Ubuntu | `wsl -d Ubuntu` |
| 关闭 WSL | `wsl --shutdown` |
| Microsoft Store 下载失败 | `wsl --install -d Ubuntu --web-download` |
| PowerShell 下载失败 | 设置 `$env:HTTP_PROXY` / `$env:HTTPS_PROXY` |
| 查看 Linux IP | `ip addr` / `ip a` |
| 获取 WSL 默认网关 | `ip route show \| grep -i default` |
| WSL 无法访问外网 | 配置 `http_proxy` / `https_proxy` |
| apt 无法联网 | 配置 `/etc/apt/apt.conf.d/99proxy` |
| apt IPv6 连接失败 | 配置 `Acquire::ForceIPv4 "true"` |
| GitHub SSL 错误 | 重新安装 `ca-certificates` |
| GitHub 临时关闭 SSL 验证 | `git config --global http.https://github.com/.sslVerify false` |
| Git 全局关闭 SSL 验证 | `git config --global http.sslVerify false` |
| curl SSL 错误 | `~/.curlrc` 设置 `insecure` |
| wget SSL 错误 | `wget --no-check-certificate` |
| nvim-treesitter 缺 C compiler | `sudo apt install -y build-essential` |
| 安装 NVM | 使用 NVM 安装脚本 |
| 安装 Node.js | `nvm install --lts` |
| npm 指向 Windows | 检查 `which npm` 和 `$PATH` |
| 禁止混入 Windows PATH | `/etc/wsl.conf` 中设置 `appendWindowsPath = false` |
| 修改 WSL 配置后不生效 | Windows 执行 `wsl --shutdown` |
| Windows 查看 WSL 文件 | `\\wsl.localhost\Ubuntu` |

---

# 17. 最终环境结构

完成配置后，整体环境可以理解为：

```text
Windows
│
├── PowerShell
│   ├── wsl -l -v
│   ├── wsl -d Ubuntu
│   └── wsl --shutdown
│
└── WSL2
    │
    └── Ubuntu
        │
        ├── Linux filesystem
        │   ├── /home/...
        │   ├── /etc/...
        │   └── /usr/...
        │
        ├── Network
        │   ├── Windows host IP
        │   └── HTTP/HTTPS proxy
        │
        ├── Development tools
        │   ├── gcc
        │   ├── g++
        │   ├── make
        │   ├── git
        │   ├── curl
        │   └── wget
        │
        ├── Node.js
        │   ├── NVM
        │   └── npm
        │
        └── Neovim
            │
            └── ~/.config/nvim
                ├── init.lua
                └── lua/
                    ├── config/
                    └── plugins/
```
