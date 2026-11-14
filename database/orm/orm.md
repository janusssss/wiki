# ORM 知识点

## 一、ORM 是什么

ORM（Object-Relational Mapping，对象关系映射）：把数据库表映射为程序里的类/结构体，把行映射为对象，用面向对象的方式操作数据库，自动生成 SQL。

价值：
- 用类型语言操作数据，减少手写 SQL 的拼写错误
- 数据库方言隔离（MySQL / PostgreSQL / SQLite 可切换）
- 自带迁移、关联、验证等基础设施

代价：
- 隐藏了 SQL，复杂查询和性能调优仍需理解底层
- 存在对象-关系阻抗失配（Impedance Mismatch）

## 二、核心映射概念

| 概念 | 说明 |
|------|------|
| Model | 一个类/结构体对应一张表 |
| 主键 | 标识行，默认 `id`，复合主键支持较弱 |
| 字段映射 | 属性名 ↔ 列名（常按命名策略转换，如驼峰转蛇形） |
| 关联 | 外键关系映射为对象引用/集合 |
| 脏检查 | 变更追踪，提交时只写改动的列 |
| 会话/工作单元 | 一级缓存与事务边界，跟踪本次操作的对象 |
| 二级缓存 | 跨会话缓存，性能高但有一致性风险 |

## 三、关联关系类型

- **一对一**：外键在任意一表（如 User ↔ Profile）
- **一对多**：一方持外键（如 User → Orders）
- **多对多**：需要中间表（如 Student ↔ Course）

加载策略：
- **立即加载（Eager）**：查询时一次 JOIN/子查询取全部关联，避免 N+1
- **延迟加载（Lazy）**：访问属性时才发 SQL，方便但易产生 N+1

## 四、ORM 必知的问题

### 1. N+1 查询问题

先查 100 条主记录，再循环里各查一次关联 → 1 + 100 条 SQL。
解决：`Preload` / `JOIN` / `IN` 批量查询后内存组装。

### 2. 脏 SQL 与注入

ORM 的参数占位（`?` / `$1`）天然防注入；但**拼接字符串条件**（如直接拼列名、排序字段）会绕过防护，用户输入必须白名单。

### 3. 阻抗失配

对象有继承、多态、嵌套集合；关系表是扁平的。映射不完美时会出现：
- 继承映射三种策略：单表（Joined）、类表、具体表
- 值类型 vs 引用类型
- NULL 与零值的语义差异（Go 的 `0` vs SQL `NULL`）

### 4. 事务与并发

- 会话通常非线程安全，不能跨 goroutine/线程共享一个 Session
- 长事务 + ORM 懒加载 = 锁持有时间变长
- 乐观锁（version 字段）与悲观锁（`SELECT FOR UPDATE`）支持视框架而定

### 5. 性能陷阱

- 全字段查询 vs `Select` 指定列
- 分页在深页（`OFFSET 100000`）会退化，改用游标（`WHERE id > last_id ORDER BY id LIMIT n`）
- 批量操作循环单条执行，应改批量 API

## 五、迁移（Migration）

ORM 通常内置迁移工具（如 Alembic / golang-migrate / TypeORM）：
- 根据 Model 变更生成 SQL 变更脚本
- 支持升级/回滚（Up/Down）
- 生产环境建议审查生成的 SQL，大表改列型需手动干预（如 MySQL 8 的 instant DDL）

## 六、常见 ORM 对比

| 生态 | ORM | 特点 |
|------|-----|------|
| Go | GORM | 生态最大，链式 API，钩子丰富 |
| Go | sqlx / ent | sqlx 轻量手写 SQL；ent Facebook 出品，强 schema |
| Java | Hibernate / JPA | 全功能，一级/二级缓存，重量级 |
| Python | SQLAlchemy | 事实标准，Core + ORM 双模式 |
| Node | TypeORM / Prisma | TypeORM 装饰器风格；Prisma 自研 DSL + 引擎 |
| Rust | Diesel / SeaORM | Diesel 编译期 SQL 校验；SeaORM 异步 |

## 七、什么时候不该用 ORM

- 大数据分析/报表：大量聚合与自定义 SQL，ORM 反而碍事
- 极致性能的热点路径：用存储过程或原生 SQL
- 高度非规范化 schema

实践中常见折中：主体业务用 ORM + 复杂查询用原生 SQL（ORM 一般提供 `Raw` / `Exec` 逃生舱）。

## 八、总结

ORM 解决的是**开发效率与可维护性**，不是性能。
用好 ORM 的关键是：理解它生成的 SQL，警惕 N+1、深分页、长事务三个性能杀手，必要时用原生 SQL 逃生。
