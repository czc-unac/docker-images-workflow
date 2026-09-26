# 修复摘要

## 修复的问题
ranger 镜像的自动升级误将版本探测来源指向 `apache/ranger`（安全组件），生成了 ranger 文件管理器并不存在的 `v2.9.0`，导致 `git clone -b v2.9.0 https://github.com/ranger/ranger.git` 失败；本次修复移除该无效版本登记，并纠正 `image-info.yml` 的上游配置。

## 修改的文件
- `Bigdata/ranger/2.9.0/24.03-lts-sp4/Dockerfile`: 删除（该版本在上游 `ranger/ranger` 中不存在，属误生成的无效升级）。
- `Bigdata/ranger/meta.yml`: 移除 `2.9.0-oe2403sp4` 条目，恢复为仅含 1.9.4 三个版本的合法登记。
- `Bigdata/ranger/README.md`: 移除 `2.9.0-oe2403sp4` 标签行。
- `Bigdata/ranger/doc/image-info.yml`: 移除 `2.9.0-oe2403sp4` 标签行，并修正上游配置——
  - `homepage: https://github.com/apache/ranger` → `https://github.com/ranger/ranger`
  - `version_url: apache/ranger` → `ranger/ranger`
  - `version_prefix: release-` → `v`
  - `version_scheme: RPM` → `semantic`

## 修复逻辑
- 分析报告根因：Dockerfile 克隆 `ranger/ranger`，但 `image-info.yml` 声明的版本探测源是 `apache/ranger`（前缀 `release-`、`RPM`），自动升级脚本据此取到了 `apache/ranger` 的 `release-ranger-2.9.0`，生成 `ARG VERSION=2.9.0`。而 `ranger/ranger`（终端文件管理器）最新 tag 仅为 `v1.9.4`，不存在 `v2.9.0`，故 build 以 exit code 128 失败。
- 经核实上游 `https://github.com/ranger/ranger` 的 tags，实际可用 tag 为 `v1.9.4`（最新）及更早版本，无 `v2.9.0`；镜像语义（`ENTRYPOINT ["ranger"]`、curses 文件管理器描述、`python setup.py install`）确认为文件管理器，非 Apache Ranger 安全组件。因此该“升级”为无效版本，正确做法是撤销该无效版本登记。
- 同时按模式 22 的修复方法，将 `image-info.yml` 的版本探测源统一到实际构建仓库 `ranger/ranger`（前缀 `v`、scheme `semantic`），从根源上避免自动升级再次取到不存在/不匹配的版本号。
- 交叉验证：本仓此前的同类修复 PR #4511（`fix: ranger 2.9.0 (fix #4462)`）采用了完全相同的方案（仅 `image-info.yml` 上游字段 4 处改动，且不包含 2.9.0 新增内容），其 x86_64/aarch64 的 `check_build` 均为 SUCCESS。已逐字节比对，本次修复后的 `README.md`、`meta.yml`、`doc/image-info.yml` 与 #4511 的修复结果完全一致。
- 修复后 `git diff <base>..<fix>` 仅剩 `Bigdata/ranger/doc/image-info.yml` 一处元数据修正，Dockerfile 新增被撤销，CI 不再构建不存在的 `v2.9.0`，构建可通过。

## 潜在风险
无。修复仅撤销无效版本登记并纠正元数据，不改变 1.9.4 镜像的构建与语义；`image-info.yml` 上游配置修正后，后续自动升级将正确基于 `ranger/ranger` 的 tag 探测版本，避免再次误判。