# 修复摘要

## 修复的问题
本次未新增代码改动。经获取未截断的完整 Jenkins 构建日志，确认失败根因是 **getdeps 未能把 openEuler 识别为 rpm 系发行版**（`distro_family=None`），导致 `openssl` 依赖的 `[rpms]` 系统包配置无法匹配而抛出 `KeyError`；对应修复（`fix_getdeps.py` 第 1 步，把 `openeuler` 加入 `rhel` 家族）**已存在于当前 fix 分支**，无需重复修改。

## 修改的文件
- 无（当前 fix 分支已包含全部所需修复，未做任何新增改动）

## 修复逻辑

### 1. 拿到真实报错（补齐分析报告缺失的证据）
分析报告给出置信度「低」，且日志在 `Building lz4...` 处被截断，未能定位根因。我通过 PR #4724 评论中的门禁链接，拉取了未截断的完整门禁日志：
- x86_64：`https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/x86-64/job/openeuler-docker-images/4801/consoleText`
- aarch64：`https://ci.openeuler.openatom.cn/job/multiarch/job/openeuler/job/aarch64/job/openeuler-docker-images/4897/console`

在完整日志（2.7 MB）中定位到真正的失败原因（`raw4801.log:17687-17708`）：

```
Traceback (most recent call last):
  ...
  File "/build/build/fbcode_builder/getdeps/manifest.py", line 668, in _create_fetcher
    raise KeyError(
KeyError: 'project openssl has no fetcher configuration or system packages matching
{distro=****, distro_family=None, distro_vers=24.03, fb=off, fbsource=off, os=linux, test=off}
- have you run `getdeps.py install-system-deps --recursive`?'
```

### 2. 根因定位
`getdeps_platform.py` 中：
- `get_package_manager()` 依据 `distro_family()` 返回包管理器；只有 family 为 `rhel`/`fedora` 才返回 `rpm`（`getdeps_platform.py:318-330`）。
- `v2026.09.28.00` 的 `DISTRO_FAMILIES` 中 `"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky")` **不含 `openeuler`**，因此 `distro_family("openeuler")` 返回 `None` → `get_package_manager()` 返回 `None`。
- 于是 `openssl` manifest 的无条件 `[rpms] openssl openssl-devel openssl-libs` 段不生效，且其 `[download.not(any(os=linux,...))]` 在 Linux 上被排除 → 无可用 fetcher → `KeyError`，整个 `getdeps.py build fbthrift` 以 exit code 1 结束。

这与分析报告的「方向 2：`fix_getdeps.py` 针对上游文件的 patch 未生效」一致：PR 原始 `fix_getdeps.py` 第 1 步替换的字符串 `("fedora", "centos", "centos_stream", "rocky", "alma")` 在 v2026.09.28.00 的 `DISTRO_FAMILIES` 中**不存在**，`str.replace` 静默 no-op，导致修复未生效。

### 3. 已存在的修复（当前分支对应内容）
`Others/fbthrift/2026.09.28.00/24.03-lts-sp4/fix_getdeps.py` 第 1 步已改为匹配新版 `DISTRO_FAMILIES` 的 `rhel` 行并追加 `openeuler`：

```python
c = c.replace(
    '"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),',
    '"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky", "openeuler"),'
)
```

`Others/fbthrift/2026.09.28.00/24.03-lts-sp4/Dockerfile:23` 亦已恢复为与上一可用版本 `2026.09.21.00` 一致的 `build --no-tests fbthrift`。

### 4. 上游验证结果（按要求执行）
已从上游 `facebook/fbthrift` 的 `v2026.09.28.00`（Dockerfile `ARG VERSION` 指定）拉取被 patch 的文件并逐条在本地用 Python 验证：
- `build/fbcode_builder/getdeps/getdeps_platform.py`：目标串 `"rhel": ("rhel", "centos", "centos_stream", "alma", "rocky"),` 精确存在（出现 1 次）；patch 后 `distro_family("openeuler") == "rhel"`，`get_package_manager()` 返回 `rpm`。**正则/字符串替换验证匹配成功**。
- `build/fbcode_builder/getdeps/fetcher.py`：`_verify_hash(self) -> None:` 真实存在，其后紧跟 4 空格缩进的 `def _download_dir`；正则 `def _verify_hash\(self[^)]*\)[^:]*:.*?(?=\n    def )` **匹配成功**，且不会误删后续方法。
- `build/fbcode_builder/manifests/libaio`：上游该版本**已经是** `subdir = libaio-0.3.113`，第 3 步字符串替换为 no-op（无害）。
- 另确认 `getdeps_platform.py`、`fetcher.py`、`manifests/libaio` 在 `v2026.09.21.00` 与 `v2026.09.28.00` 之间内容完全一致。

### 5. 与参考版本对齐情况
当前 fix 分支的工作树内容与 `origin/master`（已合入的参考版本）在 PR 文件上**功能完全一致**（唯一差异是 Dockerfile 末尾换行符，属上游自动生成格式差异，不影响构建，按最小化原则未改动）。

## 潜在风险
- 当前分支不是本次失败时的状态：分析报告所引用的日志（`build fbthrift`，无 `--no-tests`、旧版第 1 步 patch）对应的是 PR head `353f78450`；当前 `fix/4724` 已含修复。若 CI 使用的仍是未更新的 PR head，需要将 fix 分支的改动同步到构建分支后再验证；就本分支内容而言，根因修复已就位。
- 仓库内预置的 `libaio-libaio-0.3.113.tar.gz` 经查为有损 UTF-8 转换而损坏（非 gzip 格式），且 Dockerfile `cp` 的目标文件名 `libaio-libaio-libaio-0.3.113.tar.gz` 与 getdeps `ArchiveFetcher` 实际计算的 `libaio-libaio-0.3.113.tar.gz` 不一致（多一段 `libaio-`），故该预置包实际未被使用，libaio 仍联网下载。该现象在 `2026.09.21.00` 已存在，非本次回归、也非本次失败触发点，按最小化原则未改动；若后续需要完全离线构建，建议另行修正文件名并替换为正确的二进制包。