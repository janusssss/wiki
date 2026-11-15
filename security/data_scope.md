# 数据权限（Data Scope / 行级权限）

## 一、是什么

- 俗称**行级权限** / **数据范围**，英文 Data Scope
- 回答的问题：**这个用户能看到哪几行数据？**
- 功能权限管"能不能调这个接口"，数据权限管"调了之后返回哪几条"

典型场景：销售只能看自己的单，经理能看本部门的单，总监能看本部门及以下。

## 二、原理（一张表就够）

核心思路：**一张规则表 + 查询时拼 WHERE 条件**。

规则表记录"哪个角色，在哪张表上，用哪个字段，按什么范围过滤"。查询时根据当前用户的角色查规则，把规则翻译成 SQL 片段拼进 WHERE。

```
业务代码: db.Find(&orders)
实际执行: SELECT * FROM orders WHERE dept_id IN (1,2,3)   ← 规则自动拼的
```

范围类型做成枚举，5 种够用：

| scope_type | 含义 | 翻译成条件 |
|------------|------|------------|
| all | 全部数据 | 不加条件 |
| self | 仅本人 | `column = 当前用户ID` |
| dept | 本部门 | `column = 用户部门ID` |
| dept_and_below | 本部门及以下 | `column IN (部门子树ID)` |
| custom | 自定义 | `column IN (配置的值)` |

## 三、表设计

只需要一张表：

```sql
CREATE TABLE role_table_rule (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  role_id     BIGINT      NOT NULL,        -- 角色
  table_name  VARCHAR(64) NOT NULL,        -- 业务表名，如 orders
  column_name VARCHAR(64) NOT NULL,        -- 过滤字段，如 dept_id / create_by
  scope_type  VARCHAR(16) NOT NULL,        -- all / self / dept / dept_and_below / custom
  scope_value VARCHAR(512),                -- custom 时存 "1,2,3"
  UNIQUE KEY uk_role_table (role_id, table_name)
);
```

一个角色对一张表只有一条规则。用户可能有多个角色，查询时把所有角色的规则取出来，用 **OR** 合并（取并集，最宽松的生效）。

> 业务表本身要带过滤字段（`dept_id`、`create_by`），这是前提，不是规则表管的事。

## 四、Go + GORM 实现

### 1. 模型与上下文

```go
type ScopeType string

const (
    ScopeAll          ScopeType = "all"
    ScopeSelf         ScopeType = "self"
    ScopeDept         ScopeType = "dept"
    ScopeDeptAndBelow ScopeType = "dept_and_below"
    ScopeCustom       ScopeType = "custom"
)

// 规则表对应的模型
type RoleTableRule struct {
    ID         int64     `gorm:"primaryKey"`
    RoleID     int64     `gorm:"uniqueIndex:uk_role_table"`
    TableName  string    `gorm:"size:64;uniqueIndex:uk_role_table"`
    ColumnName string    `gorm:"size:64"`
    ScopeType  ScopeType `gorm:"size:16"`
    ScopeValue string    `gorm:"size:512"`
}

// 当前登录用户，从 JWT 解出来塞进 gin.Context
type LoginUser struct {
    UserID   int64
    DeptID   int64
    RoleIDs  []int64
    DeptPath string // 祖先链 "0,1,3"，子树用
}
```

### 2. 规则查询（带缓存）

规则表数据量小、变更少，启动时全量加载到内存或 Redis，避免每次查询都读库：

```go
// key = "roleID:tableName"，value = RoleTableRule
var ruleCache sync.Map

func getRules(roleIDs []int64, table string) []RoleTableRule {
    var rules []RoleTableRule
    for _, rid := range roleIDs {
        key := fmt.Sprintf("%d:%s", rid, table)
        if v, ok := ruleCache.Load(key); ok {
            rules = append(rules, v.(RoleTableRule))
        }
    }
    if len(rules) > 0 {
        return rules
    }
    // 缓存未命中，回源（生产环境改成预加载）
    db.Where("role_id IN ? AND table_name = ?", roleIDs, table).Find(&rules)
    return rules
}
```

### 3. 核心：Scope 函数

GORM 的 `Scopes` 接收一个 `func(*gorm.DB) *gorm.DB`，在 `Find` 前往 Statement 上追加条件，最终拼进 SQL。这就是原理落点：

