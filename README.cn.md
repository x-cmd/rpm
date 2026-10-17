# x-cmd RPM 仓库

> `https://www.x-cmd.com/rpm/` —— 一个 yum / dnf / zypper 仓
> 库，托管官方 **x-cmd** RPM 包。

本仓库按 yum/dnf 标准布局（`Packages/` + `repodata/`）原样
发布到 `https://www.x-cmd.com/rpm/`，所以对任何 RPM 客户端
（`dnf`、`yum`、`zypper`）都直接可用。

包由上游
[x-cmd/release](https://github.com/x-cmd/release) 构建并发
布；本仓库只镜像 `.rpm` 产物并维护 `repodata/` 元数据。

---

## 安装（Fedora / RHEL / CentOS / Rocky / Alma）

```sh
# 1. 启用仓库
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://www.x-cmd.com/rpm/x-cmd.repo

# 2. 安装 x-cmd
sudo dnf install -y x-cmd
```

一行版本：

```sh
sudo curl -fsSL https://www.x-cmd.com/rpm/x-cmd.repo \
    -o /etc/yum.repos.d/x-cmd.repo
sudo dnf install -y x-cmd
```

验证：

```sh
x --version
```

---

## 安装（openSUSE）

```sh
sudo zypper ar -f https://www.x-cmd.com/rpm/ x-cmd
sudo zypper refresh
sudo zypper in x-cmd
```

---

## 仓库布局

```
.
├── Packages/                       # 托管的 .rpm 产物
│   └── x-cmd_0.9.9_all.rpm
├── repodata/                       # createrepo_c 生成的元数据
│   ├── repomd.xml                  #   客户端最先 fetch 的入口
│   ├── <sha>-primary.xml.gz        #   Name, Version, Requires...
│   ├── <sha>-filelists.xml.gz      #   每个包的文件清单
│   └── <sha>-other.xml.gz          #   changelogs、依赖
├── x-cmd.repo                      # 开箱即用的 dnf/yum 配置
├── README.md                       # 英文 README
├── README.cn.md                    # 本文件
└── docs/                           # 双语文档，每个主题一个目录
    ├── 0-install/    # en.md, cn.md, llms.md, faq.yml
    ├── 1-packaging/
    ├── 2-security/
    └── 3-stats/
```

`baseurl` 是仓库根，包的 `<location href>` 是相对路径
`Packages/x-cmd_0.9.9_all.rpm`。

包是 `noarch`（架构无关）——一份 RPM 服务所有 RPM 支持的架构。

---

## 新版本发布

1. 把新的 `.rpm` 放进 `Packages/`（可选地删掉过期的）。
2. 用 `createrepo_c` 重建元数据。macOS 没有 `createrepo_c`，
   跑容器：

   ```sh
   docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
     dnf install -y createrepo_c >/dev/null &&
     createrepo_c --general-compress-type=gz --basedir /repo /repo'
   ```

   元数据用 gzip 压缩（`--general-compress-type=gz`）保证
   RHEL 7 老 `yum` 兼容。

3. 提交 `Packages/*.rpm`、更新过的 `repodata/`、push。网
   站每天从本分支刷新。

---

## 说明

- 仓库和包目前**未签名**（`gpgcheck=0`、`repo_gpgcheck=0`）。
  GPG key 发布后开启两个 check。
- 构建出处和源码在
  [x-cmd/release](https://github.com/x-cmd/release)。
- 包许可证：Apache-2.0。
- 配套 APT 仓库：[x-cmd/deb](https://github.com/x-cmd/deb) →
  `https://www.x-cmd.com/deb/`。

---

## 文档站

用户文档在 `docs/`，每个主题一个目录，内含 en.md / cn.md / llms.md / faq.yml
四件套（目录名携带主题编号），结构对齐
[x-cmd/cve](https://github.com/x-cmd/cve)：

| 页面 | 内容 |
| --- | --- |
| [`docs/0-install/`](docs/0-install/cn.md) | 怎么装：dnf / yum / zypper 三种 |
| [`docs/1-packaging/`](docs/1-packaging/cn.md) | 怎么自己打包 .rpm + 托管仓库 |
| [`docs/2-security/`](docs/2-security/cn.md) | `gpgcheck=0` 真相 + GPG 签名计划 |
| [`docs/3-stats/`](docs/3-stats/cn.md) | 仓库对外发布什么（版本、大小、checksum） |
