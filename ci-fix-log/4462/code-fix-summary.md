# 修复摘要

## 修复的问题
撤销了基于错误上游元数据生成的、上游并不存在的 ranger 2.9.0 镜像，并修正 `image-info.yml` 的上游信息，避免自动升级再次生成不存在的版本号。

## 修改的文件
- `Bigdata/ranger/2.9.0/24.03-lts-sp4/Dockerfile`: 删除该无效版本 Dockerfile（`ARG VERSION=2.9.0` 在上游 `ranger/ranger` 不存在对应 tag，构建必然失败）。
- `Bigdata/ranger/meta.yml`: 移除 `2.9.0-oe2403sp4` 条目（恢复为仅含 1.9.4 的三个条目）。
- `Bigdata/ranger/README.md`: 移除 2.9.0 的镜像标签表格行。
- `Bigdata/ranger/doc/image-info.yml`: 移除 2.9.0 的标签表格行；并将上游信息由 Apache Ranger 修正为真正的上游：`homepage: https://github.com/ranger/ranger`、`version_url: ranger/ranger`、`version_prefix: v`、`version_scheme: semantic`。

## 修复逻辑
分析报告根因：Dockerfile 实际打包的是 **ranger 终端文件管理器**（`git clone https://github.com/ranger/ranger.git`、`ENTRYPOINT ["ranger"]`、`python setup.py install`），但 `doc/image-info.yml` 中 `upstream.version_url: apache/ranger`、`version_prefix: release-` 指向了 Apache Ranger，导致自动升级工具按 Apache Ranger 的版本体系生成了 `2.9.0`，而 `ranger/ranger` 上游不存在 `v2.9.0`（构建日志 `fatal: Remote branch v2.9.0 not found in upstream origin`，exit code 128）。

修复采取本仓库处理“无效版本”的既有约定（参见历史提交 `delete tags`、`删除文件 .../5.0.4-tentative/Dockerfile`）：删除无效版本目录并同步清理 `meta.yml`/`README.md`/`image-info.yml` 中的引用；同时修正 `image-info.yml` 的上游元数据，从根因上阻止自动升级再次生成错误版本。

验证结果：
- 通过 GitHub API 确认 `ranger/ranger` 最新 tag 为 `v1.9.4`，且 `git ls-remote https://github.com/ranger/ranger.git refs/tags/v1.9.4` 成功解析（`324ae4b...`），上游确实不存在 `v2.9.0`，因此无法在保留 2.9.0 目录的前提下让构建通过，删除该无效版本是唯一正确选择（若仅把 VERSION 改为 1.9.4 却保留 2.9.0 目录/标签，会产生“标签为 2.9.0、内容为 1.9.4”的错误镜像并重复已有 1.9.4 镜像）。
- `meta.yml`、`image-info.yml` 经 PyYAML 解析通过；`grep` 确认 ranger 目录下不再残留 `2.9.0` / `apache/ranger` 引用。
- 修复分支上 `2.9.0-oe2403sp4` 对应条目已从 `meta.yml` 移除，CI 不再对该不存在的上游版本执行构建。

## 潜在风险
无。本次改动仅移除一个本就不存在的版本并纠正上游元数据，未触及 1.9.4 的任何构建逻辑与其余文件；`Bigdata/image-list.yml` 中镜像项（`ranger: ranger`）为目录级映射，不受版本目录增删影响。