```go
// DataScope 返回一个 GORM Scope，查询时链上即可自动加数据范围
func DataScope(u *LoginUser, table string) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        rules := getRules(u.RoleIDs, table)
        if len(rules) == 0 {
            return db.Where("1 = 0") // 没配规则默认拒绝，安全兜底
        }

        // 多角色规则取并集：WHERE (cond1 OR cond2 OR ...)
        var conds []string
        var args  []any
        for _, r := range rules {
            switch r.ScopeType {
            case ScopeAll:
                return db // 有一条 all 就全放行，不用拼了
            case ScopeSelf:
                conds = append(conds, r.ColumnName+" = ?")
                args = append(args, u.UserID)
            case ScopeDept:
                conds = append(conds, r.ColumnName+" = ?")
                args = append(args, u.DeptID)
            case ScopeDeptAndBelow:
                conds = append(conds, r.ColumnName+" IN ?")
                args = append(args, subtreeIDs(u.DeptPath))
            case ScopeCustom:
                conds = append(conds, r.ColumnName+" IN ?")
                args = append(args, parseIDs(r.ScopeValue))
            }
        }
        // 用 raw 拼条件，args 走预编译，防注入
        expr := strings.Join(conds, " OR ")
        return db.Where(expr, args...)
    }
}

// 祖先链 "0,1,3" → 本部门及所有子部门 ID
// 简单做法：dept_id IN (SELECT id FROM dept WHERE FIND_IN_SET(?, ancestors) OR id = ?)
// 这里用内存预算好的子树，避免每次子查询
func subtreeIDs(deptPath string) []int64 { /* 略 */ return nil }
```

### 4. 用法

```go
// 从 context 拿登录用户
u := c.MustGet("user").(*LoginUser)

var orders []Order
db.Scopes(DataScope(u, "orders")).Find(&orders)
// → SELECT * FROM orders WHERE dept_id IN (1,2,3)

// 分页、排序照常链
db.Scopes(DataScope(u, "orders")).
    Where("status = ?", 1).
    Order("id desc").
    Find(&orders)
```

业务代码只多了一个 `.Scopes(...)`，过滤逻辑全收口在 `DataScope` 函数里。

## 五、原理要点回顾

1. **规则表**：`role_table_rule` 存"角色×表×字段×范围"，是唯一的配置数据源。
2. **查询改写**：GORM Scope 在 `Find` 前往 Statement 追加 WHERE，最终拼进 SQL —— 等价于 MyBatis 拦截器改写 BoundSql，只是 GORM 用函数式 Scope 更直观。
3. **多角色并集**：一个用户多个角色，规则取 OR，最宽松的生效。
4. **默认拒绝**：查不到规则就 `WHERE 1=0`，宁可看不见也别泄露。
5. **字段名安全**：`column_name` 来自规则表（管理员配置），不是用户输入，拼进 SQL 安全；值一律走预编译 `?`。

## 六、常见坑

| 坑 | 现象 | 解法 |
|----|------|------|
| 表名对不上 | GORM 默认表名是复数蛇形（`Order` → `orders`），规则表里写错就匹配不到 | 规则表存实际表名，或统一实现 `TableName()` |
| 聚合统计漏过滤 | 列表过滤了，`Count` / 首页卡片统计却是全量 | 统计查询也要 `.Scopes(DataScope(...))` |
| JOIN 表没过滤 | 主表加了条件，JOIN 的明细表还是全量 | 每张需要过滤的表都单独加 Scope，或用子查询 |
| 缓存不刷新 | 改了规则没生效 | 规则变更后清 `ruleCache`，或加版本号短 TTL |
| 慢查询 | `IN` 上千个 ID | 转成 `JOIN 子查询`，或 `column IN (SELECT ...)` |
| 业务表忘加字段 | 没有过滤字段，Scope 拼了条件也过滤不了 | 建表规范强制 `dept_id` / `create_by` |

## 七、关联

- 想要数据库层强制过滤（应用绕不过）→ [RLS](RLS.md)
- 功能权限（能不能调接口）→ [RBAC](RBAC.md)
- 总览 → [security.md](security.md)
