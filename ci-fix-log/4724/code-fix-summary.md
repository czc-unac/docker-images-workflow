# 修复摘要

## 修复的问题
恢复 fbthrift 构建命令中丢失的 `--no-tests` 参数，使 2026.09.28.00 版本与上一可用版本 2026.09.21.00 的构建方式一致，修复 `getdeps.py build fbthrift` 阶段 exit code 1 的失败。

## 修改的文件
- `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile`: 第 23 行构建命令由 `build fbthrift` 改回 `build --no-tests fbthrift`。

## 修复逻辑
1. **定位回归点**：对比本 PR 新增的 `2026.09.28.00` 与仓库中上一个可用版本 `2026.09.21.00`，两份 `Dockerfile` 的实质差异只有一处（除版本号外）：
   - `2026.09.21.00`（上一版，已合入）：`getdeps.py ... build --no-tests fbthrift`
   - `2026.09.28.00`（本次失败）：`getdeps.py ... build fbthrift`
   本次自动升级疑似基于更早的 `2026.09.14.00` 模板重新生成，丢失了 `2026.09.21.00` 手工加入的 `--no-tests`（`git log -S "no-tests"` 显示该参数正是于 `2026.09.21.00` 升级提交 `8c1193d41` 中引入）。

2. **参数有效性确认**：已从上游 `facebook/fbthrift` 的 `v2026.09.28.00` 拉取 `build/fbcode_builder/getdeps/cmd_base.py`，确认 `--no-tests` 是 `ProjectCmdBase.setup_parser` 中合法参数（`action="store_false", dest="enable_tests"`），其作用是把顶层项目 fbthrift 的 `test` 上下文置为 `off`，从而在 `fbthrift` manifest 中命中 `[cmake.defines.any(os=windows,test=off)] THRIFT_TESTS=OFF`，避免构建 fbthrift 自身测试。该参数在 `2026.09.21.00` 中已实际验证可用。

3. **未改动原因**：CI 报告在 `Building lz4...` 处日志被截断，无法直接看到真实报错行，报告置信度为「低」。在无法补齐完整日志的情况下，采用「与最近一次成功版本保持完全一致」的最小化策略，仅恢复唯一缺失的 `--no-tests`，不额外改动其他内容。

4. **fix_getdeps.py 上游验证（按要求执行）**：已从上游 `v2026.09.28.00` 拉取三个被 patch 的文件并逐条验证（本地 Python 内存测试）：
   - `build/fbcode_builder/getdeps/getdeps_platform.py`：存在 `"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),`，patch #1 的字符串替换**匹配成功**（openEuler 归一化名为 `openeuler`，加入 rhel 系后 `get_package_manager()` 才返回 `rpm`）。
   - `build/fbcode_builder/getdeps/fetcher.py`：`_verify_hash(self) -> None:` 方法真实存在，其后紧跟 4 空格缩进的 `def _download_dir`，正则 `r'def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )'` **匹配成功**并可整体替换为 no-op。
   - `build/fbcode_builder/manifests/libaio`：上游该版本**已经是** `subdir = libaio-0.3.113`，故 patch #3 的字符串替换为空操作（no-op），无害，无需修改。
   结论：`fix_getdeps.py` 在 `v2026.09.28.00` 上功能正确（且与 `2026.09.21.00` 完全相同），不是本次失败根因，因此未做改动。

## 潜在风险
- 该修复基于「与上一可用版本配置对齐」的推断，而非完整构建日志中的确定错误行。若失败真实发生在 `Building lz4...` 之后的依赖构建（而非 fbthrift 测试构建），则本次改动可能不足以修复；但这是当前证据下唯一且最小、可复现的与成功版本的差异点，风险可控。
- 另注意到 `Dockerfile` 第 21 行预置 libaio tarball 的目标文件名（`libaio-libaio-libaio-0.3.113.tar.gz`）与 getdeps `ArchiveFetcher` 实际计算的文件名（`libaio-libaio-0.3.113.tar.gz`）不一致，因此该预置 tarball 实际未被使用，libaio 仍由 getdeps 联网下载；同时仓库内该 tarball 文件内容疑似经 UTF-8 有损转换而损坏（非 gzip 格式）。该现象在上一成功版本 `2026.09.21.00` 中同样存在，并非本次回归、也不是本次失败的触发点，故按最小化原则未改。若后续需要彻底离线构建，建议另行修正该文件名并替换为正确二进制包。
- `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile` 末尾缺少换行符（与上游自动生成一致），本次未改动该格式问题。