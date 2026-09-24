# 修复摘要

## 修复的问题
修复 Slicer 5.12.4 构建时 PythonQt 的 CMake 配置失败（`CMake Error at CMakeLists.txt:180 (message): PythonQt: missing generated wrapper sources for Qt 5.15`），根因是 PythonQt 绑定生成器在 openEuler 上解析出 0 个类、未产出 `generated_cpp` 目录。

## 修改的文件
- `HPC/3dslicer/5.12.4/24.03-lts-sp4/build-Slicer.sh`：在脚本开头（`set -eo pipefail` 之后）新增 `export PYTHONQT_INCLUDE=/usr/include`，使环境变量传递给 Slicer 构建过程中由 CTK 调用的 PythonQt 绑定生成器（PythonQtGenerator）。

## 修复逻辑
1. 通过 Jenkins 原始日志确认截断的消息正文为：
   ```
   PythonQt: missing generated wrapper sources for Qt 5.15.
   Expected: /opt/Slicer-Release/CTK-build/PythonQtGenerator-output-5.15.10/generated_cpp
   ```
   即 CTK 的 `PythonQt.cmake` 在 `PythonQt-configure` 前运行的 `GenerateWrapper` 步骤虽返回成功，但并未生成 `generated_cpp`。

2. 定位到依赖链：Slicer → CTK → PythonQt。CTK 使用 PythonQt 的 `PythonQtGenerator` 生成绑定源码，生成器把结果写入 `<--output-directory>/generated_cpp/`（`generator/setupgenerator.cpp:280`、`generator/shellgenerator.h:56`），再由 `-DPythonQt_GENERATED_PATH` 指向该目录（CTK 传入的正是 `.../PythonQtGenerator-output-5.15.10/generated_cpp`）。
3. 生成器内部用 simplecpp 预处理 Qt 头文件时，只把 Qt 头目录加入 include 路径，**未加入 glibc 头目录**。openEuler 上 `<bits/wordsize.h>` 找不到导致 `__WORDSIZE` 未定义，`qconfig.h` 触发 `#error "unexpected value for __WORDSIZE macro"`，最终 `classes` 为 0，`generate()` 提前返回、不创建 `generated_cpp`，但生成器退出码为 0，故 `GenerateWrapper` 显示成功、随后的 PythonQt 配置因目录不存在而失败。
4. 生成器支持从环境变量 `PYTHONQT_INCLUDE` 读取额外 include 路径（`generator/main.cpp:100-107`，仅生成器使用该变量，无副作用）。补上 `/usr/include` 后 `__WORDSIZE` 可正确解析，生成器正常产出绑定源码。

### 修复验证（已实测）
在 `openeuler/openeuler:24.03-lts-sp4` 容器内安装 `cmake gcc gcc-c++ qt5-devel qt5-qtbase-devel qt5-qttools-devel` 等，用 Slicer 5.12.4 所用 PythonQt 提交 `74dcd675e1515324cd7467a328d63dd25d263679` 的 `generator` 编译出 `PythonQtGenerator`，按 CTK 相同方式运行：
- 不设 `PYTHONQT_INCLUDE`：`Classes in typesystem: 0`，`gen-out` 下**无** `generated_cpp`，与 CI 现象完全一致（stderr 首行即 `qconfig.h:19: #error "unexpected value for __WORDSIZE macro"`）。
- 设置 `PYTHONQT_INCLUDE=/usr/include`：`Classes in typesystem: 873`，成功生成 `generated_cpp/`。
- 随后以 `-DPythonQt_GENERATED_PATH=/work/gen-out2/generated_cpp` 配置 PythonQt，输出 `Required Qt components [Core;Widgets;Multimedia;PrintSupport;Network;MultimediaWidgets;UiTools]` 后 `Configuring done`，原先第 180 行的致命检查通过。

本修复不涉及对第三方/上游源文件的正则 patch，故无需正则匹配验证。环境变量会随进程继承传递到 `cmake --build` 触发的 CTK/ExternalProject/生成器，无需改动 CTK 或上游源码。

## 潜在风险
- 仅对 PythonQt 绑定生成器生效（该环境变量只被 `generator/main.cpp` 读取），不影响 Slicer、CTK、TBB 的编译与安装。
- 生成器仍会打印若干 simplecpp `#error`（如 `Qt has not been ported to this OS`、`requires C++11`），这是该生成器对预处理宏的既有行为（Ubuntu 上亦如此），不影响类解析；本次修复只需保证 `generated_cpp` 能被产出。
- aarch64 上 glibc 头同样位于 `/usr/include/bits/`，`PYTHONQT_INCLUDE=/usr/include` 同样适用；如需额外系统头目录可按 `:` 追加路径。