# CI 失败分析报告

## 基本信息
- PR: #4555 — 【自动升级】scann容器镜像升级至d36068b版本.
- 失败类型: dependency-error
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: PyPI无此版本
- 新模式症状关键词: Could not find a version, No matching distribution found, pip install, scann==, commit hash

## 根因分析

### 直接错误
```
#9 [4/4] RUN /opt/python39/bin/pip3 install --no-cache-dir -i https://pypi.tuna.tsinghua.edu.cn/simple scann==d36068b
#9 1.063 ERROR: Could not find a version that satisfies the requirement scann==d36068b
       (from versions: 1.2.2, 1.2.3, 1.2.4, 1.2.5, 1.2.6, 1.2.7, 1.2.8, 1.2.9, 1.2.10, 1.3.0, 1.3.1, 1.3.2, 1.3.3, 1.3.4, 1.3.5, 1.4.0, 1.4.1, 1.4.2)
#9 1.063 ERROR: No matching distribution found for scann==d36068b
#9 ERROR: process "/bin/sh -c /opt/python39/bin/pip3 install ... scann==${VERSION}" did not complete successfully: exit code: 1
ERROR: failed to solve: process "... scann==${VERSION}" did not complete successfully: exit code: 1
Dockerfile:21
```

### 根因定位
- 失败位置: `Others/scann/d36068b/24.03-lts-sp4/Dockerfile:21`（`RUN /opt/python39/bin/pip3 install ... scann==${VERSION}`）
- 失败原因: `ARG VERSION=d36068b` 是一个 **上游 git commit 短哈希**，但该值被直接用作 **PyPI 包版本号**（`scann==d36068b`）。PyPI 上 scann 只发布了数字版本（最高 `1.4.2`），不存在 `d36068b` 这个发行版本，因此 pip 解析失败，Docker 构建在第 21 行终止。

### 与 PR 变更的关联
- 本 PR 新增 `Others/scann/d36068b/24.03-lts-sp4/Dockerfile`，其中 `ARG VERSION=d36068b` 且安装命令为 `pip3 install scann==${VERSION}`（Dockerfile:19-21）。该写法直接导致 `scann==d36068b` 在 PyPI 无法命中。
- 同 PR 的 `meta.yml`、`README.md`、`doc/image-info.yml` 只是登记/文档变更，不构成失败原因。
- 结论：失败**由本 PR 新增的 Dockerfile 直接触发**，与基础设施无关。

### 次要问题（同一 Dockerfile，非本次失败的直接原因）
- Python 3.9.19 编译阶段报 `fatal error: ffi.h: No such file or directory`，导致 `_ctypes` 模块未编译成功（`Failed to build these modules: _ctypes`）。原因是 `dnf install` 未包含 `libffi-devel`。该问题被 Python 安装流程视为可选模块而"吞掉"，构建继续进入 pip 阶段，因此**不是**本次失败的第一现场；但若 scann/tensorflow 运行时依赖 `_ctypes`，则构成潜在隐患，应一并确认（属知识库模式10"缺少构建依赖 -devel"范畴）。

## 修复方向

### 方向 1（置信度: 高）
`d36068b` 是上游 `google-research/google-research` 仓库的 commit 哈希，并非 PyPI 发行版本。应在 Dockerfile 中改为**从源码/git 安装**该 commit 的 scann，而不是从 PyPI 用 `scann==${VERSION}` 安装；或退回到 PyPI 实际存在的版本（如 `1.4.2`）。选择哪种取决于本次"升级到 d36068b"的意图：若要保留该 commit，需用 git clone/`pip install git+...@${VERSION}` 方式获取对应源码。

### 方向 2（可选）
若确认应基于 PyPI 安装，则必须保证 `${VERSION}` 取自 PyPI 已有版本号（当前上限 `1.4.2`），并同步修正 `meta.yml` / `image-info.yml` / `README.md` 中登记的 tag 与 `doc/image-info.yml` 的版本校验配置（`version_scheme` 等），使自动升级逻辑不再把 commit 哈希当作版本号下发。

### 方向 3（可选，次要）
在首个 `dnf install` 中补充 `libffi-devel`，确保 Python `_ctypes` 模块可正常编译（`ffi.h` 缺失问题）。

## 需要进一步确认的点
1. 需确认"scann d36068b"在预期发布形态中究竟对应 **PyPI 发行版**还是 **上游 git commit**：从日志看 PyPI 明确无 `d36068b`，倾向后者。
2. 需确认 scann 的正确安装来源与命令（是否使用 `pip install git+https://github.com/google-research/google-research.git@${VERSION}#subdirectory=scann` 之类的形式），这属于代码库/上游约定，日志未提供。
3. 需确认 `d36068b` 对应的 scann 是否要求特定 Python/TensorFlow 版本（Dockerfile 用 Python 3.9.19），避免修复版本来源后出现新的依赖冲突。
4. 需确认 `_ctypes` 缺失是否会影响 scann 运行；若会，应一并纳入修复。
5. 该 PR 的自动升级流程（`version_scheme: RPM`、`version_filter` 等）为何产出一个 commit 哈希作为版本号，是否为上游元数据配置问题。

## 修复验证要求
本次修复不涉及"修改正则 patch 外部源文件"，无额外验证要求。但 code-fixer 在提交前应确认所选安装来源确实包含 `d36068b` 对应内容（如从上游仓库核实该 commit/tag 存在且路径正确），不能仅凭假设改动。
