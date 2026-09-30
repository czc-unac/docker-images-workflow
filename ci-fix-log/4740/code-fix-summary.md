# 修复摘要

## 修复的问题
ceph 21.3.0 镜像在 x86_64/aarch64 构建时，链接 `librgw.so` 因 Apache Thrift 符号未解析（`Link Error: RGW library not found`）而失败；通过 `-DWITH_RADOSGW_SELECT_PARQUET=OFF` 关闭依赖 Arrow 的 S3 Select Parquet 组件消除该链接错误。

## 修改的文件
- `Storage/ceph/21.3.0/24.03-lts-sp4/Dockerfile`: 在 `./do_cmake.sh` 参数中追加 `-DWITH_RADOSGW_SELECT_PARQUET=OFF`（第 43 行）。

## 修复逻辑
分析报告本身不含 `ci.logs`（标记为 `(not available)`），因此报告无法定位根因。为获得真实证据，我从 PR #4740 评论中取得门禁实际构建 job 的 Jenkins 链接并抓取了两架构的完整控制台日志：
- x86_64: `https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/x86-64/openeuler-docker-images/4822/`
- aarch64: `https://log-ci.openeuler.openatom.cn/job/multiarch/openeuler/aarch64/openeuler-docker-images/4918/`

两个架构的失败点完全一致（均 `collect2: error: ld returned 1 exit status` + `Link Error: RGW library not found`），发生在 `[1706/1762]` 生成 Python RGW 绑定（`src/pybind/rgw`）链接 `librgw.so` 时，报错为：
```
/usr/bin/ld: /opt/ceph/build/lib/librgw.so: undefined reference to `apache::thrift::protocol::TProtocolFactory::~TProtocolFactory()'
... typeinfo for apache::thrift::transport::TMemoryBuffer
... vtable for apache::thrift::protocol::TProtocol
... apache::thrift::protocol::TProtocol::skip_virt(...)
```
即 `librgw.so` 中残留未解析的 Apache Thrift 符号（共 10 个，两架构相同）。

根因定位（已从上游 `v21.3.0` tag 拉取源文件核对）：
- `src/rgw/rgw_s3select.cc` 在处理 Parquet 对象时使用 Apache Arrow（`#ifdef _ARROW_EXIST`），`src/rgw/CMakeLists.txt:8-9` 仅在 `WITH_RADOSGW_SELECT_PARQUET` 开启时把 `ARROW_LIBRARIES`（`Arrow::Arrow Arrow::Parquet`）链接进 `rgw_common`/`librgw`。
- 上游 `CMakeLists.txt:579` 中该选项默认 `ON`；构建日志显示 configure 阶段打印 `arrow is installed, radosgw/s3select-op is able to process parquet objects`，随后 ceph 从源码构建了 bundled Arrow 19.0.1（`ARROW_BUILD_STATIC=ON`、`ARROW_THRIFT_USE_SHARED=OFF`，日志 7474/7671）。
- 上游 `cmake/modules/BuildArrow.cmake:23-29`（v21.3.0 相对 v20.3.0 的变更）：当系统 thrift 版本 `< 0.17` 时，ceph 改为让 Arrow 自带 bundled thrift 构建，并**不再**把 `thrift` 加入 `Arrow::Arrow` 的 `INTERFACE_LINK_LIBRARIES`；而 openEuler 24.03-LTS-SP4 自带 thrift 为 `0.14.0`（日志 5844：`Found thrift: /usr/lib64/libthrift.so (found suitable version "0.14.0")`），正好落入该分支，导致 Arrow 静态库携带的 thrift 符号未被传递给 `librgw` 链接。这是 ceph 21.3.0（v20.3.0 为无条件 `arrow_INTERFACE_LINK_LIBRARIES thrift`）引入的回归。
- 验证：上游 `src/CMakeLists.txt:1173-1190` 中 `build_arrow()` 的调用受 `if(WITH_RADOSGW_SELECT_PARQUET OR WITH_RADOSGW_ARROW_FLIGHT)` 控制；关闭 `WITH_RADOSGW_SELECT_PARQUET`（且 `WITH_RADOSGW_ARROW_FLIGHT` 默认 OFF）后不再构建/链接 Arrow，`ARROW_LIBRARIES` 为空，`librgw` 不再引用 thrift 符号。`rgw_s3select.cc` 在未定义 `_ARROW_EXIST` 时走无 Arrow 的降级分支（`#ifndef _ARROW_EXIST` → "arrow library is not installed"），代码仍可正常编译。

该修复与 PR 作者此前处置同类构建期可选组件联网/依赖问题的方式一致（`-DWITH_MGR_DASHBOARD_FRONTEND=OFF`、`-DWITH_JAEGER=OFF`），仅关闭可选的 S3 Select Parquet 能力，不影响 ceph mon/osd/mds/mgr 及常规 RGW 功能，属最小改动。

## 潜在风险
- 关闭 `WITH_RADOSGW_SELECT_PARQUET` 后，librgw 不再支持对 Parquet 对象的 S3 Select 查询（仅此可选特性受影响）；RGW 本体及其余 S3 功能不受影响。
- 若后续门禁在通过本链接点后暴露出更靠后的编译/链接问题（日志此前在首个错误处即停止），可能还需继续修复。
- 未改动的既有 BuildKit 警告（`ENV LD_LIBRARY_PATH=...:$LD_LIBRARY_PATH` 的 `UndefinedVar`、`FROM ... as` 的 `FromAsCasing`）不导致构建失败，按最小化原则未处理。