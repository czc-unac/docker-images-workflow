# 修复摘要

## 修复的问题
CI 分析报告判定为 `infra-error`（日志缺失，证据不足），未确认任何代码根因，因此本次**未修改任何代码**。

## 修改的文件
- 无（未修改任何文件）

## 修复逻辑
分析报告明确指出：

- 失败类型：`infra-error（CI 日志缺失，无法定位）`；
- `ci.logs = "(not available — analyze based on PR diff only)"`，未提供任何失败 job 日志；
- 报告自身声明"日志不足以确定根因时必须判定为证据不足"，其列出的方向 1/方向 2 均为"未经日志证实的候选排查方向"，不构成根因结论。

依据 Agent 约束"如果分析报告指出是 `infra-error`（CI 基础设施问题），在 output_file 中说明无需代码修改，不要强行改代码"，本次不做任何猜测性改动，避免引入新问题或掩盖真实故障。

为确认"是否确实存在可定位的代码缺陷"，已对 PR 涉及的 5 个文件做静态核查，结论如下（均正常，无需要修改之处）：

1. **新镜像构建配方与已验证版本一致**
   - `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile` 与已存在的 `Storage/ceph/20.3.0/24.03-lts-sp4/Dockerfile` 逐字节对比，仅 `ARG VERSION=20.3.0` → `21.3.0` 一行不同；
   - `entrypoint.sh` 两版本完全相同（diff 为空）；
   - 说明新目录并非引入全新的、未经检验的构建逻辑。

2. **上游 tag 真实存在（已联网核实）**
   - `git ls-remote --tags https://github.com/ceph/ceph.git` 返回：
     - `refs/tags/v21.3.0` 有效；
     - `refs/tags/v20.3.0` 有效；
   - 因此 Dockerfile 中 `git clone -b v21.3.0 ...` 不会因 tag 不存在而失败。

3. **元数据内部一致、与成功合并的自动升级 PR 模式一致**
   - `meta.yml` 新增 `21.3.0-oe2403sp4: path: 21.3.0/24.03-lts-sp4/Dockerfile`，指向的路径真实存在；
   - `doc/image-info.yml`、`README.md` 的 tags 表均新增了 `21.3.0-oe2403sp4` 行；
   - `Storage/image-list.yml` 以场景目录为最小单元（`ceph: ceph`），按项目规范无需为新增版本追加条目；
   - 全库检索确认没有任何其它文件引用 ceph 版本号，无需同步更新。

4. **排除"文件末尾换行缺失"这一伪根因**
   - PR 的改动移除了 `README.md`、`doc/image-info.yml`、`meta.yml` 末尾换行（diff 中 `\ No newline at end of file`）；
   - 但仓库现状中 284 个 `meta.yml` 均无末尾换行，且近期成功合并的同类自动升级 PR（如 reedsolomon `1.14.2`、starrocks `4.1.3`）结果同样无末尾换行仍通过 CI；
   - 故该差异不是 CI 失败原因，不做修改。

5. **排除分析报告列出的低置信度候选点**
   - `ENV LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH` 自引用变量在 20.3.0 中同样存在且其镜像已合入仓库，非本次新增缺陷；
   - 若构建阶段确因 ceph 21.3.0 新增依赖失败，需实际 `error:`/`FAILED` 日志才能确定缺失的 `-devel` 包，盲目增删依赖风险高，故不在无证据情况下修改。

## 潜在风险
无（未对源代码做任何改动）。若该 PR 的 CI 实际仍在 Docker 构建阶段失败，需补充失败架构 job 的完整日志（`ci.logs` 中首个 `ERROR`/`error:`/`FAILED` 之前的内容）后才能定位并修复；在此之前任何代码改动都属于猜测。