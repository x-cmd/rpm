---
x-title: 仓库统计与对外发布清单
x-desc: 这个 RPM 仓库此刻对外发布的 RPM 和 repodata 的精确快照——大小、checksum、suite 级 repomd.xml 元数据。
x-sidebar: 统计
x-keywords: RPM, 统计, 包版本, noarch, checksum, repomd.xml, primary.xml, filelists.xml, other.xml, sha256
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '仓库统计与对外发布清单'
      inLanguage: 'zh-Hans'
      about: '仓库元数据、大小、checksum'
---

# 仓库统计与对外发布清单

> 此刻这个仓库里到底有什么——规范化、精确 pin 死的快照。大小和
> checksum 直接从提交的文件读出来，它们变了这一页也跟着变。

## Suite 概览

| 项 | 值 |
| --- | --- |
| Origin | x-cmd |
| Label | x-cmd |
| Suite | （单一扁平） |
| Architectures | `noarch`（一份 RPM 服务所有） |
| Source | x-cmd / x-cmd |
| 元数据压缩 | gzip（`--general-compress-type=gz`） |
| 元数据生成器 | `createrepo_c` |

仓库是**扁平布局**——没有像 Debian 的 `dists/stable/` 子目录。
`Packages/` 和 `repodata/` 都在根目录。

## 包快照

| 字段 | 值 |
| --- | --- |
| 包名 | `x-cmd` |
| Version | `0.9.9` |
| Release | `1` |
| 架构 | `noarch` |
| 维护者 | Li Junhao <l@x-cmd.com> |
| Group | （未设） |
| License | Apache-2.0 |
| URL | https://www.x-cmd.com |
| 文件名 | `Packages/x-cmd_0.9.9_all.rpm` |
| 大小 | 4,363,684 字节（~4.16 MB） |
| Source RPM | （无——预编译 binary release） |

包是 `noarch`——一份 RPM 服务所有 RPM 支持架构（x86_64、
aarch64、ppc64le、s390x 等）。

## 产物（`Packages/`）

| 文件 | 大小 |
| --- | --- |
| `Packages/x-cmd_0.9.9_all.rpm` | 4,363,684 |

一份 RPM，对所有架构镜像。

## 元数据（`repodata/`）

每个文件的 SHA-256 都由 `repomd.xml` 引用（suite 级 manifest）。
单位是字节。

| 文件 | 大小 |
| --- | --- |
| `repodata/repomd.xml` | 1,546 |
| `repodata/b7faf596ebb1c2510ba2cf8082b84e0ec1dc4b1a77fa2d15047a72a3b2c29c50-primary.xml.gz` | 717 |
| `repodata/497dcdc88383d492c7f7733e5b30cd810b77d212e55a85fdec7b9a7be4a275d2-filelists.xml.gz` | 261 |
| `repodata/75c04b581370f1d3fb262dec445aafa3073db147fa3113759592ed91274ca213-other.xml.gz` | 320 |

每个文件的作用：
- `repomd.xml` —— 入口，引用所有元数据文件 + SHA-256 + 时间戳
- `primary.xml.gz` —— 包元数据（Name、Version、Requires、
  size、location 等）
- `filelists.xml.gz` —— 每个包的文件清单
- `other.xml.gz` —— changelogs、依赖解析等

## 仓库元数据（`repomd.xml` 节选）

```xml
<repomd>
  <revision>...</revision>
  <data type="primary">
    <checksum type="sha256">b7faf596ebb1c2510ba2cf8082b84e0ec1dc4b1a77fa2d15047a72a3b2c29c50</checksum>
    <open-checksum type="sha256">...</open-checksum>
    <location href="repodata/b7faf596...-primary.xml.gz"/>
    <timestamp>...</timestamp>
    <size>717</size>
    <open-size>...</open-size>
  </data>
  <data type="filelists">...</data>
  <data type="other">...</data>
</repomd>
```

## 人气（代理指标）

本仓库本身不统计下载量——纯静态镜像，没分析层。要看代理
人气，看上游 x-cmd 项目：

| 渠道 | URL |
| --- | --- |
| GitHub stars | <https://github.com/x-cmd/x-cmd> |
| GitHub releases | <https://github.com/x-cmd/release> |
| 配套 APT 仓库 | <https://github.com/x-cmd/deb> |
| 官网 | <https://www.x-cmd.com> |

## 更新节奏

- 仓库在 x-cmd 上游每次发版时发布
- CDN 镜像每天从本 GitHub 仓库同步
- 没有固定周期的 CI rebuild——没有 release 就没必要重建

## 怎么直接拉原始数据

```sh
# Suite 元数据
curl -fsSL https://www.x-cmd.com/rpm/repodata/repomd.xml

# 包元数据（gzip 压缩）
curl -fsSL https://www.x-cmd.com/rpm/repodata/primary.xml.gz | gunzip

# 包本身
curl -fsSLO https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
```

三个文件都从和首页同一个 CDN 服务出来。

## 跟 Debian 兄弟的区别

| | Debian（`x-cmd/deb`） | RPM（`x-cmd/rpm`） |
| --- | --- | --- |
| 布局 | `pool/` + `dists/stable/{Release,main/binary-*/Packages*}` | 扁平 `Packages/` + `repodata/` |
| Components / Suites | 多 Component Suite | 单一扁平 repo |
| per-arch 索引 | 是（5 个，相同内容） | 否（`noarch` 一份够） |
| 元数据生成器 | `dpkg-scanpackages` + `apt-ftparchive` | `createrepo_c` |
| 签名 | `[trusted=yes]`（目前） | `gpgcheck=0` + `repo_gpgcheck=0`（目前） |
| 客户端 | `apt` | `dnf` / `yum` / `zypper` |

## 下一步

- 想看这些文件怎么生成？看 [打包](../1-packaging/cn.md)
- 想看安全模型今天承诺什么？看 [安全](../2-security/cn.md)

数据源：
[`repodata/repomd.xml`](../../repodata/repomd.xml)、
[`repodata/primary.xml.gz`](../../repodata/b7faf596ebb1c2510ba2cf8082b84e0ec1dc4b1a77fa2d15047a72a3b2c29c50-primary.xml.gz)、
[`Packages/x-cmd_0.9.9_all.rpm`](../../Packages/x-cmd_0.9.9_all.rpm)
截至 `git log` 锁定的提交。
