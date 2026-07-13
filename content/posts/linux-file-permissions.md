---
title: "Linux 文件权限详解：chmod、chown 与 umask"
date: 2026-07-13
draft: false
tags: ["文件系统", "权限管理", "基础命令"]
summary: "深入理解 Linux 文件权限模型，掌握 chmod、chown 和 umask 的使用技巧。"
---

## 文件权限模型

Linux 是一个多用户操作系统，每个文件和目录都有三组权限：

- **所有者（Owner）** — 文件创建者
- **所属组（Group）** — 与文件关联的用户组
- **其他用户（Others）** — 其他所有人

每组权限有三种类型：

| 权限 | 文件 | 目录 |
|------|------|------|
| `r` (4) | 读取文件内容 | 列出目录内容 |
| `w` (2) | 修改文件 | 创建/删除目录中的文件 |
| `x` (1) | 执行文件 | 进入目录 |

使用 `ls -l` 可以查看权限：

```bash
$ ls -l
-rw-r--r-- 1 root root  1024 Jul 13 10:00 notes.txt
drwxr-xr-x 2 root root  4096 Jul 13 10:00 scripts/
```

权限字符串 `-rw-r--r--` 分解如下：

```
-  rw-  r--  r--
|  |    |    |
|  |    |    └─ 其他用户: r--
|  |    └────── 所属组: r--
|  └────────── 所有者: rw-
└──────────── 文件类型 (- 普通文件, d 目录)
```

## chmod：修改权限

### 符号模式

```bash
# 给所有者添加执行权限
chmod u+x script.sh

# 移除其他人的读取权限
chmod o-r secret.txt

# 给所有者和组添加读写权限
chmod ug+rw data.txt

# 给所有权限设置（所有者 rwx, 组 rx, 其他人 rx）
chmod u=rwx,g=rx,o=rx directory/
```

### 数字模式

每个权限用一个八进制数字表示：

- `r` = 4
- `w` = 2
- `x` = 1

三组权限的和构成三位数字：

```bash
chmod 755 script.sh
# 所有者: 7 (rwx), 组: 5 (r-x), 其他人: 5 (r-x)

chmod 644 notes.txt
# 所有者: 6 (rw-), 组: 4 (r--), 其他人: 4 (r--)

chmod 700 private/
# 所有者: 7 (rwx), 组: 0 (---), 其他人: 0 (---)
```

常用权限组合速查：

| 权限 | 数字 | 说明 |
|------|------|------|
| `-rw-------` | 600 | 仅所有者可读写 |
| `-rw-r--r--` | 644 | 所有者读写，其他人只读（常见文件） |
| `-rwx------` | 700 | 仅所有者可执行 |
| `-rwxr-xr-x` | 755 | 所有人可执行（常见脚本） |
| `-rwxrwxrwx` | 777 | 所有人完全控制（**不推荐**） |

### 递归修改

```bash
# 对整个目录及其内容应用权限
chmod -R 755 /path/to/directory/
```

> **注意**：对目录使用 `-R` 时，可能不希望所有文件都可执行。更安全的方式是分两次：
> ```bash
> find /path -type d -exec chmod 755 {} \;
> find /path -type f -exec chmod 644 {} \;
> ```

## chown：修改所有者和组

```bash
# 修改文件所有者
sudo chown alice notes.txt

# 同时修改所有者和组（用冒号分隔）
sudo chown alice:developers notes.txt

# 只修改所属组
sudo chown :developers notes.txt

# 递归修改目录
sudo chown -R alice:developers /home/alice/
```

## umask：默认权限掩码

`umask` 决定了新建文件/目录的默认权限。它表示"要移除的权限"：

```bash
# 查看当前 umask
umask
# 输出: 0022

# 计算默认权限
# 文件: 666 - 022 = 644  (rw-r--r--)
# 目录: 777 - 022 = 755  (rwxr-xr-x)
```

常见 umask 值：

| umask | 文件默认 | 目录默认 | 适用场景 |
|-------|----------|----------|----------|
| 022 | 644 | 755 | 通用，其他人可读 |
| 002 | 664 | 775 | 同组用户可写 |
| 077 | 600 | 700 | 严格私有 |
| 007 | 660 | 770 | 仅组内成员访问 |

临时修改：

```bash
umask 077  # 只在当前 shell 生效
```

持久化修改需要写入 `~/.bashrc` 或 `~/.profile`。

## 特殊权限位

除了基本的 rwx，还有三个特殊权限：

### SUID (4xxx)

当可执行文件设置了 SUID，其他用户执行时会**临时获得文件所有者的权限**：

```bash
chmod u+s /usr/bin/program
# 或: chmod 4755 program
```

典型例子：`/usr/bin/passwd` 需要 root 权限修改密码。

> 注意：SUID 对 Shell 脚本无效，且存在安全风险，谨慎使用。

### SGID (2xxx)

文件设置了 SGID 后，执行时会获得文件所属组的权限。目录设置了 SGID 后，**在该目录下新建的文件会自动继承目录的组**：

```bash
chmod g+s shared_directory/
# 或: chmod 2755 shared_directory/
```

这对团队协作非常有用。

### Sticky Bit (1xxx)

主要用于共享目录，**只有文件所有者（或 root）才能删除自己的文件**：

```bash
chmod +t /tmp/
# 或: chmod 1777 /tmp/
```

典型例子：`/tmp` 目录，所有人可以写入，但不能删除别人的文件。

检查特殊权限：

```bash
ls -l /tmp/
# drwxrwxrwt ...   (t 表示 sticky bit)
# drwxr-sr-x ...   (s 在组权限位置 = SGID)
# -rwsr-xr-x ...   (s 在所有者位置 = SUID)
```

## 实战练习

```bash
# 1. 创建一个团队共享目录
sudo mkdir /srv/project
sudo chown root:developers /srv/project
sudo chmod 2770 /srv/project
# 结果: drwxrws---, 新文件自动属于 developers 组

# 2. 创建一个只能由你查看的私钥文件
touch ~/.ssh/mykey
chmod 600 ~/.ssh/mykey
# 结果: -rw-------

# 3. 修复常见的权限问题
chmod 755 ~  # 确保主目录可进入
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
```
