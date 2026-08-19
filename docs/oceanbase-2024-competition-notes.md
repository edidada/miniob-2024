# OceanBase 2024 数据库大赛（miniob）赛题解析与实现说明

> 仓库：北京科技大学「我真的参加了系统内核赛」队（王诺贤、廖玮珑、陈渠）
> 成绩：初赛满分通过，全国第 19，北京赛区第 2，校第 1
> 比赛平台：OceanBase 训练营（自动化黑盒测试，按测试用例输出比对判分）
> 本文件整理：赛题内容、涉及知识点、完成思路、对应代码位置

---

## 一、比赛赛题是什么

这是 **OceanBase 2024 数据库内核实现大赛**（miniob 项目）的初赛题。
miniob 是一个面向教学的迷你数据库内核（约几万行 C++），选手要在其上实现各类 SQL 功能。
题目分为**必做题**与**选做题**，先完成必做题，选做题才开始计分。

赛题范围可从此仓库的自动化测试用例中完整还原
（`test/case/test/` 下 20 个 `.test` 文件，`test/case/result/` 是对应期望输出）：

### 必做题（基础功能）

| 测试用例 | 功能 | 说明 |
|---|---|---|
| `basic.test` | 基础 CRUD | 建表、插入、删除、单表查询、where 条件、索引使用、`calc` 表达式计算 |
| `primary-select-meta.test` | 查询元数据校验 | 查询不存在的列名/表名必须返回失败 |
| `primary-drop-table.test` | drop table | 删除表及其全部关联资源（含索引） |
| `primary-update.test` | update | 带条件/不带条件的单字段更新 |
| `primary-date.test` | date 类型 | 日期字段的词法→执行全链路，含非法日期校验、闰年、索引比较 |
| `primary-select-tables.test` | 多表查询 | `select * from t1, t2` 笛卡尔积、带条件查询、表名限定列 |
| `primary-aggregation-func.test` | 聚合函数 | max/min/count/avg，`count(*)`/`count(1)`/`count(col)` |

### 选做题（计分功能）

| 测试用例 | 功能 | 说明 |
|---|---|---|
| `primary-join-tables.test` | INNER JOIN | 多表 join、多 on 条件、join+where 混用 |
| `primary-insert.test` | 多行插入 | 一条 insert 插入多行，同时成功或失败 |
| `primary-unique.test` | 唯一索引 | `create unique index`，重复插入失败 |
| `primary-null.test` | NULL 类型 | `nullable` 字段、IS NULL/IS NOT NULL、NULL 比较恒为 FALSE |
| `primary-simple-sub-query.test` | 简单子查询 | IN(NOT IN)、与子查询结果比较、子查询内聚合 |
| `primary-multi-index.test` | 多列索引 | `create index i on t(id, age)` |
| `primary-text.test` | TEXT 超长字段 | 4096 字节变长字段、超长截断 |
| `primary-expression.test` | 表达式 | where/select 中 +-*/ 算术运算、类型转换、除零按 NULL |
| `primary-complex-sub-query.test` | 复杂子查询 | 子查询多层嵌套、与父查询关联 |
| `primary-order-by.test` | ORDER BY | 升序/降序、多字段排序、NULL 排序处理 |
| `primary-group-by.test` | GROUP BY | 分组聚合、group by 多列、多表分组 |

### 2024 新增（向量化方向，非 2021 老题）

| 测试用例 | 功能 | 说明 |
|---|---|---|
| `vectorized-basic.test` | 向量 + 向量化执行 | 向量字段、`storage format=pax`、`set execution_mode='chunk_iterator'` |
| `vectorized-aggregation-and-group-by.test` | 向量化聚合/分组 | 在 chunk 执行模式 + PAX 存储下的聚合分组 |

---

## 二、涉及的知识点

1. **SQL 词法/语法解析**：flex/yacc（`yacc_sql.y`、`lex_sql.l`），AST 构建，语法报错统一输出 `FAILURE`（评测要求）
2. **类型系统**：int/char/float/date/text/vector，类型转换与比较规则（`common/type/`）
3. **表达式体系**：算术表达式、比较、逻辑与/或、函数调用、NULL 传播（`sql/expr/`）
4. **查询编译三阶段**：SQL → 逻辑计划 → 物理计划 → 执行
   - 语句处理（`sql/stmt/`）→ 逻辑计划生成（`optimizer/logical_plan_generator`）
   - 优化 rewrite（`optimizer/`）→ 物理计划生成（`optimizer/physical_plan_generator`）
5. **执行算子模型**：table_scan / index_scan / predicate / project / join / aggregate / group_by / order_by / update / delete / insert（`sql/operator/`）
6. **索引**：B+ 树（`storage/index/bplus_tree*`）、唯一索引、多列索引、NULL 键处理
7. **存储引擎**：buffer pool（LRU 淘汰）、record manager、变长 TEXT 记录、PAX 列存格式（`storage/buffer|record|table`）
8. **事务**：MVCC 事务模型（`storage/trx/`）
9. **视图**（进阶自研）：create/insert/update/select view，多表视图（`sql/`、`storage/table/table.cpp`）
10. **向量索引**：引入 annoy 库（`deps/3rd/annoy/`），向量距离检索（`storage/index/vector_index*`）
11. **向量化执行**（进阶）：chunk iterator 执行模式、`*_vec_physical_operator` 系列算子

---

## 三、如何完成赛题（实现思路）

### 总路线
miniob 官方代码已带基础 CRUD 和部分查询，赛题本质是在现成的
"解析 → 逻辑计划 → 物理计划 → 执行" 流水线上逐点打通。团队遵循开发规范：
**每次提交必须过编译**、提交前格式化、UBSan/ASan 全开辅助排错。

### 各模块的实现方式与代码对应

