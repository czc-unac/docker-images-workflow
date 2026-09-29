# 修复摘要

## 修复的问题
修复 `fix_getdeps.py` 中“openEuler 发行版识别”补丁未命中上游新版脚本、导致 getdeps 无法识别 openEuler 发行版家族（进而无法使用系统包）的构建失败。

## 修改的文件
- `Others/fbthrift/2026.09.28.00/24.03-lts-sp4/fix_getdeps.py`: 将补丁 1 的目标字符串从旧的元组格式 `'("fedora", "centos", "centos_stream", "rocky", "alma")'` 改为新版 getdeps 的 `DISTRO_FAMILIES` 字典格式 `'"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),'`，使替换真正命中并把 `"openeuler"` 加入 rhel 家族。

## 修复逻辑
失败类型为 `build-error`，失败发生在 `python3 /tmp/fix_getdeps.py && getdeps.py ... build fbthrift` 这一步（Dockerfile:18-23），分析报告“方向 2”指出需针对 `v2026.09.28.00` 的上游源码重新校准三处修补。

已按分析报告“修复验证要求”，通过 WebFetch/curl 拉取上游 fbthrift tag `v2026.09.28.00`（Dockerfile `ARG VERSION=v2026.09.28.00`）的对应文件核对：

1. `build/fbcode_builder/getdeps/getdeps_platform.py`：上游已重构为 `DISTRO_FAMILIES: dict[str, tuple[str, ...]]`，其中为 `"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),`。原补丁搜索的旧字符串 `("fedora", "centos", "centos_stream", "rocky", "alma")` **已不存在**，`str.replace` 静默失败，`"openeuler"` 未被加入 rhel 家族。实测修复前文件无任何差异；修复后差异为在第 27 行 rhel 元组追加 `"openeuler"`，且调用 `distro_family("openeuler")` 返回 `rhel`，`get_package_manager()` 恢复为 `rpm`。
   - 这正是 2026.09.21.00 版本（人工修复版）所用的写法，本次自动升级回退成了 09.14/06.22 的旧模板，故按前一可用版本恢复。
2. `build/fbcode_builder/getdeps/fetcher.py`：`_verify_hash(self) -> None:` 位于类内，其后仍有 `\n    def `（`_download_dir`），正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )`（DOTALL）**匹配成功**，替换后方法体为 `pass` 且缩进正确，`ast.parse` 通过，文件仍可被 Python 正常解析/导入。无需修改。
3. `build/fbcode_builder/manifests/libaio`：上游 `[build] subdir = libaio-0.3.113` **已是目标值**，补丁 3 为无害空操作。无需修改。

验证方式：已在本地构造与构建树一致的目录结构（`build/fbcode_builder/getdeps/`、`build/fbcode_builder/manifests/`），放入上游 `v2026.09.28.00` 的真实文件，实际运行修改后的 `fix_getdeps.py`，确认补丁 1 产生预期差异、补丁 2 命中且两文件均可被 `ast.parse` 解析。

（未改动 `--no-tests`：`build` 子命令不会运行测试，且前一可用版本 2026.09.14.00 同样未使用该参数，测试失败证据不足，遵循最小化原则不改。未改动预置 `libaio` tarball：该文件在 09.14/09.21/09.28 各版本 MD5 完全一致，为历史既有状态，并非本次 PR 新引入。）

## 潜在风险
- 修复仅使补丁 1 生效，作用是让 getdeps 将 openEuler 识别为 rhel 家族并启用系统包（rpm）。若 CI 失败的真实报错实际位于依赖源码编译阶段（日志被截断，未能取得），则本次修复可能不足以完全解决；但该修补是相对前一可用版本（2026.09.21.00）的确定性回归修复，且无其他上游侧证据可依。
- 修改仅限 `pr.changed_files` 中的 `fix_getdeps.py`，不影响其他镜像构建。