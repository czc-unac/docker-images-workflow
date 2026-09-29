# CI 失败分析报告

## 基本信息
- PR: #4714 — 【自动升级】torchvision容器镜像升级至0.29.0版本.
- 失败类型: infra-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: 构建器连接中断
- 新模式症状关键词: rpc error, Unavailable, graceful_stop, no builder found, buildkit, docker-container

## 根因分析

### 直接错误
```
#8 8.277 Installing collected packages: mpmath, typing-extensions, sympy, setuptools, pillow,
            numpy, networkx, MarkupSafe, fsspec, filelock, jinja2, torch, torchvision
ERROR: failed to receive status: rpc error: code = Unavailable desc = closing transport due to:
       connection error: desc = "error reading from server: EOF", received prior goaway:
       code: NO_ERROR, debug data: "graceful_stop"
ERROR: no builder "euler_builder_20260929_082519" found
Build step 'Execute shell' marked build as failure
Finished: FAILURE
```

### 根因定位
- 失败位置: aarch64 构建 job，`#8 [3/3] RUN pip install --no-cache-dir --index-url https://download.pytorch.org/whl/cpu torch==2.14.0 torchvision==0.29.0` 步骤的包安装阶段（Installing collected packages）。
- 失败原因: buildx 构建器容器 `euler_builder_20260929_082519`（docker-container driver / moby/buildkit）在与 BuildKit daemon 的通信中被 `graceful_stop`（对端优雅关闭，EOF），随后构建器被移除（`no builder ... found`），导致构建中断。这是构建基础设施/编排层的故障，而非 Dockerfile 语法或依赖错误。

### 与 PR 变更的关联
本次日志显示 pip 依赖解析全部成功：`torch==2.14.0` 与 `torchvision==0.29.0` 的元数据、wheel（torch 159.2 MB、torchvision 2.1 MB 及全部传递依赖）均下载完成，并已进入 `Installing collected packages`，**没有出现 ResolutionImpossible 或任何 pip 报错**。因此 PR 的变更不是本次失败的直接触发原因，失败发生在安装过程中构建器失联的瞬间。

## 修复方向

### 方向 1（置信度: 中）
本失败为 infra-error，与代码无关：重跑 CI 流水线即可，不需要改动 Dockerfile 或元数据文件。

### 方向 2（置信度: 低）
若重跑后构建器仍反复消失，需在运维侧排查 aarch64 runner 上 docker/buildkit 的稳定性与资源（磁盘、内存、构建器生命周期/清理策略），确认是否有并发清理或 OOM 导致构建器被 graceful stop。

## 需要进一步确认的点
- **版本不一致**：`ci.logs` 中实际安装的是 `torch==2.14.0`，而 PR diff 中 Dockerfile 声明 `ARG TORCH_VERSION=2.12.1`（第 4 行），且 `torch==${TORCH_VERSION}`。需确认仓库中该 Dockerfile 的真实 `TORCH_VERSION` 取值。
- **潜在依赖冲突（模式23）**：torchvision 对 torch 有精确版本约束（历史案例：torchvision 0.27.1 需 `torch==2.12.1`）。若实际部署仍为 `2.12.1`，而 torchvision 0.29.0 要求 `torch==2.14.0`，则在安装阶段会触发 pip `ResolutionImpossible`（模式23 同类问题）。本次日志因构建器失联而未观察到该报错，需要对应构建日志确认。
- **下游 job 日志缺失**：当前日志仅覆盖到 BuildKit 构建层，构建器失联后 `pip install` 的最终结果未知。需要获取 x86-64 / aarch64 架构专属构建 job 的完整日志，确认是否还有被掩盖的依赖冲突或其他错误。
- **触发分支**：日志显示 `originally caused by: PR 4738 [unac:fix/4714 -> master] trigger by merge_request`，需确认 PR 4738 是否即为对 PR 4714 的修复提交，以对齐所分析的 diff 与实际构建内容。

## 修复验证要求
本报告置信度为"中"，code-fixer 在采取任何动作前必须：
1. 确认本次失败是否为单次 infra 故障：优先触发一次重跑，观察是否可复现。若重跑成功，则本 PR 无代码问题，不应改动 Dockerfile。
2. 若重跑仍失败，核实 Dockerfile 中 `TORCH_VERSION` 的真实值与 `torchvision==0.29.0` 上游所要求的精确 torch 版本是否一致（参考模式23 的验证方式），并取得实际报错日志后再定位。
