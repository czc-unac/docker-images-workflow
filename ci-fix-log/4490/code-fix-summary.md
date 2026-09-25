# 修复摘要

## 修复的问题
将 langgraph 镜像构建所安装的版本从 PyPI 上不存在的 `4.2.0` 更正为真实存在的 `1.2.12`，并同步更正元数据中的 Tag 与版本描述。

## 修改的文件
- `AI/langgraph/4.2.0/24.03-lts-sp4/Dockerfile`: `ARG VERSION=4.2.0` → `ARG VERSION=1.2.12`，使 `pip install langgraph==${VERSION}` 能解析成功。
- `AI/langgraph/README.md`: 标签行 `4.2.0-oe2403sp4` / `langgraph 4.2.0` → `1.2.12-oe2403sp4` / `langgraph 1.2.12`（链接仍指向现有的 `4.2.0/24.03-lts-sp4/Dockerfile` 路径，未新增/改名文件）。
- `AI/langgraph/doc/image-info.yml`: 同步将标签 `4.2.0-oe2403sp4` 与描述 `langgraph 4.2.0` 更正为 `1.2.12`。
- `AI/langgraph/meta.yml`: 条目键 `4.2.0-oe2403sp4` → `1.2.12-oe2403sp4`，`path` 保持不变（仍指向现有目录）。

## 修复逻辑
CI 报错根因（分析报告方向 1，置信度高）：`langgraph==4.2.0` 在指定 PyPI 索引上不存在，日志中可用版本最高为 `1.2.12`。经查询 PyPI 官方 JSON 确认 `langgraph` 最新版本为 `1.2.12`（`langgraph-cli` 为 0.4.32、`langgraph-sdk` 为 0.4.5，均非 4.2.0），说明自动升级脚本解析出的 `4.2.0` 有误。因此把 Dockerfile 中实际安装的版本改为 `1.2.12`，并同步更新元数据中的 Tag/描述，使镜像内容与元数据一致。

修复方式遵循仓库已有的同类 CI 修复约定（见历史提交 `8e9570a77`，PR #2659 redis 5.4.1 版本不存在的修复）：**保持原目录路径不变，仅更正 Dockerfile 的 `ARG VERSION` 及各元数据文件中的 Tag/版本描述**，因此不新增任何文件、不重命名目录，改动范围严格限定在原始 PR 涉及的 4 个文件内。

## 潜在风险
- 目录名仍为 `4.2.0/`，而镜像 Tag 与内容已更正为 `1.2.12`，目录名与实际版本不一致（与历史修复 `5.4.1` 目录容纳 `8.6.4` 的先例相同）。这是为满足"不新增/重命名文件"约束而采用的最小化改法，不影响构建。
- 自动升级脚本解析出 `4.2.0` 的根因（`image-info.yml` 中 `version_filter: cli;sdk` / `version_scheme: RPM` 可能误匹配上游 `cli==`/`sdk==` 标签）未在本次范围内修改；未来自动升级仍可能解析出错误版本，但不在本次 CI 失败的直接修复范围内。