---
title: "从零开始：SSH 远程连接计算服务器"
date: 2026-05-15
draft: false
tags: ["入门", "SSH", "Linux", "环境配置"]
categories: ["基础入门"]
series: ["从零开始学计算"]
summary: "第一篇教程：从 Windows/WSL 环境通过 SSH 连接到远程 Linux 计算服务器，包括密钥配置、免密登录和常见问题排查。"
---

## 为什么需要 SSH？

合成生物学中的计算分析——无论是 AlphaFold 预测蛋白结构，还是 Rosetta 做酶设计——通常需要在高性能计算服务器（HPC）上运行。你的个人电脑只是"遥控器"，真正的计算在远程服务器上完成。

本篇面向**完全零基础**的读者，手把手教你建立从本地到服务器的 SSH 连接。

## 第一步：确认你有服务器账号

开始之前，你需要从管理员处获取以下信息：

| 信息 | 示例 | 说明 |
|------|------|------|
| 服务器地址 | `10.0.0.100` 或 `lab-server.example.com` | IP 地址或域名 |
| 用户名 | `zhangsan` | 你的登录名 |
| 密码 | `******` | 初始密码（首次登录后建议修改） |
| 端口 | `22` | SSH 默认端口，部分服务器可能使用其他端口 |

> 如果没有这些信息，联系你的实验室管理员或导师。

## 第二步：本地环境准备

### Windows 用户

推荐使用 **WSL2（Windows Subsystem for Linux）**，它让你在 Windows 上运行完整的 Linux 环境。

**安装 WSL2：**

以管理员身份打开 PowerShell，运行：

```powershell
wsl --install
```

安装完成后重启电脑，首次进入 Ubuntu 时会提示创建用户名和密码。

### macOS / Linux 用户

macOS 和 Linux 自带 SSH 客户端，无需额外安装。打开终端（Terminal）即可使用。

## 第三步：首次 SSH 连接

打开终端（WSL 中打开 Ubuntu），输入：

```bash
ssh username@server_address
```

例如：

```bash
ssh zhangsan@10.0.0.100
```

如果服务器使用非默认端口（比如 2222），加上 `-p` 参数：

```bash
ssh -p 2222 zhangsan@10.0.0.100
```

**首次连接提示：**

```
The authenticity of host '10.0.0.100' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no)?
```

输入 `yes` 并回车。这是正常的——你的电脑第一次认识这台服务器，需要确认它的"身份证"。

然后输入密码（输入时不显示字符，这是安全设计），回车即可登录。

## 第四步：配置 SSH 密钥（免密登录）

每次输密码很麻烦，而且密码登录安全性较低。SSH 密钥用一对"锁和钥匙"替代密码。

### 生成密钥对

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

一路回车即可（使用默认路径 `~/.ssh/id_ed25519`）。如果已有密钥，可以跳过这一步。

### 将公钥上传到服务器

```bash
ssh-copy-id username@server_address
```

输入一次密码后，以后登录就不用再输密码了！

如果服务器不支持 `ssh-copy-id`，手动操作：

```bash
cat ~/.ssh/id_ed25519.pub | ssh username@server_address "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 验证免密登录

```bash
ssh username@server_address
```

如果直接进入服务器而无需输入密码，恭喜，配置成功！

## 第五步：SSH 配置文件（可选但推荐）

当你有多个服务器时，手动输入 IP 地址很麻烦。在 `~/.ssh/config` 中配置别名：

```
Host myserver
    HostName 10.0.0.100
    User zhangsan
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host lab-gpu
    HostName gpu-server.example.com
    User zhangsan
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

配置后，只需 `ssh myserver` 或 `ssh lab-gpu` 即可连接。

## 常见问题

### 1. "Connection refused"

- 服务器是否开机？
- IP 地址和端口是否正确？
- 是否在校园网/VPN 环境下？

### 2. "Permission denied (publickey)"

- 公钥是否已正确上传到 `~/.ssh/authorized_keys`？
- 本地私钥权限是否正确？运行 `chmod 600 ~/.ssh/id_ed25519`

### 3. 连接后很快断开

服务器可能设置了空闲超时。在 `~/.ssh/config` 中添加：

```
Host *
    ServerAliveInterval 60
```

## 下一步

成功登录服务器后，下一篇教程将介绍如何在服务器上安装 Conda 环境，为后续安装 AlphaFold、Rosetta 等工具做准备。
