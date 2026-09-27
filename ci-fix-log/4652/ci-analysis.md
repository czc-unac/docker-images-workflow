# CI 失败分析报告

## 基本信息
- PR: #4652 — 【自动升级】onnxruntime容器镜像升级至1.30.0版本.
- 失败类型: `dependency-error`
- 置信度: 高
- 知识库匹配: 新模式
- 新模式标题: yum包名不存在
- 新模式症状关键词: `Unable to find a match`, `No match for argument`, `gcc-toolset-14-c++`, `yum install`, `exit code: 1`

## 根因分析

### 直接错误
```
#7 158.0 Package tar-2:1.35-8.oe2403sp4.x86_64 is already installed.
#7 158.0 Package ca-certificates-2023.2.64-6.oe2403sp4.noarch is already installed.
#7 160.1 No match for argument: gcc-toolset-14-c++*
#7 160.2 Error: Unable to find a match: gcc-toolset-14-c++*
#7 ERROR: process "/bin/sh -c yum update -y &&     yum install -y
     tar
     ca-certificates
     gcc-toolset-14-gcc*          <- 有匹配
     gcc-toolset-14-binutils*     <- 有匹配
     gcc-toolset-14-c++*          <- 无匹配（根因）
     ... git && yum clean all ..." did not complete successfully: exit code: 1
ERROR: failed to solve: process ... did not complete successfully: exit code: 1
```

### 根因定位
- 失败位置: `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`:14（`RUN yum install` 步骤中的 glob `gcc-toolset-14-c++*`）
- 失败原因: openEuler 24.03-LTS-SP4 的软件仓库中不存在以 `gcc-toolset-14-c++` 开头的 RPM 包名，yum 对该 glob 报 `No match for argument`，导致整条 `yum install` 以 exit code 1 退出，Docker 构建在 `[builder 2/4]` 层失败。

### 与 PR 变更的关联
本次 PR 新增了 `AI/onnxruntime/1.30.0/24.03-lts-sp4/Dockerfile`，其第 12-14 行分别安装了 `gcc-toolset-14-gcc*`、`gcc-toolset-14-binutils*`、`gcc-toolset-14-c++*` 三个 glob。日志证据显示前两个 glob 均解析成功（未报错），唯独 `gcc-toolset-14-c++*` 报 `No match for argument`。因此失败由本 PR 新增 Dockerfile 中的错误包名直接触发，与上游 onnxruntime 源码无关。

- 失败类型判定依据：错误发生在依赖包安装阶段（`yum install`），本质是包名无法解析，而非编译期错误，故归类为 `dependency-error`。
- 日志末尾为 `Finished: FAILURE`，日志与失败状态一致，不存在"日志成功但状态失败"的触发/编排层假象。

## 修复方向

### 方向 1（置信度: 高）
移除无效的 `gcc-toolset-14-c++*` 参数。openEuler 的 gcc-toolset C++ 编译器包名遵循 `gcc-toolset-<ver>-gcc-c++` 规则，即 C++ 编译器实际为 `gcc-toolset-14-gcc-c++`，通常已被现有 `gcc-toolset-14-gcc*` glob 覆盖，因此 `gcc-toolset-14-c++*` 属于多余且前缀错误的参数。

### 方向 2（置信度: 中）
若确需显式声明 C++ 编译器，则应改用仓库中真实存在的包名（如 `gcc-toolset-14-gcc-c++`），而非 `gcc-toolset-14-c++*`。建议先以 `dnf list available 'gcc-toolset-14-*'` 确认实际包名后再定名。

## 需要进一步确认的点
- 日志仅证明 `gcc-toolset-14-c++*` 无匹配，未列出仓库中全部可用的 gcc-toolset-14 包名。需确认 openEuler 24.03-LTS-SP4（含 EPOL/update 源）中 gcc-toolset-14 系列的确切包名清单。
- 后续步骤 `ln -sf /usr/lib64/libgcc_s.so.1 /opt/openEuler/gcc-toolset-14/root/usr/lib64/libgcc_s.so.1` 与 `source /opt/openEuler/gcc-toolset-14/enable` 依赖 gcc-toolset-14 的安装路径；包名修正后需确认 `/opt/openEuler/gcc-toolset-14/` 与实际安装路径一致。
- 构建日志中 `source /opt/openEuler/gcc-toolset-14/enable` 使用 `source`（bash 内建），RUN 默认 `/bin/sh`，需确认基础镜像的 `/bin/sh` 是否支持 `source`。
- 构建日志中的 `FromAsCasing`（`as`/`FROM` 大小写不一致，Dockerfile 第 4、40 行）仅为警告，非本次失败根因，但可作为顺带清理项。

## 修复验证要求
本次修复不涉及"修改正则 patch 外部源文件"，无需填写该验证要求。
