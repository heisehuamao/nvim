# WSL Ubuntu 开发环境配置指南

本文整理 WSL Ubuntu 下常用的开发环境配置，包括字体、`en_US.UTF-8` locale、剪切板、`ripgrep`/`fd`，以及将 WSL Ubuntu 从 C 盘迁移到其他磁盘。

---

## 1. 安装 JetBrains Mono 字体

### 1.1 更新软件源并安装字体包

```bash
sudo apt update
sudo apt install -y fonts-jetbrains-mono
```

### 1.2 刷新 Linux 字体缓存

```bash
fc-cache -fv
```

### 1.3 验证字体是否安装成功

```bash
fc-list | grep -i "JetBrains Mono"
```

如果能够看到包含 `JetBrains Mono` 的字体信息，说明安装成功。

---

## 2. 安装并配置 `en_US.UTF-8` locale

在 WSL 中使用 Neovim、编译工具链或其他 Linux 开发工具时，建议配置完整的 UTF-8 locale。

### 2.1 安装 `locales`

```bash
sudo apt update
sudo apt install -y locales
```

### 2.2 生成 `en_US.UTF-8`

```bash
sudo locale-gen en_US.UTF-8
```

### 2.3 配置系统默认 locale

执行：

```bash
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

### 2.4 使配置立即生效

```bash
source ~/.bashrc
```

如果当前 shell 没有立即读取新的 locale 配置，也可以重新打开一个 WSL 终端。

### 2.5 检查 locale 配置

```bash
locale
```

预期可以看到：

```text
LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
```

配置成功后，使用相关 Linux 工具时不应再出现类似：

```text
warning: setlocale: ...
```

的 locale 警告。

---

## 3. 安装 `win32yank` 优化 WSL 剪切板

如果在 WSL 中使用 Neovim，并通过 PowerShell 获取 Windows 剪切板，可能会感觉到轻微延迟，例如按 `p` 后需要等待约半秒才能完成粘贴。

可以使用专门为 WSL 剪切板设计的 `win32yank.exe`。

### 3.1 下载并安装

```bash
curl -sLO https://github.com/equalsraf/win32yank/releases/latest/download/win32yank-x64.zip
unzip win32yank-x64.zip -d /tmp
sudo mv /tmp/win32yank.exe /usr/local/bin/
chmod +x /usr/local/bin/win32yank.exe
```

安装完成后，可以确认它能够被 PATH 找到：

```bash
which win32yank.exe
```

预期类似：

```text
/usr/local/bin/win32yank.exe
```

### 3.2 Neovim 配置

只要满足以下条件：

1. `win32yank.exe` 位于 `$PATH` 中；
2. Neovim 配置了：

```lua
vim.opt.clipboard = "unnamedplus"
```

Neovim 即可使用系统剪切板。

通常不需要额外配置 `vim.g.clipboard`。

---

## 4. 安装 `ripgrep`

`ripgrep`（`rg`）是很多 Neovim 搜索插件的重要依赖，尤其适合在大型源码树中进行快速文本搜索。

### 4.1 安装

```bash
sudo apt update
sudo apt install -y ripgrep
```

### 4.2 验证安装

```bash
rg --version
```

如果能够正常显示版本信息，说明安装成功。

---

## 5. 安装 `fd`

`fd` 是一个快速、简洁的文件查找工具。

很多 Neovim 文件搜索插件，例如 Telescope，会同时使用 `rg` 和 `fd`：

- `rg`：搜索文件内容
- `fd`：搜索文件名和路径

### 5.1 安装

Ubuntu 中软件包的命令名通常是 `fdfind`：

```bash
sudo apt install -y fd-find
```

### 5.2 创建 `fd` 软链接

为了可以直接使用 `fd` 命令：

```bash
sudo ln -s $(which fdfind) /usr/local/bin/fd
```

### 5.3 验证

```bash
fd --version
```

---

## 6. C 盘空间不足：迁移 WSL Ubuntu

### 6.1 为什么 WSL 会占用 C 盘空间？

WSL 2 的 Linux 文件系统通常存储在一个虚拟磁盘文件中，例如：

```text
ext4.vhdx
```

如果 WSL 安装在默认位置，这个文件通常会位于 C 盘。

当在 Linux 中进行以下操作时，`ext4.vhdx` 可能不断增长：

- 安装大量 Linux 软件包
- 下载 Linux kernel 源码
- 编译大型项目
- 构建 Docker 镜像
- 保存大量开发工具和依赖

最终可能导致 C 盘空间不足。

一种简单可靠的方法，是使用 WSL 的 `export/import` 功能，将整个 Ubuntu 实例迁移到 D 盘或其他容量更大的磁盘。

---

### 6.2 迁移步骤

以下命令在 **Windows PowerShell** 中执行。

#### 第一步：关闭 WSL

```powershell
wsl --shutdown
```

确保 Ubuntu 等 WSL 实例已经停止。

#### 第二步：导出 Ubuntu

将当前 Ubuntu 导出为 tar 文件：

```powershell
wsl --export Ubuntu D:\wsl-ubuntu-backup.tar
```

这里：

- `Ubuntu` 是 WSL 发行版名称
- `D:\wsl-ubuntu-backup.tar` 是临时备份文件

可以先通过以下命令确认发行版名称：

```powershell
wsl -l -v
```

#### 第三步：注销原来的 Ubuntu

```powershell
wsl --unregister Ubuntu
```

**注意：**

`--unregister` 会删除当前注册的 Ubuntu 实例及其原有虚拟磁盘。

因此，执行之前必须确认 `wsl --export` 已经成功完成，并且 `.tar` 文件存在。

#### 第四步：导入到 D 盘

创建目标目录并将 Ubuntu 导入：

```powershell
wsl --import Ubuntu D:\WSL\Ubuntu D:\wsl-ubuntu-backup.tar --version 2
```

其中：

```text
Ubuntu                 WSL 发行版名称
D:\WSL\Ubuntu          新的 Linux 文件系统存储位置
D:\wsl-ubuntu-backup.tar
                       导出的 Ubuntu 镜像
