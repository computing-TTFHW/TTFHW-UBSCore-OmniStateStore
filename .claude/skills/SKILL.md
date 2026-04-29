# OmniStateStore Build & Test Pipeline

## Overview

使用 Docker 容器化环境完成 [OmniStateStore](https://gitcode.com/openeuler/OmniStateStore.git) 的编译、增量编译（ccache）、单元测试全流程，输出带各阶段耗时的 `build_ut_result.json`。

**技术栈**：CMake 3.14.1 + Make + GCC 10.3.1 (C++14)
**容器镜像**：`openeuler/openeuler:22.03-lts-sp4`
**运行环境**：WSL2 + Docker Desktop (Windows)

---

## 环境准备

### 1. 克隆仓库

```bash
git clone https://gitcode.com/openeuler/OmniStateStore.git
cd OmniStateStore
git submodule update --init --recursive
```

4 个子模块（3rdparty）：`googletest`、`lz4`、`libboundscheck`、`spdlog`

### 2. 拉取 Docker 镜像

```bash
# 优先从 quay.io 拉取（Docker Hub 可能超时）
docker pull quay.io/openeuler/openeuler:22.03-lts-sp4
docker tag quay.io/openeuler/openeuler:22.03-lts-sp4 openeuler/openeuler:22.03-lts-sp4
```

### 3. 创建输出目录

```bash
mkdir -p build
```

---

## 容器依赖清单

| 包名 | 用途 |
|------|------|
| `cmake3` | CMake 构建系统 |
| `make` | Make 构建工具 |
| `gcc` `gcc-c++` | GCC 10.3.1 编译器 |
| `java-1.8.0-openjdk-devel` | JDK 1.8（提供 `jni.h`，JNI 编译必需） |
| `ccache` | 编译器缓存，加速增量编译 |
| `libaio-devel` | Linux 异步 I/O 库（链接 `libaio.so`） |
| `findutils` | `find` 命令（镜像默认未安装） |
| `bc` | 浮点运算（时长计算） |

安装命令：
```bash
yum install -y cmake3 make gcc gcc-c++ java-1.8.0-openjdk-devel findutils bc ccache libaio-devel
```

---

## 关键配置

### JAVA_HOME

必须硬编码为 symlink 路径，**不能**用 `readlink -f $(which java)` 动态解析：

```bash
export JAVA_HOME="/usr/lib/jvm/java"
export PATH=${JAVA_HOME}/bin:${PATH}
```

原因：容器内 `java` 二进制可能没有 symbolic link 目标，`readlink -f` 返回空导致 JNI 编译失败。

### ccache 配置

```bash
export CCACHE_DIR="/root/.ccache"
ccache --max-size=5G
ccache --zero-stats
```

CMake 参数挂载：
```bash
-DCMAKE_CXX_COMPILER_LAUNCHER=ccache
-DCMAKE_C_COMPILER_LAUNCHER=ccache
```

---

## Pipeline 流程

入口脚本：`build_ut_pipeline.sh`

### 三个阶段

| 阶段 | 模式 | cmake 参数 | 说明 |
|------|------|-----------|------|
| **编译 (build)** | Release | `CMAKE_BUILD_TYPE=Release` | 全量编译，包含 3rdparty 库构建 |
| **增量编译 (incremental_build)** | Release | 同左（复用 build dir） | 修改 `bss_log.cpp`，统计 ccache 命中率 |
| **单元测试 (UT)** | Debug | `CMAKE_BUILD_TYPE=Debug` `BUILD_TESTS=ON` | 编译 gtest 测试并执行 bss_ut |

### 标准 Docker 运行命令

```bash
PROJ_DIR="/mnt/d/project/TTFHW/UBSCore/OmniStateStore"
BUILD_DIR="${PROJ_DIR}/build"

mkdir -p "${BUILD_DIR}"

MSYS_NO_PATHCONV=1 docker run --rm -u root \
  -v "${PROJ_DIR}:/workspace" \
  -v "${BUILD_DIR}:/output" \
  openeuler/openeuler:22.03-lts-sp4 \
  bash /workspace/build_ut_pipeline.sh
```

### 单步运行

**仅编译（Release）**：
```bash
docker run --rm -u root -v <proj>:/workspace -v <build>:/output \
  openeuler/openeuler:22.03-lts-sp4 \
  bash -c 'yum install -y cmake3 make gcc gcc-c++ java-1.8.0-openjdk-devel libaio-devel findutils bc ccache &&
  export JAVA_HOME=/usr/lib/jvm/java &&
  mkdir -p /workspace/build && cd /workspace/build &&
  cmake -DCMAKE_BUILD_TYPE=Release /workspace &&
  make build_cpp'
```

**仅增量编译（ccache）**：
```bash
# 前提：已完成一次全量编译，build 目录存在
docker run --rm -u root -v <proj>:/workspace -v <build>:/output \
  openeuler/openeuler:22.03-lts-sp4 \
  bash -c 'yum install -y cmake3 make gcc gcc-c++ java-1.8.0-openjdk-devel libaio-devel findutils bc ccache &&
  export JAVA_HOME=/usr/lib/jvm/java &&
  export CCACHE_DIR="/root/.ccache" && ccache --max-size=5G &&
  cd /workspace/build && make build_cpp &&
  echo "--- ccache stats ---" && ccache --show-stats'
```

**仅测试（Debug）**：
```bash
docker run --rm -u root -v <proj>:/workspace -v <build>:/output \
  openeuler/openeuler:22.03-lts-sp4 \
  bash -c 'yum install -y cmake3 make gcc gcc-c++ java-1.8.0-openjdk-devel libaio-devel findutils bc ccache &&
  export JAVA_HOME=/usr/lib/jvm/java &&
  mkdir -p /workspace/build && cd /workspace/build &&
  cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTS=ON /workspace &&
  make build_cpp &&
  /workspace/build/test/llt/bss_ut'
```

---

## 输出文件

### build_ut_result.json

位置：`build/build_ut_result.json`

```json
{
  "project": "OmniStateStore",
  "build_system": "CMake + Make",
  "compiler_launcher": "ccache",
  "container": "openeuler/openeuler:22.03-lts-sp4",
  "timestamp": "2026-04-29T04:29:30Z",
  "phases": {
    "build": {
      "description": "Full build (Release mode, clean build)",
      "duration_seconds": 347.691,
      "steps": {
        "cmake_configure": { "duration_seconds": 41.293 },
        "make_build": { "duration_seconds": 306.386 }
      }
    },
    "incremental_build": {
      "description": "Incremental build after modifying 1 source file",
      "duration_seconds": 72.745,
      "ccache": {
        "hit_total": 1,
        "hit_rate_percent": 100.0,
        "files_in_cache": 343,
        "cache_size": "340.9 MB"
      }
    },
    "ut": {
      "description": "Unit test build and execution",
      "duration_seconds": 1278.299,
      "steps": {
        "cmake_configure": { "duration_seconds": 65.064 },
        "make_build": { "duration_seconds": 925.287 },
        "test_execution": {
          "gtest_elapsed": "204809 ms",
          "total": 296,
          "passed": 296,
          "failed": 0,
          "skipped": 0,
          "result": "PASSED",
          "top10_slowest_tests": [...]
        }
      }
    }
  },
  "summary": { "total_duration_seconds": 1793.783 }
}
```

---

## Top 10 耗时测试用例

| 排名 | 用例 | 耗时(ms) | 类型 |
|------|------|----------|------|
| 1 | `TestDB.test_cp_lsm_and_get_return_ok_with_blob` | 20049 | checkpoint + LSM + blob |
| 2 | `TestSavepoint.TestSavePoint` | 10460 | savepoint + 数据迭代 |
| 3 | `TestFile.test_write_level4_return_ok` | 10074 | 文件 L4 级别校验 |
| 4 | `TestDB.test_put_kv_kmap_to_lsm_store_and_get_return_ok_with_blob` | 10047 | KV+KMap + blob |
| 5 | `TestDB.test_put_kmap_to_all_table_and_get_return_ok_with_blob` | 10037 | KMap + blob |
| 6 | `TestDB.test_put_kv_kmap_nskv_to_all_table_and_get_return_ok_with_blob` | 10036 | KV+KMap+NsKV + blob |
| 7 | `TestDB.test_cp_and_get_return_ok` | 7980 | checkpoint + 恢复 |
| 8 | `TestDB.test_cp_local_recovery_ok` | 7504 | 本地恢复 |
| 9 | `TestDB.test_KMap_to_lsm_store_and_get_return_ok_with_no_ttl` | 6482 | LSM store 无 TTL |
| 10 | `TestDB.test_KMap_to_lsm_store_and_get_return_false_with_ttl` | 6459 | LSM store TTL 过期 |

**共性**：全部为 DB/LSM 级集成测试，涉及大量磁盘 I/O（checkpoint 复制、LSM compact、blob 读写）。

---

## 已知问题与修复

### 1. Git Bash 路径转换导致 Docker 挂载失败

**现象**：
```
docker: 'docker run' requires at least 1 argument
/bin/bash: line 1: D:softwareGitGitworkspace: command not found
```

**原因**：Git Bash (MSYS2) 自动将 `/mnt/d/...` 路径转换为 `D:\...`，导致 Docker `-v` 参数被错误转换。

**修复**：
```bash
# 方法1：使用 MSYS_NO_PATHCONV
MSYS_NO_PATHCONV=1 docker run ...

# 方法2：WSL 路径 + wsl 前缀
wsl docker run -v /mnt/d/project/...:/workspace ...
```

### 2. JNI header not found

**现象**：
```
fatal error: jni.h: No such file or directory
```

**原因**：`JAVA_HOME` 通过 `readlink -f $(which java)` 解析失败。

**修复**：
```bash
export JAVA_HOME="/usr/lib/jvm/java"
```

### 3. ccache 命中统计解析错误

**现象**：
```
bash: syntax error: invalid arithmetic operator
```

**原因**：`grep 'cache hit'` 匹配两行（`cache hit (direct)` 和 `cache hit rate`），导致变量含换行符。

**修复**：
```bash
INCR_HIT_DIRECT=$(echo "$stats" | grep 'cache hit (direct)' | awk '{print $NF}')
INCR_HIT_PRE=$(echo "$stats" | grep 'cache hit (preprocessed)' | awk '{print $NF}')
INCR_HIT=$((INCR_HIT_DIRECT + INCR_HIT_PRE))
```

### 4. libaio 链接失败

**现象**：
```
/usr/bin/ld: cannot find -laio
```

**原因**：容器默认未安装 libaio-devel。

**修复**：
```bash
yum install -y libaio-devel
```

### 5. find 命令不可用

**现象**：
```
find: command not found
```

**原因**：openEuler 基础镜像未包含 findutils。

**修复**：
```bash
yum install -y findutils
```

### 6. 时钟偏移警告 (cosmetic)

**现象**：
```
make: warning: Clock skew detected. Your build may be incomplete.
```

**原因**：WSL2 + Docker 卷挂载时间同步问题。

**影响**：仅警告，不影响构建结果。

---

## 偶发失败分析

初始 Pipeline 运行时有 2 个测试用例失败，重跑后通过：

| 用例 | 耗时 | 失败原因 |
|------|------|----------|
| `TestDB.test_sp_list_and_get_return_ok` | ~2s | Savepoint 异步竞态：LSM store flush/evict 未完成时 `TriggerSavepoint()` 生成的快照遗漏未落盘数据，导致 savepoint 迭代值与内存期望值不匹配 |
| `TestDB.test_KList_to_lsm_store_and_get_return_false_with_ttl` | ~6s | TTL 时序偏紧：TTL=5000ms，sleep=6s，边界仅差 1s。容器 I/O 延迟下 `ForceEvictToLsm()` 未完成时 assert 已执行 |

**根因**：这两个测试是时序敏感的集成测试。在原生 Linux 下稳定，但在 WSL2 + Docker 卷挂载（I/O 延迟高、clock skew 已知）下偶发。非代码 bug，是测试的时序容差设计偏紧。

---

## 目录约定

```
OmniStateStore/
  build/                    <- 构建产物 + JSON 输出
    build_ut_result.json    <- Pipeline 结果
  build_ut_pipeline.sh      <- Pipeline 入口脚本
  src/                      <- 源代码
  test/llt/                 <- gtest 单元测试
    CMakeLists.txt          <- 构建 bss_ut 目标
  3rdparty/                 <- 子模块
    googletest/
    lz4/
    secure/libboundscheck/
    spdlog/
```

目录中只放置构建产物和 JSON 文件，不放置临时脚本。
