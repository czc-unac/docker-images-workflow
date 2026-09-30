# 修复摘要

## 修复的问题
`Others/dnsmasq/4.3.1/24.03-lts-sp4/Dockerfile` 中 `ARG VERSION=4.3.1` 指向的 dnsmasq 版本在上游不存在（`dnsmasq-4.3.1.tar.gz` 返回 HTTP 404），导致源码下载失败、CI 构建失败。已将版本修正为上游实际存在的最新版本 `2.93`。

## 修改的文件
- `Others/dnsmasq/4.3.1/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=4.3.1` 改为 `ARG VERSION=2.93`（单行改动，下载 URL `https://thekelleys.org.uk/dnsmasq/dnsmasq-${VERSION}.tar.gz` 及解压目录均随 `${VERSION}` 生效）。

## 修复逻辑

1. **根因确认（对照上游实测，非仅凭 diff 推断）**：分析报告指出日志缺失、置信度低，并明确要求先核实 `dnsmasq-4.3.1` 是否存在。已对上游 thekelleys.org.uk 的发布序列做实际校验：
   - 拉取 `https://thekelleys.org.uk/dnsmasq/` 目录，97 个 `dnsmasq-*.tar.gz` 版本中最大为 `2.93`（`LATEST_IS_2.93`），不存在任何 `4.x` 版本；
   - `curl -I https://thekelleys.org.uk/dnsmasq/dnsmasq-4.3.1.tar.gz` → **HTTP 404**（`text/html` 错误页）；
   - `curl -I https://thekelleys.org.uk/dnsmasq/dnsmasq-2.93.tar.gz` → **HTTP 200**（`application/x-gzip`，933K）。
   这证实了分析报告「模式02：软件包版本不存在」的推断，且失败即发生在 Dockerfile 的 `wget` 步骤。

2. **版本选择**：`2.93` 是 dnsmasq 上游当前实际可用的最新版本（该仓库 `Others/dnsmasq/2.91`、`2.93` 既有镜像也均基于真实版本构建）。`4.3.1` 疑似自动升级流程误取了其它项目的版本号（`Bigdata/kafka/4.3.1` 恰为该版本），并非 dnsmasq 版本。

3. **未同步修改 `meta.yml` / `README.md` / `doc/image-info.yml` 的原因（最小化原则）**：这三个文件中的 tag 必须与目录 `4.3.1/` 对应。`2.93-oe2403sp4` 已作为既有镜像存在于三处：
   - `meta.yml:7-8`（`2.93-oe2403sp4: path: 2.93/24.03-lts-sp4/Dockerfile`）
   - `README.md:22`、`doc/image-info.yml:15`
   若把新增条目 tag 改成 `2.93-oe2403sp4`，会在 `meta.yml` 中产生 YAML 重复键（后写覆盖前值，可能使既有 2.93 镜像指向错误路径），并在 README/image-info 中产生重复行。因此按仓库既有同类修复先例（scann `d36068b`→`1.4.2`、mongoose `7.22`→`7.21`、blat `2.5.1`→`1.1`）的做法，仅修改 Dockerfile，保留 4.3.1 目录与 tag 不动，构建即可通过。

4. **为何不改下载 URL 而改 `VERSION`**：Dockerfile 中的变量 `VERSION` 同时用于下载文件名与解压目录，改一处即整体一致，改动最小、最不易遗漏。

> 说明：本次修复不涉及对第三方/上游源文件的正则 patch，无正则验证要求；版本存在性已通过上述上游 HTTP 校验完成。

## 潜在风险
- 修正后该镜像实际内容为 dnsmasq 2.93，与既有 `2.93-oe2403sp4` 镜像相同（本 PR 本质是把错误版本号误当成新版本的重复升级）。目录名 `4.3.1/` 与 tag `4.3.1-oe2403sp4` 仍保留，`meta.yml`/`README.md`/`image-info.yml` 中该 tag 仍为 4.3.1，存在「tag 名 4.3.1、实际内容 2.93」的语义不一致。这是自动升级 PR 的既有命名，为遵守「只改最小范围、避免重复键」原则本次未扩大改动；若维护者希望彻底消除该重复版本，应另行关闭/回退该升级 PR 并清理 `Others/dnsmasq/4.3.1/` 及三处元数据中的 4.3.1 条目。
- 该改动仅作用于 `Others/dnsmasq/4.3.1/24.03-lts-sp4/Dockerfile`，不影响既有 `2.91`、`2.93` 等镜像；一旦构建成功，CI 的 `check_build` 即可通过。