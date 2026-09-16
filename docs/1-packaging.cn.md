---
x-title: 自己打包 .rpm 并托管为 yum/dnf 仓库
x-desc: 如何用 shell 项目组装 noarch RPM 包、用 createrepo_c 重建仓库元数据，并按规范的 Packages/ + repodata/ 布局托管。
x-sidebar: 打包
x-keywords: rpm, rpmbuild, createrepo_c, noarch, Fedora, RHEL, SUSE, yum 仓库, spec 文件, dnf config-manager, repodata, repomd.xml
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '自己打包 .rpm 并托管为 yum/dnf 仓库'
      inLanguage: 'zh-Hans'
      about: '打包 RPM 包并托管 yum/dnf 仓库'
---

# 自己打包 .rpm 并托管为 yum/dnf 仓库

> 一条最短路径：把一个目录里的文件打成（签或不签的）`.rpm`，
> 用 `createrepo_c` 重建仓库元数据，然后放到任意静态 CDN 上对
> 外服务。本仓库用的就是这一套。

如果你只想把 shell 脚本、补全、manpage 发给 Fedora / RHEL
/ openSUSE 用户、不想走官方 archive 流程，这条路最短。

## `.rpm` 到底是什么

RPM 文件是 **cpio payload 包在 cpio header 里**，外层再包
**lead + signature + headers + payload + size**：

```
x-cmd_0.9.9_all.rpm =
  lead                       # magic + arch
  signature                  # （可选）RPM 签名
  header                     # Name, Version, Requires, ...
  payload (cpio)             # 实际文件（noarch → /usr/local/x-cmd/...）
  payload-size trailer
```

`noarch` 包的 payload 是 shell / 脚本——一份 RPM 服务所有架构。

## 1. 摆源码树

RPM 从 `BUILDROOT` 编译——一个扁平目录树映射装好之后的系统：

```
rpmbuild/BUILDROOT/
└── usr/local/x-cmd/pkg/
    └── myapp-1.2.3/
        ├── bin/myapp
        ├── share/man/man1/myapp.1
        └── share/myapp/...
```

noarch 包路径不限——`/usr/local/x-cmd/pkg/` 或 `/opt/x-cmd/pkg/` 都行。

## 2. 写 `.spec` 文件

最重要的一个文件。最小可跑版本：

```spec
Name:           myapp
Version:        1.2.3
Release:        1%{?dist}
Summary:        一句话简介
License:        Apache-2.0
URL:            https://example.com/myapp
Source0:        https://example.com/myapp-%{version}.tar.gz

BuildArch:      noarch
Requires:       bash

%description
一句话简介。
多行更长描述（任意行数）。

%install
mkdir -p %{buildroot}/usr/local/x-cmd/pkg/myapp-%{version}/bin
mkdir -p %{buildroot}/usr/local/x-cmd/pkg/myapp-%{version}/share/man/man1
install -m 0755 myapp %{buildroot}/usr/local/x-cmd/pkg/myapp-%{version}/bin/
install -m 0644 myapp.1 %{buildroot}/usr/local/x-cmd/pkg/myapp-%{version}/share/man/man1/

%files
/usr/local/x-cmd/pkg/myapp-%{version}/

%changelog
* Mon Jan 01 2026 Maintainer <you@example.com> - 1.2.3-1
- Initial package.
```

按需加字段：`BuildRequires`、`Recommends`、`Suggests`、
`Requires(post)` 等。完整字段表看 `man rpm.spec`。

## 3. 打 `.rpm`

```sh
rpmbuild -ba myapp.spec \
    --define "_topdir $PWD/rpmbuild" \
    --define "_sourcedir $PWD" \
    --define "_rpmdir $PWD/RPMS"

# 产物：
ls RPMS/noarch/myapp-1.2.3-1.noarch.rpm
```

`rpmbuild` 需要 `rpm-build` 包。

**macOS 用户**——默认没 `rpmbuild`，跑 Fedora 容器：

```sh
docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
  dnf install -y rpm-build >/dev/null
  cd /repo
  rpmbuild -ba myapp.spec \
      --define "_topdir $PWD/rpmbuild" \
      --define "_sourcedir $PWD" \
      --define "_rpmdir $PWD/RPMS"
'
```

## 4. 放进 `Packages/`

yum/dnf 标准布局是扁平的：

```
repo/
├── Packages/
│   └── x-cmd_0.9.9_all.rpm
└── repodata/
```

直接 `.rpm` 放 `Packages/`——没有 `pool/`，没有子目录（那是 Debian 约定）。

## 5. 重建仓库元数据

`createrepo_c` 扫 `Packages/` 生成 dnf / yum / zypper 实际读的元数据：

```sh
docker run --rm -v "$PWD:/repo" fedora:latest bash -lc '
  dnf install -y createrepo_c >/dev/null
  cd /repo
  createrepo_c --general-compress-type=gz --basedir /repo /repo
'

# 产物：
repodata/
├── repomd.xml
├── <sha>-primary.xml.gz     # 包元数据
├── <sha>-filelists.xml.gz   # 文件清单
└── <sha>-other.xml.gz       # changelogs、依赖
```

**重点**：`--general-compress-type=gz` 保持 gzip 压缩——经典
`yum`（RHEL 7）不读 zstd 元数据，选 gzip 是最安全。

## 6. 服务出去

`Packages/` + `repodata/` 都是纯静态文件。任意 HTTP server 都
行——`python3 -m http.server`、GitHub Pages、nginx、CDN。

用户加 `.repo`：

```ini
# /etc/yum.repos.d/x-cmd.repo
[x-cmd]
name=x-cmd - 一句话简介
baseurl=https://example.com/rpm/
enabled=1
gpgcheck=0
```

或 `dnf config-manager --add-repo`。

## 7. 加 `.repo` 片段

仓库根放一份现成 `.repo` 对人类友好：

```ini
[x-cmd]
name=x-cmd - Bootstrap 1000+ command line tools in seconds
baseurl=https://example.com/rpm/
enabled=1
gpgcheck=0
repo_gpgcheck=0
```

本仓库就是这个套路——见 [`x-cmd.repo`](../../x-cmd.repo)。

## 什么时候不要走这条

- **想让包进 Fedora / RHEL / openSUSE 官方**——提 PR 到上游仓库
  或找 maintainer；自托管适合不想走流程的项目
- **有多架构原生二进制**——用 Fedora Koji / openSUSE OBS，rpmbuild
  多架构可行但要 build matrix
- **要可复现构建 / SBOM**——加 `rpmbuild` + CI 矩阵锁 source

## 下一步

- 想给仓库加 GPG 签名？看 [安全](./2-security.cn.md)。
- 想看本仓库对外的标准布局？看 [统计](./3-stats.cn.md)。

数据源：本仓库的 [`README.cn.md`](../../README.cn.md) 及上游
Fedora packaging guide。
