# 修复摘要

## 修复的问题
jetty 自动升级 PR #4897 将镜像指向上游 Maven Central 尚不存在的版本 `12.1.14`，导致构建阶段
下载 `jetty-home-12.1.14.tar.gz` 得到错误页（非 gzip），`tar` 解压失败而构建中断；修复方式为把
Dockerfile 的下载版本改为上游实际存在的最新版 `12.1.13`。

## 修改的文件
- `Others/jetty/12.1.14/24.03-lts-sp4/Dockerfile`: 第 3 行 `ARG VERSION=12.1.14` → `ARG VERSION=12.1.13`。

> 本次复核确认该修复已处于生效状态（工作区 `ARG VERSION=12.1.13`，与 `pr-head` 的差异仅为这一行），
> 属于既有 fix 提交已落地的内容，本次运行未新增代码改动，也未触碰其余 5 个原始 PR 文件。

## 修复逻辑
1. **上游版本核验（已联网验证）**
   - `https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-home/maven-metadata.xml` 的
     `<latest>` / `<release>` 均为 `12.1.13`，版本列表中不存在 `12.1.14`。
   - `.../jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` → `HTTP 404`；
     `.../jetty-home/12.1.13/jetty-home-12.1.13.tar.gz` → `HTTP 200`。
2. **真实 CI 日志确证根因**（弥补分析报告"日志缺失"的空白）
   - 原始 PR 的 aarch64 构建 job（`#5112`）失败输出：
     `curl ... jetty-home/12.1.14/jetty-home-12.1.14.tar.gz` 下载到 `100 554` 字节错误页 →
     `gzip: stdin: not in gzip format` → `tar: Child returned status 1` →
     `sed: can't read etc/jetty.conf` → `ERROR: ... did not complete successfully: exit code: 1`。
     即 `ARG VERSION=12.1.14` 指向了上游不存在的制品，与知识库模式 42 的历史案例 PR #4852 一致。
   - Dockerfile 的下载地址完全由 `ARG VERSION` 驱动，因此将版本改为上游真实存在的最新版 `12.1.13`
     后，下载/解压/启动脚本链路可正常完成（fix PR #4921 的 aarch64 job `#5136` 已 `check_build SUCCESS`）。
3. **改动范围最小**：仅调整版本号一行，未改动构建脚本、元数据注册等其他内容，符合最小化原则。

## 潜在风险
- **标签与内容名义不一致**：目录名、`README.md`、`meta.yml`、`doc/image-info.yml` 仍登记为
  `12.1.14-oe2403sp4`，而镜像内实际为 jetty `12.1.13`。受"只允许修改原始 PR 涉及文件、不得新增/删除文件"
  的约束，无法重命名目录或删除冗余登记；以构建通过为目标的当前修复下，这是可接受的最小改动，
  但需发布侧知悉：建议待 jetty 正式发布 12.1.14 后重新生成升级 PR。
- **fix PR #4921 的 x86_64 失败属 infra-error，非代码问题**：该 job（`#5040`）日志显示构建尚未开始，
  失败发生在 CI 引导阶段下载 `build.sh` 时：`curl: (22) The requested URL returned error: 429`，
  随后 `chmod: cannot access 'build.sh': No such file or directory`、`./build.sh: No such file or directory`。
  同一代码的 aarch64（`#5136`）已构建成功，说明 429 为 CI 侧限流，与 Dockerfile 无关；
  重新触发 CI 或等待限流恢复即可，无需代码修改。
- 其余构建/运行逻辑未改动，不影响已有功能。