**1. 语法层（新增语法全靠这里）**
- 词法：`src/observer/sql/parser/lex_sql.l`
- 语法：`src/observer/sql/parser/yacc_sql.y`
- AST 结构：`src/observer/sql/parser/parse_defs.h`
- 生成脚本：`src/observer/sql/parser/gen_parser.sh`
- 思路：为每个新功能加语法规则 → 产出 AST 节点 → 在 `stmt/` 做语义检查转换。
  提交历史 `feat: 识别 ORDER BY 语法` → `feat: 实现聚合函数语法解析` 等即此过程。

**2. 表达式系统（支撑 where/select/聚合）**
- `src/observer/sql/expr/expression.cpp/h`：表达式类体系
- `arithmetic_operator.hpp`：算术运算符模板
- `tuple_cell`：值封装与类型比较
- 关键点：字符串/数字类型比较（`feat: 字符串和数字类型的比较`）、NULL 传播、
  表达式合法性校验（聚合与非聚合同列判断）

**3. 语句语义处理**
- `src/observer/sql/stmt/`：每个 SQL 语句一个 Stmt 类（如 `update_stmt`、`select_stmt`、
  `create_view_stmt`、`create_vector_index_stmt`），负责解析 AST、绑定表/列、检查元数据。

**4. 逻辑计划 + 优化**
- `src/observer/sql/optimizer/logical_plan_generator.cpp`：AST/Stmt → 逻辑算子树
- `rewriter.cpp` + `rewrite_rule.h`：规则化重写框架
- `predicate_pushdown_rewriter.cpp`：谓词下推（`feat: 实现 or 的逻辑下推逻辑`）
- `conjunction_simplification_rule.cpp` / `comparison_simplification_rule.cpp`：条件化简
- 提交 `fix: 暂时去掉 rewrite` / `feat: 重新恢复 rewrite` 说明团队在这块反复调优

**5. 物理计划 + 执行算子**
- `src/observer/sql/optimizer/physical_plan_generator.cpp`：逻辑 → 物理算子
- `src/observer/sql/operator/`：每个物理算子一个类，对应执行逻辑：
  - `join_physical_operator.cpp`：INNER JOIN
  - `aggregate_vec_physical_operator.cpp` + `aggregator.cpp` + `aggregate_hash_table.cpp`：聚合
  - `group_by_physical_operator.cpp` + `hash_group_by_physical_operator.cpp`：分组
  - `order_by_physical_operator.cpp`：排序（含 NULL 排序修复）
  - `index_scan_physical_operator.cpp` / `table_scan_physical_operator.cpp`：扫描路径选择
  - `predicate_physical_operator.cpp`：过滤
  - `update/delete/insert_physical_operator.cpp`：DML
  - `*_vec_physical_operator.cpp` 系列：chunk 模式向量化执行
- `sql/executor/`：DDL 类命令的直接执行器（create table/index/view、drop table 等）

**6. 索引**
- `src/observer/storage/index/bplus_tree.cpp` + `bplus_tree_index.cpp`：B+ 树实现与封装
- `vector_index.cpp` + `vector_index_meta.cpp`：向量索引（annoy 包装）
- 多列索引键构造（`feat: 初步实现 multi-index`）、唯一索引 NULL 处理
  （`feat: 支持 unique index 中 null 的重复插入`）

**7. 存储与类型**
- `src/observer/storage/buffer/`：buffer pool（LRU）
- `src/observer/storage/record/`：record manager（TEXT 变长支持）
- `src/observer/storage/table/table.cpp`：表元数据、增删索引、视图元数据
- `src/observer/common/type/`：`attr_type`、`date`、`text`、`vector` 类型实现
- PAX 列存：`storage format=pax` 解析与存储（向量题）

**8. 视图（团队自研进阶，超出必做）**
- `create_view_stmt.cpp`、`create_view_executor.cpp`、`view_physical_operator` 相关、
  `table.cpp` 的视图元数据；支持视图的 insert/update/select（提交历史多条 fix）

**9. 向量功能**
- 语法：`vector` 类型（`feat: 支持不加引号的向量定义`）
- 索引：`create vector index`（`feat: 向量索引`），底层 annoy（`deps/3rd/annoy/`）
- 算子：`vector_index_scan_physical_operator.cpp`、`expr_vec_physical_operator.cpp`
- 存储：PAX + chunk 执行模式（`set execution_mode='chunk_iterator'`）

### 完成节奏（可从 git 提交历史还原）
1. 先打通必做：update、date、drop table、多表查询、聚合
2. 再做选做：null → 子查询 → 表达式 → order by → group by → join → 多列/唯一索引 → text → 多行 insert
3. 中期引入 rewrite 优化并在聚合分组功能上反复调试
4. 后期实现视图、limit、update-select/create-table-select 等自选功能
5. 最后攻克 2024 向量化：vector 类型、annoy 向量索引、chunk/PAX 执行模式

---

## 四、快速查阅指引

| 想找 | 位置 |
|---|---|
| 赛题测试用例 | `test/case/test/*.test` |
| 期望输出 | `test/case/result/*.result` |
| 本地跑测试 | `python3 test/case/miniob_test.py` |
| 语法/词法 | `src/observer/sql/parser/` |
| 表达式 | `src/observer/sql/expr/` |
| 语句语义 | `src/observer/sql/stmt/` |
| 计划生成/优化 | `src/observer/sql/optimizer/` |
| 执行算子 | `src/observer/sql/operator/` |
| DDL 执行器 | `src/observer/sql/executor/` |
| 索引（B+树/向量） | `src/observer/storage/index/` |
| 存储（buffer/record/table/trx） | `src/observer/storage/` |
| 类型实现 | `src/observer/common/type/` |
| 向量检索库 | `deps/3rd/annoy/` |
