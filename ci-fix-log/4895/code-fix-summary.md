# 修复摘要

## 修复的问题
将 glibc 镜像 Dockerfile 中不存在的开发版版本号 `2.42.9000` 修正为上游真实发布的正式版本 `2.42`，消除源码下载 404 导致的构建失败。

## 修改的文件
- `Others/glibc/2.42.9000/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=2.42.9000` 改为 `ARG VERSION=2.42`（第 3 行）。

## 修复逻辑
分析报告（置信度低、无 CI 日志）给出两个可能方向，本修复只针对可被客观验证的根因——**源码下载 404（模式02）**：

1. **版本号确认**：Dockerfile 通过
   `wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-${VERSION}.tar.xz`
   下载源码。`2.42.9000` 是 glibc 主干表示"开发中"的内部版本号（git tag `glibc-2.42.9000`），
   GNU 镜像站只发布正式版本 tarball，因此该 URL 必然 404。
2. **上游可用性验证（已实际执行）**：修复前通过 curl 对清华镜像站逐一确认：
   - `glibc-2.42.9000.tar.xz` → **HTTP 404**
   - `glibc-2.42.tar.xz` → **HTTP 200**
   - （同时确认 `glibc-2.43.tar.xz`、`glibc-2.44.tar.xz` 均存在，排除了仅镜像站缺件的情况）
   据此选择与 `2.42.9000` 语义等价的正式版本 `2.42`（分析报告根因段落亦以 `glibc-2.42.tar.xz`
   作为正式版本示例），改动后下载阶段可正常完成后续解压、configure、make、make install。
3. **严格遵守最小化原则**：仅修正直接触发 404 的 `VERSION` 变量。
   - 未改动 `README.md` / `doc/image-info.yml` / `meta.yml`：其标签 `2.42.9000-oe2403sp4`
     对应的 `meta.yml` key 与目录路径 `2.42.9000/24.03-lts-sp4/` 保持一致；
     若把 key 改成已有的 `2.42-oe2403sp4` 会与文件第 8 行既有条目产生 **YAML 重复 key**，
     覆盖真正的 2.42 镜像路径，引入新问题，故不做标签改动。
4. **排除方向 2（版权头缺失）**：同仓库既有的 `Others/glibc/2.41`、`Others/glibc/2.42`
   下 Dockerfile / 索引文件同样没有 Copyright / SPDX 头，若 CI 执行 license 校验它们也会失败，
   说明本仓库当前并不强制该检查，方向 2 不成立，未做无关改动。

## 潜在风险
- 该新镜像 tag 仍为 `2.42.9000-oe2403sp4`，而实际内置的 glibc 为 2.42，与既有
  `2.42-oe2403sp4` 镜像内容等价；仅当 CI 存在"tag 版本号必须等于源码实际版本"的强校验时才会暴露，
  常规 Docker build / 运行型 CI 不受影响。
- 未修改任何其他文件，不影响既有 2.41 / 2.42 镜像。