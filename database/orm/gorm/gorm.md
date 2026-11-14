# GORM 知识点

## 一、GORM 是什么

Go 语言最流行的 ORM 库，将结构体与数据库表映射，用面向对象方式操作数据库。

- 作者：jinzhu（现由社区维护）
- 支持数据库：MySQL、PostgreSQL、SQLite、SQL Server，另有ClickHouse等第三方驱动
- 当前主线：v2（`gorm.io/gorm`），v1（`github.com/jinzhu/gorm`）已停止维护，API 不兼容

```go
import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

## 二、核心概念

### 1. Model 与约定

结构体即表，默认表名是复数蛇形（`User` → `users`），主键默认 `ID`。

```go
type User struct {
    gorm.Model          // 内嵌：ID、CreatedAt、UpdatedAt、DeletedAt
    Name string
    Age  int
}
```

`gorm.Model` 内含 `DeletedAt`，开启**软删除**：`Delete` 只是 `UPDATE deleted_at=now()`，查询默认过滤已删数据。

### 2. 命名策略

字段名默认转蛇形（`CreatedAt` → `created_at`），可全局改或用 tag 精确控制：

```go
type User struct {
    Name   string `gorm:"column:user_name"`
    Email  string `gorm:"uniqueIndex:idx_email"`
    Age    int    `gorm:"default:18"`
}
```

常用 tag：`column`、`primaryKey`、`autoIncrement`、`uniqueIndex`、`index`、`not null`、`default`、`type:varchar(64)`、`-"`（忽略字段）。

## 三、CRUD 操作

### 创建

```go
user := User{Name: "Janus", Age: 18}
db.Create(&user)               // 返回后 user.ID 自动回填

var users = []User{{Name: "a"}, {Name: "b"}}
db.Create(&users)              // 批量创建
db.Create(&user, db.Session(&gorm.Session{SkipHooks: true})) // 跳过钩子
```

### 查询

```go
db.First(&user, 10)                    // SELECT * FROM users WHERE id=10 LIMIT 1
db.First(&user, "name = ?", "Janus")   // 条件查询
db.Find(&users, db.Where("age > ?", 18).Order("id desc")) // 链式条件

db.Model(&User{}).Where("age > ?", 18).Count(&total)  // 计数
db.Model(&User{}).Select("name, age").Find(&users)    // 指定列
db.Preload("Orders").Find(&users)                     // 预加载关联（避免 N+1）
db.Limit(10).Offset(20).Find(&users)                  // 分页
```

### 更新

```go
db.Model(&user).Update("name", "Janus2")      // 单字段（零值也会更新）
db.Model(&user).Updates(User{Name: "x", Age: 0}) // struct 更新会跳过零值
db.Model(&user).Updates(map[string]interface{}{"name": "x", "age": 0}) // map 更新含零值
db.Model(&User{}).Where("age > ?", 18).UpdateColumn("status", 1) // 批量更新，必须带条件
```

⚠️ 防误操作：`Model` 不带 `Where` 的批量更新会报 `ErrMissingWhereClause`，需显式 `AllowGlobalUpdate`。

### 删除

```go
db.Delete(&user, 10)               // 软删除
db.Unscoped().Delete(&user, 10)    // 物理删除
db.Unscoped().Where("age < 18").Delete(&User{}) // Unscoped 查 + 删都绕过软删除
```

## 四、关联关系

| 关系 | 字段类型 | 示例 |
|------|---------|------|
| 一对多 | `[]` | 一个用户有多个订单 `Orders []Order` |
| 多对一 | 外键指针 | 订单属于用户 `UserID uint` |
| 多对多 | 中间表 | 多个角色多对多用户 `Roles []Role gorm:"many2many:user_roles"` |
| 拥有者 | `*` | 一对一，外键在本表 |

关联操作：

```go
db.Preload("Orders").Find(&users)        // 预加载（单独 SQL 查关联，避免 N+1）
db.Create(&User{Name: "a", Orders: []Order{{No: "1"}}}) // 级联创建
db.Model(&user).Association("Roles").Append([]Role{r1}) // 追加关联
db.Model(&user).Association("Roles").Clear()  // 清空关联
```

## 五、钩子（Hooks）

模型可实现钩子方法，在生命周期特定时机执行：

- `BeforeSave` / `AfterSave`
- `BeforeCreate` / `AfterCreate`
- `BeforeUpdate` / `AfterUpdate`
- `BeforeDelete` / `AfterDelete`
- `AfterFind`

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.Age < 0 { return errors.New("age must > 0") }
    u.Password = hash(u.Password)
    return nil
}
```

⚠️ 钩子里的写操作要通过传入的 `tx *gorm.DB`，才能保证在**同一事务**内。

## 六、事务

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil { return err }
    if err := tx.Model(&user).Update("balance", gorm.Expr("balance - ?", 100)).Error; err != nil {
        return err
    }
    return nil // 返回 nil 自动 Commit，返回 error 自动 Rollback
})
```

手动控制：

```go
tx := db.Begin()
tx.Create(&order)
tx.Commit() // 或 tx.Rollback()
```

⚠️ 事务里必须用 `tx`，继续用外层 `db` 会在事务外执行，数据不一致。

## 七、原生 SQL 与性能

