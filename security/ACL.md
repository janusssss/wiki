# ACL（Access Control List）访问控制列表

## 一、全称与定义

- **A**ccess **C**ontrol **L**ist，访问控制列表
- 每个**资源**挂一张清单：**谁**能对它做**什么**
- 直接维护"主体 ↔ 资源"的映射，不经过角色

## 二、原理

### 三元组

```
(subject, resource, action) → allow / deny
```

### 决策流程

```
请求(user U, 资源 R, 操作 A)
  → 查 ACL 表：R 的清单里有没有 (U 或 U 的组, A)
  → 命中 deny → 拒绝
  → 命中 allow → 放行
  → 都没命中 → 默认拒绝（deny by default）
```

### 与 RBAC 的区别

| 维度 | RBAC | ACL |
|------|------|-----|
| 权限挂载点 | 角色 | 资源 |
| 新增资源 | 通常自动继承角色权限 | 要新建清单 |
| "分享给某人" | 不擅长 | 天然支持 |
| 角色管理 | 复杂 | 简单 |

## 三、表设计

```sql
-- 资源表（可选，也可由业务表充当）
CREATE TABLE resource (
  id        BIGINT PRIMARY KEY AUTO_INCREMENT,
  type      VARCHAR(32),       -- doc / project / file
  owner_id  BIGINT,            -- 所有者
  name      VARCHAR(256)
);

-- ACL 核心表
CREATE TABLE acl_entry (
  id            BIGINT PRIMARY KEY AUTO_INCREMENT,
  resource_id   BIGINT NOT NULL,
  subject_type  VARCHAR(16),       -- user / group
  subject_id    BIGINT NOT NULL,
  action        VARCHAR(32),       -- read / write / share / manage
  effect        VARCHAR(8),        -- allow / deny
  inherit       TINYINT DEFAULT 1, -- 子资源是否继承
  UNIQUE KEY uk_acl (resource_id, subject_type, subject_id, action)
);

-- 组（避免条目爆炸）
CREATE TABLE acl_group (
  id   BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(64)
);

CREATE TABLE acl_group_member (
  group_id BIGINT,
  user_id  BIGINT,
  PRIMARY KEY (group_id, user_id)
);
```

核心就是 **`acl_entry` 一张表**。`subject_type` 区分"给个人"还是"给组"，`effect` 支持**显式拒绝**（deny 优先于 allow）。

## 四、实现

### 应用层查表（带缓存）

```go
func (s *Service) CanRead(ctx *gin.Context, resourceID, userID int64) (bool, error) {
    key := fmt.Sprintf("acl:%d:%d", resourceID, userID)
    if v, ok := cache.Get(key); ok {
        return v.(bool), nil
    }
    ok, err := repo.CheckACL(resourceID, userID, "read")
    cache.Set(key, ok, 10*time.Minute)
    return ok, err
}
```

### 组与继承

- 用户的 ACL = 用户直挂条目 ∪ 所属组的条目
- 目录类资源：子资源继承父资源清单，用 `inherit` 标志或物化到子资源

### 系统级 ACL

- **POSIX ACL**：`setfacl -m u:janus:rw file`，Linux 文件系统原生
- **Windows ACL**：资源管理器 → 安全 → 详细权限
- **对象存储**：S3 Bucket Policy、OSS ACL

## 五、常见坑

| 坑 | 现象 | 解法 |
|----|------|------|
| 条目爆炸 | 几万条 ACL 记录 | 引入组 + 继承 + 默认权限 |
| 默认放行 | 新建资源没配 ACL 就全员可读 | **deny by default**，没命中就拒绝 |
| 检查点分散 | 业务代码里到处调 Can() | 统一拦截器 / 装饰器收口 |
| 性能 | 每请求查 ACL 表 | 缓存 + 条目变更主动失效 |
| 与 RBAC 打架 | 两套规则结果冲突 | 明确分工：功能 RBAC、资源 ACL，deny 优先 |

## 六、适用与不适用

- ✅ 文档 / 项目 / 文件共享，"分享给某人 / 某组"是核心交互
- ✅ 资源数量少但权限差异大
- ❌ 功能权限为主、组织架构稳定的后台 —— 用 [RBAC](RBAC.md)
- ❌ 需要行级过滤 —— 叠加 [数据权限](数据权限.md)
