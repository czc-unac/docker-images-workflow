# 修复摘要

## 修复的问题
glibc 构建因下载 `glibc-2.42.9000.tar.xz`（开发快照版，上游无此 release tarball）返回 404 而失败；本次确认当前分支上的既有修复（改用正式发布版 `2.43` 并同步元数据标签）已能成功构建，剩余 CI 失败为基础设施（infra）问题，无需再改代码。

## 修改的文件
- 无（本次未新增代码改动）。

说明：当前 `fix/4850` 分支已包含上一轮的最小化修复（commit `2f312eb53`），本次核实其正确性：
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: `ARG VERSION=2.42.9000` → `2.43`
- `Others/glibc/meta.yml`: 标签 `2.42.9000-oe2403sp4` → `2.43-oe2403sp4`
- `Others/glibc/doc/image-info.yml`: 同上标签同步
- `Others/glibc/README.md`: 同上标签同步

## 修复逻辑
1. **原始根因（真实 CI 日志确认，非猜测）**：从原始 PR #4850 的 CI 结果评论中定位失败 job 并下载控制台日志（x86_64 build 4964 / aarch64 build 5060），两个架构均在 `Dockerfile:18` 的 `RUN wget .../gnu/glibc/glibc-${VERSION}.tar.xz` 处失败，报 `HTTP request sent, awaiting response... 404 Not Found`。经 `curl` 实测：`.../glibc-2.42.9000.tar.xz` 在 `mirrors.tuna.tsinghua.edu.cn` 与 `ftp.gnu.org` 均返回 **404**；镜像站仅有正式发布版 `glibc-2.42 / 2.43 / 2.44.tar.xz`。`.9000` 为 glibc 开发周期滚动号，不发布到 GNU release 镜像站，属「模式02：下载 URL / 软件包版本不存在」。
2. **修复方式**：将 `ARG VERSION` 替换为其对应的正式发布版 **2.43**（上游 2.43 于 2026-01-23 发布，`curl -I` 实测 `https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.43.tar.xz` 返回 **200**），目录路径保持不变，并同步 `meta.yml`、`README.md`、`image-info.yml` 三处对外 tag，保证 tag 与实际构建内容一致。
3. **修复有效性（真实 CI 日志确认）**：下载修复 PR #4873 的 CI 日志，x86_64（build 4987）与 aarch64（build 5084）构建日志均出现
   `#9 [builder 4/6] RUN wget .../glibc-2.43.tar.xz` → `HTTP request sent, awaiting response... 200 OK`，且 job 均 `Finished: SUCCESS`。即 2.43 版本在两架构均构建成功。
4. **剩余 `ci_failed` 标签的真实原因（infra-error）**：修复 PR 另一次 aarch64 运行（build 5083）在通过镜像规格检查后，于 `eulerpublisher/update/container/app/update.py` 中调用外部命令时抛出
   `FileNotFoundError: [Errno 2] No such file or directory: 'eulerpublisher'`，`Build step 'Execute shell' marked build as failure`。该错误为 Jenkins agent 缺少 `eulerpublisher` 可执行文件导致的工具链/环境问题，与 glibc Dockerfile 改动无关（同一份代码在 build 5084 上已构建成功）。
5. **结论**：原构建失败已由既有修复解决；当前无需任何代码修改（符合约束中「infra-error 不得强行改代码」的要求）。

## 潜在风险
无。未产生任何代码改动；既有的 2.43 修复已在 x86_64/aarch64 两架构的真实 CI 构建中验证通过（HTTP 200 + SUCCESS）。目录名沿用了自动升级生成的 `2.42.9000`（受 `pr.changed_files` 约束不可重命名/新增文件），对外 tag 已统一为 `2.43-oe2403sp4`，与 Dockerfile 内 `VERSION=2.43` 一致。