--version 2            使用 WSL 2
```

导入完成后，Ubuntu 的 Linux 文件系统就会存储在 D 盘指定目录下。

#### 第五步：删除临时备份

确认新 Ubuntu 可以正常启动后，再删除 tar 文件：

```powershell
del D:\wsl-ubuntu-backup.tar
```

---

## 7. 迁移后检查

### 查看 WSL 发行版

```powershell
wsl -l -v
```

应该能够看到类似：

```text
  NAME      STATE           VERSION
* Ubuntu    Stopped         2
```

### 启动 Ubuntu

```powershell
wsl -d Ubuntu
```

进入后，可以检查之前安装的软件是否仍然存在：

```bash
rg --version
fd --version
locale
```

也可以检查字体：

```bash
fc-list | grep -i "JetBrains Mono"
```

---

## 8. 推荐的配置顺序

如果这是一个新创建的 WSL Ubuntu 开发环境，可以按照下面的顺序进行初始化：

### Step 1：更新系统软件源

```bash
sudo apt update
```

### Step 2：配置 UTF-8 locale

```bash
sudo apt install -y locales
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
```

重新打开终端后检查：

```bash
locale
```

### Step 3：安装开发常用工具

```bash
sudo apt install -y \
    fonts-jetbrains-mono \
    ripgrep \
    fd-find \
    unzip \
    curl
```

### Step 4：配置字体

```bash
fc-cache -fv
fc-list | grep -i "JetBrains Mono"
```

### Step 5：配置 `fd`

```bash
sudo ln -s $(which fdfind) /usr/local/bin/fd
```

### Step 6：安装 `win32yank`

```bash
curl -sLO https://github.com/equalsraf/win32yank/releases/latest/download/win32yank-x64.zip
unzip win32yank-x64.zip -d /tmp
sudo mv /tmp/win32yank.exe /usr/local/bin/
chmod +x /usr/local/bin/win32yank.exe
```

### Step 7：配置 Neovim 剪切板

```lua
vim.opt.clipboard = "unnamedplus"
```

完成以上配置后，WSL Ubuntu 就具备了较完整的 Neovim / Linux 源码开发基础环境。

---

## 9. 常用验证命令汇总

| 功能 | 命令 |
|---|---|
| 查看 WSL 发行版 | `wsl -l -v` |
| 启动指定 Ubuntu | `wsl -d Ubuntu` |
| 关闭 WSL | `wsl --shutdown` |
| 检查 locale | `locale` |
| 检查 JetBrains Mono | `fc-list \| grep -i "JetBrains Mono"` |
| 刷新字体缓存 | `fc-cache -fv` |
| 检查 ripgrep | `rg --version` |
| 检查 fd | `fd --version` |
| 检查 win32yank | `which win32yank.exe` |