```go
db.Raw("SELECT * FROM users WHERE age > ?", 18).Scan(&users)
db.Exec("UPDATE users SET age = ? WHERE id = ?", 20, 1)
```

避免 N+1 查询：用 `Preload` 或 `Joins` 代替循环里单查。

```go
db.Joins("JOIN orders ON orders.user_id = users.id").Where("orders.no = ?", "1").Find(&users)
```

## 八、连接池

底层复用 `database/sql` 的连接池：

```go
sqlDB, _ := db.DB()
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

## 九、常见坑

1. **零值更新**：`Updates(struct)` 跳过零值字段，想更新为 0 用 `map` 或 `gorm.Expr`
2. **`Find` 没条件**：批量操作忘了 `Where` 会被保护机制拦截（除非开 `AllowGlobalUpdate`）
3. **N+1 查询**：循环里逐条查关联数据，要用 `Preload` 一次查出
4. **事务用错句柄**：事务内继续用外层 `db` 而不是 `tx`
5. **软删除残留唯一索引**：软删除的行还在表里，唯一索引可能冲突
6. **`Scan` vs `Find`**：`Raw/Exec` 结果要 `Scan` 到结构体；`Find` 自动扫描
7. **`Select` 聚合要用 `Pluck` 或 `Select` + `Scan`**，`Find` 不适合取单个聚合值

## 十、总结

GORM 让 Go 开发者用结构体思维操作数据库，开发效率高、迁移和关联支持完善。
但它是"半自动"的 ORM：复杂查询、性能敏感处仍需手写 SQL 或用 `Raw`。
生产上最常踩的坑是**零值更新**和**事务句柄用错**。

---

## 十一、常见面试问题

### Q1：GORM v1 和 v2 有什么区别？

- 包路径：v1 `github.com/jinzhu/gorm`，v2 `gorm.io/gorm`
- v2 性能更好（重写了上下文和链式 API）、支持 `context.Context`、改进错误处理（`gorm.ErrRecordNotFound` 等）
- v2 钩子签名改为接收 `*gorm.DB`，事务 API 重写
- 两者 API 不兼容，不能混用

### Q2：GORM 怎么做软删除？原理是什么？

Model 内嵌 `gorm.Model`（含 `DeletedAt gorm.NullTime`）或自己定义带 `DeletedAt` 的字段。
`Delete` 时生成 `UPDATE ... SET deleted_at = now()` 而不是 `DELETE`；查询时 GORM 自动追加 `WHERE deleted_at IS NULL`。
用 `Unscoped()` 可以绕过软删除（物理删除、查询含已删记录）。
代价：表会越来越大，且软删除的行仍占唯一索引，需要定期归档或用 `UniqueIndex` 组合列规避。

### Q3：GORM 的 N+1 查询问题是什么？怎么解决？

循环里逐个查关联数据，比如先查 100 个用户，再循环里各查一次订单 → 1 + 100 条 SQL。
解决：
- `Preload("Orders")`：分两条 SQL 一次查出所有关联
- `Joins` 联表查询
- 手动按 ID 批量查再内存组装

### Q4：`Updates` 传 struct 和传 map 有什么区别？

- 传 `struct`：只更新**非零值**字段（`Age: 0` 不会被更新）
- 传 `map`：更新所有指定键，包括零值
- 想用 struct 风格但要更新零值，用 `gorm:"default:..."` 或 `Select` 指定列 + `map`

### Q5：GORM 的事务怎么用？要注意什么？

`db.Transaction(func(tx *gorm.DB) error {...})`，返回 nil 自动提交，返回 error 自动回滚。
注意事项：事务内所有操作必须用传入的 `tx`，用外层 `db` 会脱离事务；钩子内的写操作也要走 `tx`。
支持嵌套事务（SavePoint 实现）。

### Q6：GORM 的钩子有哪些？执行顺序？

创建：`BeforeCreate` → DB 操作 → `AfterCreate`（`BeforeSave/AfterSave` 包裹前后）
查询：查询后 `AfterFind`
更新：`BeforeUpdate` → `AfterUpdate`
删除：`BeforeDelete` → `AfterDelete`
典型用途：自动填充字段（时间戳、UUID）、密码加密、数据校验。

### Q7：GORM 怎么做批量插入？有什么要注意的？

`db.Create(&slice)` 传切量，GORM 默认分批（`CreateBatchSize` 可配，`gorm.Config{CreateBatchSize: 500}`）。
注意：批量插入**不会**给每个记录回填自增 ID（MySQL 不支持多值 `lastInsertId`），且每条记录的钩子都会执行。

### Q8：GORM 和原生 SQL 相比有什么性能开销？

- 反射：把结果集映射到结构体用反射，比手写 `Scan` 到结构体的原生写法稍慢
- SQL 构建：链式调用最终拼 SQL 字符串，本身开销小
- 真正的性能差异通常在**查询次数**（N+1）和**索引使用**，而非 ORM 本身
- 热点路径可用 `Raw`/`Exec` 写原生 SQL，或用 `Select` 只取需要的列

### Q9：如何在 GORM 里使用 `context.Context`？

```go
db.WithContext(ctx).Find(&users)
```
`WithContext` 返回带 ctx 的新句柄，支持超时取消、链路追踪（配合 OpenTelemetry 的 gorm 插件）。
HTTP handler 里推荐统一传 ctx，避免慢查询拖垮请求。
