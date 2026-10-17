---
x-title: 通过 dnf / yum / zypper 安装 x-cmd
x-desc: 在 Fedora / RHEL / openSUSE 上添加官方 x-cmd RPM 仓库，安装 x-cmd 包，一行命令即可验证。
x-sidebar: 安装
x-keywords: x-cmd, dnf, yum, zypper, Fedora, RHEL, CentOS, Rocky, Alma, openSUSE, 安装, .repo, noarch
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '通过 dnf / yum / zypper 安装 x-cmd'
      inLanguage: 'zh-Hans'
      about: '在 Fedora / RHEL / openSUSE 上通过 RPM 安装 x-cmd'
---

# 通过 dnf / yum / zypper 安装 x-cmd

> 官方 x-cmd RPM 仓库，部署在 <https://www.x-cmd.com/rpm/>。
> Fedora / RHEL / CentOS / Rocky / Alma / openSUSE 都通过标准
> `Packages/` + `repodata/` 布局消费同一个仓库。

包是 `noarch`（架构无关）——一份产物服务所有 RPM 架构。

## 快速安装

### Fedora / RHEL / CentOS / Rocky / Alma（dnf）

```sh
# 1. 启用仓库（下载 x-cmd.repo 到 /etc/yum.repos.d/）
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo

# 2. 安装 x-cmd
sudo dnf install -y x-cmd
```

验证：

```sh
x --version
```

应该看到 `0.9.9`（或更新版本）。

### 旧版 RHEL / CentOS 7（yum）

```sh
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo
sudo yum install -y x-cmd
```

### openSUSE（zypper）

```sh
sudo zypper ar -f https://www.x-cmd.com/rpm/ x-cmd
sudo zypper refresh
sudo zypper in x-cmd
```

### 手动放 .repo（不需要 plugins）

```sh
# 任何 dnf / yum / zypper 发行版
sudo curl -fsSL https://www.x-cmd.com/rpm/x-cmd.repo \
    -o /etc/yum.repos.d/x-cmd.repo
sudo dnf install -y x-cmd
```

`x-cmd.repo` 是本仓库和包一起发布的、标准的 `.repo`
片段。它设置 `baseurl`、`enabled=1`，以及（目前）
`gpgcheck=0` / `repo_gpgcheck=0`。

## 安装内容

| 项 | 值 |
| --- | --- |
| 包名 | `x-cmd` |
| 版本 | `0.9.9`（或更新） |
| 架构 | `noarch`（与架构无关的 shell 代码） |
| 维护者 | Li Junhao <l@x-cmd.com> |
| 许可证 | Apache-2.0 |

包是 `noarch`——一份 `.rpm` 服务所有 RPM 客户端。dnf 在
x86_64、aarch64 等不同架构上装同一份产物。

## 为什么要 `gpgcheck=0`（目前）

仓库目前**未签名**——`x-cmd.repo` 声明 `gpgcheck=0` 和
`repo_gpgcheck=0`。还没有公开发布的 release GPG key。这是
已知的缺口；见 [安全](../2-security/cn.md) 里有 key 发布后
开启签名元数据的计划。

`gpgcheck=0` 是对未签名包的光学接收。**key 发布后要去掉**。

## 手动校验包

```sh
rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
# 今天：只显示 digest + payload 校验，无签名
# 签名上线后：还会显示 "V3 RSA/SHA256 Signature, key ID ..."

# 或下载 + 校验 digest
curl -fsSLO https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
rpm -Kv x-cmd_0.9.9_all.rpm
```

签名上线后最严谨的做法：

```sh
rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
sudo dnf config-manager --save --setopt=x-cmd.gpgcheck=1
sudo dnf config-manager --save --setopt=x-cmd.repo_gpgcheck=1
sudo dnf update x-cmd
```

## 下一步

- 想自己打包 `.rpm` 通过 dnf / yum / zypper 分发？看
  [打包](../1-packaging/cn.md)。
- 想了解签名计划、上线后改什么？看 [安全](../2-security/cn.md)。
- 想看仓库目前对外发布什么（版本、checksum、架构）？
  看 [统计](../3-stats/cn.md)。

数据源：[`README.md`](../../README.md)、
[`x-cmd.repo`](../../x-cmd.repo)、
和 `Packages/x-cmd_0.9.9_all.rpm` 截至 `git log` 锁定的提交。
