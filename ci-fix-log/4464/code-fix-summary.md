# 修复摘要

## 修复的问题
修复 `fix_getdeps.py` 中 openEuler 发行版识别补丁在 fbthrift v2026.09.21.00 上静默失效的问题（上游 `getdeps_platform.py` 重构导致正则/字符串替换不再命中），使 getdeps 重新将 openEuler 识别为 RPM 发行版并启用 `--allow-system-packages`。

## 修改的文件
- `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/fix_getdeps.py`: 在“修复 openEuler 发行版识别”步骤中新增针对新上游结构的补丁，并保留对旧结构的兼容替换。其余步骤（`_verify_hash`、libaio subdir）保持不变。

## 修复逻辑
分析报告置信度低、日志被截断，但通过对比“上一个可用版本”和“本次升级版本”的上游源码，定位到一个**确凿的静默失效**：

1. 上游 `build/fbcode_builder/getdeps/getdeps_platform.py` 在 v2026.09.21.00 已被重构：
   - 旧版（v2026.09.14.00）`get_package_manager()` 中为 `if self.distro in ("fedora", "centos", "centos_stream", "rocky", "alma"): return "rpm"`，原补丁通过 `str.replace` 把 `"openeuler"` 追加进该元组，从而让 openEuler 走 rpm 系统包路径。
   - 新版改为 `DISTRO_FAMILIES` 字典 + `distro_family()`，`get_package_manager()` 通过 `family in ("fedora", "rhel")` 判定 rpm。openEuler 未被任何家族覆盖，`distro_family("openeuler")` 返回 `None`，因此 `get_package_manager()` 返回 `None`。
   - 结果是原补丁的 `str.replace('("fedora", ...)')` 在新版源码中**找不到目标文本，静默返回原文**（已实测 `old tuple present: False`）。
2. `get_package_manager()` 返回 `None` 会让 `SystemPackageFetcher` 认为所有系统依赖都未安装（`packages` 为 `None`），从而**放弃使用 Dockerfile 中 `dnf install` 的 `*-devel` 系统包，改为全部源码构建**（日志中可见 lz4、libevent 等被从源码“Building”，与这一行为吻合），大幅增加构建失败面与耗时。
3. 修复方式：不依赖易变的家族元组/条件字段，直接在 `get_package_manager()` 的固定锚点（`if self.is_darwin(): return "homebrew"` 之后）注入 `if self.distro == "openeuler": return "rpm"`。该改动同时适用于新旧结构：旧结构下该分支与原有追加 `"openeuler"` 结果一致，新结构下则恢复 openEuler 的 rpm 识别。

### 外部源文件正则/替换验证（按要求）
已从上游仓库按 Dockerfile 的 `ARG VERSION=v2026.09.21.00` 获取实际文件并验证（v2026.09.14.00 作为对照）：
- `getdeps_platform.py`（v2026.09.21.00）：`https://raw.githubusercontent.com/facebook/fbthrift/v2026.09.21.00/build/fbcode_builder/getdeps/getdeps_platform.py` —— 实测旧元组 `("fedora", "centos", "centos_stream", "rocky", "alma")` **不存在**，新增的 `is_darwin/homebrew` 锚点存在，注入后可编译通过，且 `HostType(ostype='linux', distro='openeuler', ...).get_package_manager()` 实测返回 `"rpm"`。
- `fetcher.py`（v2026.09.21.00）：实测正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )`（DOTALL）**匹配成功**，其后继方法为 `def _download_dir(self) -> str:`，替换后 `def _verify_hash(self): pass` 可编译通过。
- `manifests/libaio`（v2026.09.21.00）：`subdir = libaio-0.3.113` **已存在**，因此第 3 步 `str.replace('subdir = libaio-libaio-0.3.113', ...)` 为无副作用空操作（保留以兼容旧结构）。
- 验证方式：将 `fix_getdeps.py` 对上述上游文件副本执行后，`python3 -m py_compile` 通过，并检查目标位置确被修改（非静默未匹配）。

## 潜在风险
- 本修复只恢复“openEuler 被识别为 rpm”这一既有行为，未把 openEuler 加入 `DISTRO_FAMILIES` 的 `rhel` 家族，因此新上游中形如 `[rpms.distro_family=rhel]` 的家族条件对 openEuler 仍不生效（与 v2026.09.14.00 旧行为一致）。经核查 fbthrift 依赖链中的 `lz4`/`libmnl` 条件在 openEuler 下两种实现结果相同，无行为差异。
- 分析报告置信度为“低”且原始日志被截断，无法 100% 排除存在其它依赖编译问题。本次提交修复了经上游源码验证确证的一个补丁静默失效根因；若 CI 仍失败，需要提供 `exit code: 1` 之前的完整 getdeps 日志以继续定位。
- `libaio-libaio-0.3.113.tar.gz` 预置包与 Dockerfile 中目标文件名（多一个 `libaio`）不匹配，且该 tarball 文件头非合法 gzip；此为两个版本共有的既有现象，getdeps 实际会自行下载真实 tarball（哈希校验已被 patch 跳过），本次未改动，避免引入新风险。