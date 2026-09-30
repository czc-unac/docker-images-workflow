# 修复摘要

## 修复的问题
fbthrift `2026.09.28.00` 镜像在 `getdeps.py ... build fbthrift` 阶段失败（两架构均 `exit code: 1`），
真正根因是 openEuler 发行版未被 getdeps 识别（`distro_family=None`），导致 `openssl` 依赖既无法命中系统包、
也无法命中下载/构建 fetcher，抛出 `KeyError`。

## 修改的文件
- `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile`：补齐文件末尾换行，使该文件与已通过 CI 的上游
  合入版本（`feat: add fbthrift 2026.09.28.00`，commit `f8b277fe3`）逐字节一致。
- `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/fix_getdeps.py`：（本次未改动）其发行版识别 patch 已在本
  fix 分支中存在，本次仅做上游验证与确认。

> 说明：本次会话开始时，fix 分支已包含前序修复（发行版识别 patch + `--no-tests`）。
> 经与线上真实日志、上游 `v2026.09.28.00` 源码以及已合入且 CI 成功的版本逐项比对，
> 该修复即为正确根因修复；本次唯一剩余的差异是 Dockerfile 末尾缺少换行，已补齐以与验证过的版本一致。

## 修复逻辑
1. **真实根因（来自完整 Jenkins 日志，非分析报告中的截断片段）**
   从 PR #4724 评论中的真实构建 job（x86-64 #4801 / aarch64 #4897）拉取 `consoleText`，在日志尾部
   （被 `[output clipped, log limit 2MiB reached]` 截断之后）找到首条真实错误：
   ```
   File ".../getdeps/manifest.py", line 668, in _create_fetcher
       raise KeyError(
   KeyError: 'project openssl has no fetcher configuration or system packages matching
   {distro=****, distro_family=None, distro_vers=24.03, fb=off, fbsource=off, os=linux, test=off} ...'
   ```
   其中 `distro=****` 是 Jenkins 对 `openeuler` 一词的脱敏（日志中 `repo.****.org` 同理）。
   即：openEuler 的归一化发行版名 `openeuler` 未出现在上游 `getdeps_platform.py` 的
   `DISTRO_FAMILIES` 中，`distro_family` 为 `None` → `HostType.get_package_manager()` 返回 `None` →
   `manifest.py:_create_fetcher` 跳过系统包分支；而 `openssl` manifest 在 Linux 上禁用了下载
   （`[download.not(any(os=linux...))]`）且 builder 为 `nop`，`repo_url`/`url` 均为 `None`，
   因此直接抛出 `KeyError`，getdeps 以退出码 1 结束。

2. **修复**
   - `fix_getdeps.py` 第 1 项将上游的
     `"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),`
     替换为包含 `"openeuler"` 的元组，使 `distro_family("openeuler") == "rhel"`、
     `get_package_manager()` 返回 `rpm`，`openssl`（`[rpms] = openssl openssl-devel openssl-libs`）
     由 Dockerfile 已安装的系统 RPM 满足，不再落到无 fetcher 的分支。
   - `Dockerfile` 使用 `build --no-tests fbthrift`，与上一可用版本 `2026.09.21.00` 及已合入版本保持一致。

3. **上游验证（按要求执行）**
   - 依据 Dockerfile `ARG VERSION=v2026.09.28.00`，从 `facebook/fbthrift` 该 tag 拉取并校验：
     - `build/fbcode_builder/getdeps/getdeps_platform.py`：`"rhel": (...)` 字符串存在，patch #1
       实测匹配成功；
     - `build/fbcode_builder/getdeps/fetcher.py`：`_verify_hash` 正则
       `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` 实测匹配成功，替换后模块可正常编译；
     - `build/fbcode_builder/manifests/libaio`：上游已是 `subdir = libaio-0.3.113`，patch #3 为
       安全的 no-op（预置 tarball 因文件名不匹配未被使用，日志显示 libaio 从 pagure 正常下载并编译）。
   - 本地模拟 `openEuler` 的 `/etc/os-release`，确认打补丁后 `parse_os_release` 返回
     `('linux', 'openeuler', '24')`、`distro_family == 'rhel'`、`get_package_manager() == 'rpm'`。
   - 交叉验证：内容与本次修复一致的手工 PR #4713（head `f8b277fe3`）的 x86_64/aarch64
     `check_build` 均为 SUCCESS，证明该修复在真实 CI 中可通过。

## 潜在风险
无。改动仅补齐 Dockerfile 末尾换行，Docker 行为不变；核心的发行版识别 patch 已通过上游源码验证，
且与已合入并通过双架构 CI 的版本逐字节一致。