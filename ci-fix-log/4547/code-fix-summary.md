# 修复摘要

## 修复的问题
修复 fbthrift v2026.09.21.00 镜像在 openEuler 上 getdeps 因 `distro_family=None` 对系统包依赖 `openssl` 抛出 `KeyError` 而导致构建失败的问题。

## 修改的文件
- `Others/fbthrift/2026.09.21.00/24.03-lts-sp4/fix_getdeps.py`: 更新第 1 处 openEuler 发行版识别补丁的替换目标字符串，使其适配上游 v2026.09.21.00 重构后的 `DISTRO_FAMILIES` 结构（原替换的旧元组字符串在新版本中已不存在）。

## 修复逻辑

### 实际根因（已从完整 CI 日志定位，非分析报告中的"证据不足"）
分析报告只给了被截断的日志，因此无法定位。我通过 GitCode PR 评论中的 Jenkins 链接补齐了完整日志：

- x86_64: https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/4521/consoleText
- aarch64: https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/4642/consoleText

两份日志的真实错误完全一致（并非 lz4 编译错误，`Building lz4...` 只是并行构建的干扰输出）：

```
Traceback (most recent call last):
  ...
  File "/build/build/fbcode_builder/getdeps/manifest.py", line 668, in _create_fetcher
    raise KeyError(
KeyError: 'project openssl has no fetcher configuration or system packages matching
{distro=openeuler, distro_family=None, distro_vers=24.03, ...}
- have you run `getdeps.py install-system-deps --recursive`?'
```

原因链：
1. 上游 v2026.09.21.00 将发行版识别从 `get_package_manager()` 内的硬编码元组 `("fedora", "centos", "centos_stream", "rocky", "alma")` 重构为模块级 `DISTRO_FAMILIES` 字典 + `distro_family()` 函数。
2. `fix_getdeps.py` 第 1 处仍按旧字符串 `("fedora", "centos", "centos_stream", "rocky", "alma")` 做 `str.replace`，新版本中该字符串已不存在，替换**静默未命中**，导致 `openeuler` 未进入任何发行版家族。
3. 于是 `distro_family("openeuler") = None` → `HostType.get_package_manager()` 返回 `None`。
4. `openssl` manifest 在 Linux 上无 download/git 配置（只靠系统包 `[rpms] openssl openssl-devel openssl-libs`），`_create_fetcher` 里因 `get_package_manager()` 为假而跳过 `SystemPackageFetcher`，最终落到文件末尾 `raise KeyError(...)`，构建失败。

### 修复方式
将第 1 处替换目标改为上游新结构中的 rhel 家族元组，把 `openeuler` 归入 rhel 家族：

```python
c = c.replace(
    '"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky")',
    '"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky", "openeuler")'
)
```

这样 `distro_family("openeuler") = "rhel"`，`get_package_manager()` 返回 `"rpm"`，`openssl` 通过 `SystemPackageFetcher` 命中已由 dnf 安装的 `openssl/openssl-devel/openssl-libs`，与 2026.09.14.00 版本（旧补丁生效时）的行为一致。

### 正则/字符串补丁验证（按要求执行）
已从上游 `facebook/fbthrift` 的 `v2026.09.21.00` tag 拉取并验证（tag 与 Dockerfile 中 `ARG VERSION=v2026.09.21.00` 一致）：

- `build/fbcode_builder/getdeps/getdeps_platform.py`：确认旧字符串不存在、新 `"rhel": (...)` 行存在且唯一；替换后内容**确实发生变化**，`py_compile` 通过。
- `build/fbcode_builder/getdeps/fetcher.py`：`_verify_hash` 实际签名为 `def _verify_hash(self) -> None:`，正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` **匹配成功**，替换后内容变化且 `py_compile` 通过。
- `build/fbcode_builder/manifests/libaio`：上游已是 `subdir = libaio-0.3.113`（补丁目标 `subdir = libaio-libaio-0.3.113` 不存在，属静默 no-op），结果正确，无需改动。

### 功能验证
在拉取的 v2026.09.21.00 源码上实际运行修改后的 `fix_getdeps.py`，并以 `HostType(ostype=linux, distro=openeuler, distrovers=24/24.03)` 复现失败调用路径（`manifests_in_dependency_order()` + `create_fetcher()`，共 31 个依赖）：

- 修复前：`openssl` → `KeyError`（与分析报告/CI 日志一致）。
- 修复后：31 个项目全部成功创建 fetcher，`openssl/libaio/zlib/lz4` 等正常解析，`fails=0`。
- `distro_family("openeuler") = "rhel"`，`get_package_manager() = "rpm"`。
- 复核了整个 fbthrift 依赖闭包（28 个 manifest）中所有 `distro_vers`/`distro_family` 条件：仅涉及 `8/9` 与 ubuntu `18.04~24.04`，openeuler（补丁后 `distro_vers` 由 rhel 规则截断为 `24`）不会误命中，无副作用。

## 潜在风险
- 本改动仅针对 `2026.09.21.00` 版本目录下的 `fix_getdeps.py`（版本固定，Dockerfile 以 `-b v2026.09.21.00` 浅克隆），不影响其它版本。
- 将 openeuler 归入 rhel 家族会使 `parse_os_release()` 按 rhel 规则将 `VERSION_ID` 从 `24.03` 截断为 `24`；已核对依赖闭包内所有相关 manifest 条件，均不会因此误匹配（详见上节）。
- CI 中预置 libaio tarball 的文件名 `libaio-libaio-libaio-0.3.113.tar.gz` 与 getdeps 期望的 `libaio-libaio-0.3.113.tar.gz` 不一致，导致预置未生效、改为在线下载。此为历史遗留（各版本一致）、且在本次 CI 中下载成功，**非本次失败根因**，按最小化原则未改动。