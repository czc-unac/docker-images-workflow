# 修复摘要

## 修复的问题
新增的 `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile` 使用了不存在的上游 git 引用 `v5.2.5`，导致 `git clone --branch v${VERSION}` 以 exit code 128 失败，Docker 构建中断。

## 修改的文件
- `HPC/cps_public/5.2.5/24.03-lts-sp4/Dockerfile`: 将 `ARG VERSION=5.2.5` 修改为 `ARG VERSION=5_2_5`，使 clone 引用由 `v5.2.5` 变为上游实际存在的 `v5_2_5`。

## 修复逻辑
CI 报错根因为 `fatal: Remote branch v5.2.5 not found in upstream origin`（Dockerfile:13）。RBC-UKQCD/CPS_public 上游仓库的 tag 采用下划线形式而非点号形式。已通过 `git ls-remote --tags https://github.com/RBC-UKQCD/CPS_public.git` 验证：

- `refs/tags/v5.2.5` —— 不存在（无输出）
- `refs/tags/v5_2_5` —— 存在（`ef4453256b6f49ffa2751dc15f73121996d034a2`）

同时对照仓库内已能正常构建的 `HPC/cps_public/5_2_5/24.03-lts-sp4/Dockerfile`，其写法即为 `ARG VERSION=5_2_5` + `--branch v${VERSION}`，证明该包约定的 `VERSION` 取值为下划线形式。因此做最小改动，仅调整 `VERSION` 值，修改后 `v${VERSION}` 解析为 `v5_2_5`，已再次用 `git ls-remote --exit-code` 验证存在（exit=0）。

该修改只影响新增镜像的 clone 步骤，`VERSION` 参数在 Dockerfile 中仅用于拼装 clone 引用，不参与镜像标签或其它逻辑；镜像标签 `5.2.5-oe2403sp4`（见 `meta.yml`/`README.md`/`image-info.yml`）无需改动。

## 潜在风险
无。改动为单个 ARG 默认值的修正，与同目录下既有可正常工作镜像的写法保持一致；README、image-info.yml、meta.yml 中描述的版本 5.2.5 与上游 tag v5_2_5 为同一版本，语义不变。