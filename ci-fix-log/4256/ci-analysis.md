# CI 失败分析报告

## 基本信息
- PR: #4256 — 【问题修复】3dslicer容器镜像升级至5.12.4版本
- 失败类型: build-error
- 置信度: 中
- 知识库匹配: 模式10（疑似匹配：CMake configure 阶段找不到所需依赖/组件）
- 新模式标题: (不填，疑似模式10)
- 新模式症状关键词: (不填)

> 前置检查：日志末尾为 `Finished: FAILURE`，不属于"日志显示成功但 PR 失败"的情形，故继续正常分析。

## 根因分析

### 直接错误
```
#13 3111.7 CMake Error at CMakeLists.txt:180 (message):
#13 3111.7 gmake[5]: *** [CMakeFiles/PythonQt.dir/build.make:97: PythonQt-cmake/src/PythonQt-stamp/PythonQt-configure] Error 1
#13 3111.7 gmake[4]: *** [CMakeFiles/Makefile2:207: CMakeFiles/PythonQt.dir/all] Error 2
#13 3111.7 gmake[3]: *** [Makefile:136: all] Error 2
#13 3111.7 gmake[2]: *** [CMakeFiles/CTK.dir/build.make:90: CTK-prefix/src/CTK-stamp/CTK-build] Error 2
#13 3111.7 gmake[1]: *** [CMakeFiles/Makefile2:1075: CMakeFiles/CTK.dir/all] Error 2
#13 3111.7 gmake: *** [Makefile:91: all] Error 2
#13 3112.0 -- Configuring incomplete, errors occurred!
#13 ERROR: process "/bin/sh -c ./build-CTKAppLauncher.sh &&     ./build-tbb.sh &&     if [ \"$TARGETARCH\" = \"arm64\" ]; then         BRANCH=\"main\";     fi &&     ./build-Slicer.sh ${BRANCH}" did not complete successfully: exit code: 2
```

失败前的最后几行配置输出（证明已找到 Qt5 与 Python，是随后的致命检查导致中断）：
```
#13 3112.0 -- Found Python3: /opt/Slicer-Release/python-install/include/python3.12 (found version "3.12.10") found components: Development Development.Module Development.Embed
#13 3112.0 -- PythonQt: Required Qt components [Core;Widgets;Multimedia;PrintSupport;Network;MultimediaWidgets;UiTools]
#13 3112.0 -- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Success
#13 3112.0 -- Found Threads: TRUE
#13 3112.0 -- Found OpenGL: /usr/lib64/libOpenGL.so
#13 3112.0 -- Configuring incomplete, errors occurred!
```

### 根因定位
- 失败位置: PythonQt 源码 `CMakeLists.txt:180`（由 CTK superbuild 的 `PythonQt-configure` 步骤触发；日志显示 PythonQt refer 为 `74dcd675e1515324cd7467a328d63dd25d263679`）
- 失败原因: PythonQt 的 CMake 配置阶段在第 180 行主动触发 `message(...)`（FATAL_ERROR）并终止，导致 CTK → PythonQt 的 superbuild 链条（`gmake[5] … gmake`）整体返回 Error 2，最终 `-- Configuring incomplete, errors occurred!`，Docker 构建 exit code: 2。
- 关键限制: `CMake Error at CMakeLists.txt:180 (message):` 下面的**消息正文没有出现在日志中**。本数据集 "Error lines" 仅保留含 error/fail 等关键词的行，而 `message()` 的说明文本不含此类关键词，被过滤掉了。因此无法从现有日志确认该致命检查的具体断言。

### 与 PR 变更的关联
本 PR 新增了 `HPC/3dslicer/5.12.4/24.03-lts-sp4/` 全套构建脚本并新增 `meta.yml` 条目：
- `build-Slicer.sh` 以 `-b v5.12.4` clone Slicer 并做 `cmake` 配置/构建，进而拉取并构建 CTK、PythonQt 等依赖；
- 日志中的失败步骤正是 Dockerfile 第 20–25 行新增的 `RUN ./build-CTKAppLauncher.sh && ./build-tbb.sh && … ./build-Slicer.sh ${BRANCH}`。

失败发生在本 PR 新增的构建链中（日志顶部 `> [7/7] RUN …` 也对应新增的 RUN 指令），**与 PR 改动直接相关**。当前失败为 amd64 构建（日志 `if [ "amd64" = "arm64" ]`、`./build-Slicer.sh v5.12.4`）。

## 修复方向

### 方向 1（置信度: 中）
PythonQt 配置所需的某个 Qt5 组件/系统 `-devel` 依赖在 Dockerfile 的 `yum install` 列表中缺失（或包名不匹配），导致 `CMakeLists.txt:180` 的检查失败。修复思路：先取回 PythonQt 提交 `74dcd675e1515324cd7467a328d63dd25d263679` 的 `CMakeLists.txt:180` 处的实际 `message()` 内容，据此在 Dockerfile 中补齐对应包（参考模式10 的补 `-devel` 思路）。**不提供具体代码。**

### 方向 2（置信度: 低）
PythonQt 该固定提交与镜像中 Qt 5.15.10 / Python 3.12.10 存在版本或特性兼容性检查不通过（例如版本下限/上限断言），属于版本锁定问题。修复思路：调整 PythonQt 的引用版本/提交或对应 CMake 开关。此方向证据不足，需先确认方向 1。

## 需要进一步确认的点
1. **最关键**：PythonQt `CMakeLists.txt:180` 的 `message()` 正文。现有日志在 `(message):` 行后直接跳到 `gmake` 报错，缺失说明文本，无法据此断定是缺包还是版本断言。
2. Dockerfile 已安装 `qt5-devel qt5-qtbase-devel qt5-qtx11extras-devel qt5-qtmultimedia-devel qt5-qtsvg-devel qt5-qtwebengine-devel qt5-qtxmlpatterns-devel qt5-qttools-devel qt5-qtbase-private-devel`，而 PythonQt 声明需要 `Core;Widgets;Multimedia;PrintSupport;Network;MultimediaWidgets;UiTools`：需确认 openEuler 24.03-sp4 中 `Qt5::UiTools`、`Qt5::MultimediaWidgets`、`Qt5::X11Extras` 等组件分别由哪个包提供，是否全部落位。
3. Slicer v5.12.4 所期望的 CTK/PythonQt 版本与 build 时实际取到的固定提交（PythonQt `74dcd67…`）是否一致。
4. 建议获取该失败 Docker 构建步骤 `#13` 中 `PythonQt-configure` 前后的完整未截断日志（尤其是 `CMakeLists.txt:180` 消息正文与更早的 `Could NOT find …`/`Could not find …` 行）。

## 修复验证要求（条件性）
当前置信度为"中"，尚不能确定根因是缺包还是版本断言。若最终修复需要改动外部源文件（例如修改 Slicer 源码中引用 PythonQt 的 External project 配置或对 PythonQt 源码打正则补丁），则：
- code-fixer 必须先按 Dockerfile 中 `ARG VERSION=5.12.4`（Slicer 分支 `v5.12.4`）从上游拉取对应文件，并依据实际出现的 PythonQt 提交 `74dcd675e1515324cd7467a328d63dd25d263679` 校验目标内容，确认新正则/改动确实匹配后再提交；
- 若修复仅为在 Dockerfile 中补充 `yum install` 的 `-devel` 包，则无需正则匹配验证，但需先在 openEuler 24.03-lts-sp4 上确认包名存在。
