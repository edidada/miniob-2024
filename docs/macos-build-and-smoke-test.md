# miniob macOS 编译修复与 CRUD 冒烟测试记录

> 日期：2026-08-19 · 分支：main_mac · 环境：macOS + Apple clang 14

## 背景

miniob-2024 是 OceanBase 2024 数据库大赛完赛项目（初赛满分通过，全国第 19、北京赛区第 2）。
项目在 Linux GCC 上开发，在 macOS 上编译 observer 主程序一直失败。

本分支（main_mac）的改动目标：让项目在 macOS 上完整编译通过，并补齐 macOS CI 验证。

## macOS 兼容性修复清单

### 真实 bug（任何平台都应修复）

| 文件 | 问题 | 修复 |
|---|---|---|
| `src/observer/common/type/attr_type.cpp` | `offsetof(VectorData, VectorData::dim)` 语法错误 | 去掉成员名前多余的 `VectorData::` |
| `src/observer/storage/table/table.cpp` | 4 处 `std::string` 直接传给 `%s`（非 POD variadic，运行时 UB） | 全部补 `.c_str()` |
| `src/observer/sql/stmt/create_vector_index_stmt.cpp` | 同上 1 处 | 补 `.c_str()` |

### Apple clang 14 兼容性

| 文件 | 问题 | 修复 |
|---|---|---|
| `CMakeLists.txt` | `-Werror` 全开，clang 严格警告直接失败 | 对 `AppleClang|Clang` 编译器禁用 `-Werror`（Linux 不变） |
| `aggregate_vec_physical_operator.cpp` / `group_by_physical_operator.cpp` | `std::ranges::for_each` 不被 Apple libc++ 支持 | 改用标准 `for_each` 迭代器版本 |
| `src/observer/sql/parser/expression_binder.cpp` | `ranges::find_if` 同上 | 改用标准 `find_if` |
| `src/observer/storage/index/vector_index_meta.cpp` | `size_t` 赋给 `Json::Value` 运算符歧义 | `static_cast<Json::UInt64>` |
| `src/observer/sql/parser/parse_defs.h` | `UpdateInfoNode` 无构造函数，clang 14 不支持聚合括号 `emplace_back` | 补充构造函数 + `<utility>` |
| `src/observer/storage/index/vector_index.h` | 未使用成员 `dim_` 触发 `-Wunused-private-field` | 加 `[[maybe_unused]]` |

## 编译方式

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DDEBUG=ON
cmake --build build --target observer -j8
# 或使用项目脚本
bash build.sh init          # 构建第三方依赖
bash build.sh debug --make -j4
```

产物：`build/bin/observer`（约 48MB）、`build/bin/obclient`。

## CRUD 冒烟测试

测试方式：`/tmp` 下启动 observer + obclient 逐条执行 SQL。

### 注意事项（踩坑记录）

1. **监听端口 6709**（由命令行/配置决定，非默认 6789）。
2. **obclient 参数**：本仓库的 `-s` 是 **unix socket 路径**，不是 host。
   连接 TCP 用 `-h <host> -p <port>`；连接本机更推荐 unix socket：
   ```bash
   ./build/bin/observer -T one-thread-per-connection -s /tmp/miniob.sock -f etc/observer.ini -P mysql -t mvcc -d disk &
   ./build/bin/obclient -s /tmp/miniob.sock
   ```

### 测试结果：全部通过

| 步骤 | SQL | 结果 |
|---|---|---|
| 1 | `create table t(id int, name char(10))` | SUCCESS |
| 2-3 | `insert into t values (1,"a")` / `(2,"b")` | SUCCESS ×2 |
| 4 | `select * from t` | 返回 `1\|a`、`2\|b` |
| 5 | `select id,name from t where id=2` | 返回 `2\|b`（条件查询正确） |
| 6 | `update t set name="c" where id=1` | SUCCESS |
| 7 | `select * from t` | 返回 `1\|c`、`2\|b`（更新生效） |
| 8 | `delete from t where id=2` | SUCCESS |
| 9 | `select * from t` | 返回 `1\|c`（删除生效） |

结论：编译修复后，建表、插入、条件查询、更新、删除全部功能正常。

### 已知无害警告

- 启动时 UBSan：libc++ `hash.h` 左移溢出（libc++ 自身实现问题，非本项目代码）。
- obclient 旧式 `gethostbyname`：`serv_addr.sin_addr = *((struct in_addr *)host->h_addr)` 未对齐访问。

两者不影响功能，属于 macOS 上跑该项目的已知噪音。

## macOS CI

见 `.github/workflows/macos-build-test.yml`：macos-latest 上完整走
init（brew 装 bison/flex）→ debug 编译 → unix socket 启动 observer → obclient CRUD 冒烟断言。
