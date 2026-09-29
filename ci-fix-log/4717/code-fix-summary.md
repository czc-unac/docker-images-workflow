# 修复摘要

## 修复的问题
`Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile` 中构建版本被写成了 `11.1.1`，导致 11.1.2 镜像实际下载/编译的是 11.1.1 源码，与目录、Tag、元数据及 PR 升级目标不一致。

## 修改的文件
- `Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=11.1.1` 恢复为 `ARG VERSION=11.1.2`（单行改动）。

## 修复逻辑
分析报告结论为 `infra-error（证据不足）`，未给出可直接定位的日志。但通过代码库静态核对发现了一个可确定性验证的真实缺陷：

1. **当前分支状态被人为改坏**：`git show pr-head:Cloud/qemu/11.1.2/24.03-lts-sp4/Dockerfile` 显示原始 PR 正确写的是 `ARG VERSION=11.1.2`；而本 fix 分支上已有的自动化修复提交 `ee6921ebc`（"fix(ci): 修复摘要"）把它改成了 `ARG VERSION=11.1.1`。这是一次引入回归的错误修复，必须回退。
2. **全仓库一致性**：`Cloud/qemu/*/` 下所有 Dockerfile 的 `ARG VERSION` 均与所在版本目录一致（10.0.2→10.0.2 … 11.1.1→11.1.1），只有 `11.1.2/` 目录出现 `11.1.1`，明显异常。
3. **元数据一致**：`README.md:21`、`doc/image-info.yml:14`、`meta.yml:22-23` 均声明 11.1.2，PR 标题也是"升级至 11.1.2 版本"。
4. **排除下载 404**：实际对上游做 HTTP 校验，`https://download.qemu.org/qemu-11.1.2.tar.xz` 返回 `200`（`qemu-11.1.1.tar.xz`、`qemu-11.0.0.tar.xz` 同样为 200），因此并非模式02的"版本不存在"，无需降级。

因此，将 Dockerfile 的 `VERSION` 恢复为 `11.1.2` 是使该 PR 自洽且能真正构建出 11.1.2 镜像的最小必要修复。改动范围严格限定在原始 PR 文件列表内。

## 潜在风险
无。仅恢复单个 ARG 默认值，使 Dockerfile 与目录/README/image-info.yml/meta.yml/PR 目标一致；不影响 qemu 11.1.1 及更早版本目录。若后续拿到真实失败日志且证明 11.1.2 在上游确不可用，应另行按模式02处理，本次未改动其他文件。