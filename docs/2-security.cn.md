---
x-title: 安全、签名与 gpgcheck=0 的真相
x-desc: 这个 RPM 仓库今天为什么没签名，`gpgcheck=0` 和 `repo_gpgcheck=0` 实际打开了什么，如何手动校验 digest，以及加 GPG 签名开启两个 check 的计划。
x-sidebar: 安全
x-keywords: RPM, 签名, GPG, gpgcheck, repo_gpgcheck, sha256, rpm -Kv, RPM-GPG-KEY, 供应链, dnf config-manager
x-json-ld:
  '@context': https://schema.org
  '@graph':
    - '@type': TechArticle
      headline: '安全、签名与 gpgcheck=0 的真相'
      inLanguage: 'zh-Hans'
      about: 'RPM 仓库安全、签名与校验'
---

# 安全、签名与 `gpgcheck=0` 的真相

> 把这个仓库今天承诺什么、不承诺什么，完整、诚实地写出来。
> 你是安全审计员或者谨慎的运维，看这一页。

本仓库目前**未签名**。`x-cmd.repo` 声明 `gpgcheck=0` 和
`repo_gpgcheck=0`——显式 opt-in 承认这件事。下面有加真实 GPG
签名、开启两个 check 的计划。

## 今天的现状

| 属性 | 状态 |
| --- | --- |
| CDN TLS | ✅ (https://www.x-cmd.com/rpm/) |
| `primary.xml` 里每个包的 SHA-256 digest | ✅ |
| Suite 级 `repomd.xml`，含所有元数据文件的 checksum | ✅ |
| `repomd.xml.asc` detached 签名 | ❌ |
| `repodata/repomd.xml` clear-signed inline | ❌ |
| RPM header 签名（每个 `.rpm` 内置） | ❌ |
| 公钥发布到 `RPM-GPG-KEY-x-cmd` | ❌ |
| 带过期时间的 key + 轮换策略 | ❌（还没有 key） |

前三行是你现在就能依靠的完整性层。后四行就是 `gpgcheck=0` 和
`repo_gpgcheck=0` 静默关掉的检查。

## `gpgcheck=0` 和 `repo_gpgcheck=0` 实际打开的是什么

两个 flag 是独立的：

| Flag | 跳过什么 |
| --- | --- |
| **`gpgcheck=0`** | 每个 `.rpm` header 内的 GPG/PGP 签名（rpm 自带的） |
| **`repo_gpgcheck=0`** | `repomd.xml` 的签名（suite 级 manifest） |

两个加起来挡两种风险：

1. **网络 MITM**：TLS-strip 攻击者改写你要下的 `.rpm`
2. **仓库被攻陷**：操作员（或拿到操作员账号的人）推了你
   不想装的 `.rpm`

`gpgcheck=0` + `repo_gpgcheck=0` 对**这一个源**把两种都关掉。
`https://` URL 仍给你 TLS，所以 MITM 这条挡住了。操作员信任
这条就是你只能靠"操作员名声"——和从 PyPI 装包不校验 hash、
`curl | bash` 一个 README 是同一类信任模型。

对 x-cmd 具体说：仓库在 GitHub（你能审计每个 commit），通过
标准只读 pipeline 镜像到 `www.x-cmd.com`，`.rpm` 由和公开
x-cmd release artifact 同一份源确定性地打出。你可以把
SHA-256 跟 GitHub release 上的 checksum 文件对比——这就是今天
实际的安全网。

## 不加仓库也能校验

```sh
URL="https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm"
EXPECTED="<从 primary.xml 拿 sha256>"

curl -fsSLO "$URL"
echo "$EXPECTED  x-cmd_0.9.9_all.rpm" | sha256sum -c -
```

`OK` 表示你抓下来的字节和 `repodata/<sha>-primary.xml.gz`
声称的一致。把期望 hash 通过另一个渠道（GitHub release 等）
交叉对比一次就完整可信。

更严谨：抓 `repodata/repomd.xml`，跟里面的 SHA-256 链式校验
每个元数据文件、再校验包文件。

或者直接 `rpm`：

```sh
rpm -Kv https://www.x-cmd.com/rpm/Packages/x-cmd_0.9.9_all.rpm
# 今天：只显示 digest + payload 校验，无签名
# 签名上线后：还会显示 "V3 RSA/SHA256 Signature, key ID ..."
```

## 签名计划

操作员发布 GPG key 后，会做这些事：

1. Key 发布到稳定 URL
   （`https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd`）
2. 用户导入到 RPM db：
   ```sh
   rpm --import https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```
3. CI 给每个 `.rpm` 签 `rpm --addsign`，给 `repomd.xml` 签
   `gpg --detach-sign`（产出 `repomd.xml.asc`）
4. `x-cmd.repo` 开启两个 flag：
   ```ini
   [x-cmd]
   name=x-cmd - One-line description
   baseurl=https://www.x-cmd.com/rpm/
   enabled=1
   gpgcheck=1
   repo_gpgcheck=1
   gpgkey=https://www.x-cmd.com/rpm/RPM-GPG-KEY-x-cmd
   ```
5. dnf / yum / zypper 一旦签名缺失、无效、或者不是预期的
   key，就拒绝从这个源安装——和任何其它签名 RPM 源同等级别的
   保护。

这些工作还没发布之前，**不要**在你审计不到的生产集群里盲信
这个仓库。在开发机用、或者把版本 pin 死。

## 今天我们**挡不住**的供应链风险

- **`x-cmd/rpm` 账号被拿下**：被黑的 GitHub token 可以推一
  个恶意 `.rpm`。缓解：给仓库加 GPG 签名后，任何 push 都
  需要操作员的签名 key，而不只是 commit 权限。
- **镜像被攻陷**：`www.x-cmd.com/rpm/` 是静态镜像，如果
  在 TLS 层之前被替换，你拿到的是攻击者的字节。缓解：同
  样的 GPG 签名能挡住。
- **重放 / 降级**：TLS-MITM 攻击者可以发一个更老、有已知
  漏洞的 `.rpm` 给你。缓解：dnf 对 `repomd.xml` 的日期检查 +
  GPG 签名一起挡住重放/降级。

## 源出处

`.rpm` 由上游 x-cmd release 流水线
([`x-cmd/release`](https://github.com/x-cmd/release)) 打出，
逐字镜像到本仓库。构建的可复现性跟上游流水线一致——对
pin 了 commit SHA 的纯 shell 内容来说，一般都是 yes。

## 下一步

- 想看现在对外发布什么？看 [统计](./3-stats.cn.md)。
- 想自己审计包元数据？看 [打包](./1-packaging.cn.md) 里
  的索引布局。

数据源：本仓库的 [`README.cn.md`](../../README.cn.md) 及
上游 RPM 签名文档。
