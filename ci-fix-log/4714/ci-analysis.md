# CI 失败分析报告

## 基本信息
- PR: #4714 — 【自动升级】torchvision容器镜像升级至0.29.0版本.
- 失败类型: infra-error
- 置信度: 中
- 知识库匹配: 新模式
- 新模式标题: BuildKit构建器中途停止
- 新模式症状关键词: failed to receive status, rpc error code Unavailable, no builder found, graceful_stop, buildx builder

## 根因分析

### 直接错误
```
#8 8.277 Installing collected packages: mpmath, typing-extensions, sympy, setuptools, pillow, numpy, networkx, MarkupSafe, fsspec, filelock, jinja2, torch, torchvision
ERROR: failed to receive status: rpc error: code = Unavailable desc = closing transport due to: connection error: desc = "error reading from server: EOF", received prior goaway: code: NO_ERROR, debug data: "graceful_stop"
ERROR: no builder "euler_builder_20260929_082519" found
Build step 'Execute shell' marked build as failure
Notifying upstream projects of job completion
Finished: FAILURE
```

### 根因定位
- 失败位置: 非代码位置。发生在 aarch64 构建 job 的 Docker 构建第 `[3/3]` 步 `pip install torch torchvision` 期间，BuildKit 构建器 `euler_builder_20260929_082519`（`docker-container` driver）被回收/优雅关闭。
- 失败原因: 客户端与 BuildKit daemon 的 gRPC 连接收到 `goaway ... graceful_stop`，随后提示该 builder 已不存在。构建器在 `pip install` 下载/安装 `torch`、`torchvision`（约 159MB torch wheel）过程中被停止，导致构建中断。这属于构建基础设施层面的中断，而非 Dockerfile 内容触发的编译/依赖错误。

### 与 PR 变更的关联
- 本 PR 仅在 `AI/torchvision/` 下新增/更新 Dockerfile、README、`doc/image-info.yml`、`meta.yml`，Dockerfile 结构正常（`dnf install` 已成功，`pip install` 的依赖解析与下载均成功），日志中没有任何因本次改动引发的代码/依赖/语法错误。
- 从日志看 `pip` 已完成索引解析并成功下载全部 wheel（torch 2.14.0、torchvision 0.29.0 等），未出现 `ResolutionImpossible` 等依赖冲突，可排除模式23（PyTorch版本锁定冲突）。
- 因此该失败与 PR 变更无直接因果关系，属基础设施中断。
- 需注意的异常点: 日志中实际执行的 pod 为 PR 4738（`unac:fix/4714 -> master`，build 5048），且构建时使用的是 `torch==2.14.0`，与上下文 `pr.diff` 中 `ARG TORCH_VERSION=2.12.1` 不一致，说明被构建的修订与所提供的 diff 可能并非同一版本，证据链存在偏差。

## 修复方向

### 方向 1（置信度: 中）
- 重新触发 CI / 重跑该 job。构建器 `graceful_stop` 属于 CI 环境中的临时性基础设施事件（builder 被回收、daemon 重启或资源清理），与代码无关，重试通常即可通过。

### 方向 2（可选，置信度: 低）
- 若重跑后仍在同一 `pip install` 步骤稳定失败，则需怀疑构建节点资源（磁盘/内存）不足导致 BuildKit 被强制回收；此时应确认 runner 的磁盘/内存水位，而非修改 Dockerfile。

## 需要进一步确认的点
1. 需获取本次真正失败 job 的完整日志（尤其是 `pip install` 之后是否出现 `Killed`、`no space left on device`、`Aborted`、Jenkins timeout 等信号），以区分“临时回收”与“资源耗尽”。
2. 确认被构建的实际修订：日志显示构建内容为 PR 4738 且 `torch==2.14.0`，而 `pr.diff` 为 `TORCH_VERSION=2.12.1`，需核对当前 PR 分支上 Dockerfile 的真实 `TORCH_VERSION` 值，避免误判。
3. 确认是否存在并发 job 共用/清理同一 buildx builder（`euler_builder_20260929_082519`）的编排逻辑，导致构建中途被回收。
4. 确认 CI 对 buildx builder 是否设有超时/生命周期回收策略。

## 修复验证要求
本失败判定为 `infra-error`，未涉及任何正则 patch 或外部源文件匹配，Code Fixer 无需修改代码。若后续重跑仍复现同一错误，先补充上述第 1、3、4 点证据再决定是否需要修复动作。
