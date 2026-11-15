# RLS（Row Level Security）行级安全

## 一、全称与定义

- **R**ow **L**evel **S**ecurity，行级安全策略
- **PostgreSQL 原生特性**（9.5+），在**数据库层**强制行过滤
- 关键优势：应用层绕不过去（除非表 owner 或 `BYPASSRLS` 角色），是权限的**最后防线**

## 二、原理

1. 表上启用 RLS：`ALTER TABLE ... ENABLE ROW LEVEL SECURITY`。
2. 定义 **policy**：`USING` 子句写行可见条件，`WITH CHECK` 写行可写条件。
3. 用户查询时，PostgreSQL **自动**把 policy 条件 AND 进 SQL，等价于：

```sql
SELECT * FROM orders
WHERE <用户原本的条件> AND <policy 的 USING 条件>;
```

4. 条件里的值从**会话变量**取（`current_setting`），应用每次请求前 `SET`。

### 关键概念

| 概念 | 说明 |
|------|------|
| policy | 一条"谁能看哪些行"的规则，可多条叠加 |
| USING | SELECT / UPDATE / DELETE 的行过滤 |
| WITH CHECK | INSERT / UPDATE 时新行必须满足的条件 |
| 表 owner | 默认**绕过** RLS，可用 `FORCE ROW LEVEL SECURITY` 强制 |
| BYPASSRLS | 角色属性，运维/备份用 |

## 三、"表"设计

RLS 不新建表，靠**业务表字段 + 会话变量 + policy**。业务表通常带 `tenant_id` 或 `dept_id` / `create_by`：

```sql
CREATE TABLE orders (
  id         BIGSERIAL PRIMARY KEY,
  tenant_id  INT NOT NULL,
  dept_id    BIGINT,
  create_by  BIGINT,
  amount     NUMERIC(12,2)
);
```

## 四、实现

### 启用 RLS + 建 policy

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- 表 owner 也受限

-- 多租户：按 tenant_id 隔离
CREATE POLICY orders_tenant ON orders
  FOR ALL
  USING (tenant_id = current_setting('app.tenant_id')::int)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);

-- 或按用户 + 部门树
CREATE POLICY orders_scope ON orders
  FOR SELECT
  USING (
    create_by = current_setting('app.user_id')::bigint
    OR dept_id IN (
      SELECT id FROM dept_tree(current_setting('app.dept_id')::bigint)
    )
  );
```

### 应用侧：每事务设置会话变量

```sql
BEGIN;
SET LOCAL app.tenant_id = '42';
SET LOCAL app.user_id   = '1001';
SELECT * FROM orders;      -- 自动只返回 tenant_id=42 的行
COMMIT;
```

Go 示例：

```go
tx := db.Begin()
tx.Exec("SET LOCAL app.tenant_id = $1", tenantID)
tx.Exec("SET LOCAL app.user_id = $1", userID)
tx.Find(&orders)
tx.Commit()
```

### MySQL 怎么办

MySQL **没有 RLS**，只能模拟：

- 每个租户/范围建视图，应用切视图
- 代理层（ProxySQL）改写 SQL
- 应用层拦截器拼条件（回到 [数据权限](数据权限.md)）

## 五、常见坑

| 坑 | 现象 | 解法 |
|----|------|------|
| 连接池会话变量残留 | 上个请求的 tenant 泄漏给下个请求 | **必须 `SET LOCAL` + 显式事务**，不要 `SET SESSION` |
| 表 owner 绕过 RLS | 自己查全表没事，别人查被过滤 | `ALTER TABLE ... FORCE ROW LEVEL SECURITY` |
| 调试懵 | SQL 里看不到过滤条件，还以为数据丢了 | `EXPLAIN` 看 plan；先 `SET ROLE` 普通用户再查 |
| 性能 | policy 子查询每行求值 | 过滤字段加索引；复杂 policy 物化成字段 |
| 跨表 JOIN 漏过滤 | 另一张表没启用 RLS | 所有敏感表都启用并建 policy |
| 迁移痛苦 | 换数据库（MySQL）不支持 | 提前评估，或把 policy 逻辑抽成应用层可移植实现 |

## 六、适用与不适用

- ✅ 多租户 SaaS 强隔离（数据库层兜底）
- ✅ 合规要求"应用层无法越权访问"
- ✅ 用 PostgreSQL 且团队熟悉 SQL
- ❌ 需要跨库 / 多数据源统一鉴权 —— 用 [ABAC](ABAC.md) 或应用层 [数据权限](数据权限.md)
- ❌ 小项目、快速迭代 —— 运维成本不划算
