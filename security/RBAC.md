# RBAC（Role-Based Access Control）基于角色的访问控制

## 一、全称与定义

- **R**ole-**B**ased **A**ccess **C**ontrol，基于角色的访问控制
- 核心三层：**用户（User）→ 角色（Role）→ 权限（Permission）**
- 用户不直接绑权限，绑角色；权限挂在角色上。人换岗位 = 换角色，不用改权限。

## 二、原理

四个基本实体：

- **Subject 主体**：用户 / 服务账号
- **Role 角色**：一组职责的抽象（管理员、运营、只读用户）
- **Permission 权限**：对资源的一个操作（`user:add`、`order:export`）
- **Object 资源**：被保护的对象（接口、菜单、按钮）

授权链路：

```
用户 --N:M--> 角色 --N:M--> 权限 --N:1--> 资源
```

鉴权时只问一句：**用户的角色集合展开后的权限集合里，有没有当前要求的权限码？**

### NIST RBAC 分级

| 级别 | 名称 | 能力 |
|------|------|------|
| RBAC0 | 基础 | 三张核心表 + 两张关联表 |
| RBAC1 | 角色继承 | 角色分层级（admin 继承 user 的全部权限） |
| RBAC2 | 约束 | 互斥角色、角色上限、先决条件 |
| RBAC3 | 综合 | RBAC1 + RBAC2 |

## 三、表设计

```sql
-- 用户表
CREATE TABLE sys_user (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  username    VARCHAR(64) NOT NULL,
  password    VARCHAR(128) NOT NULL,
  status      TINYINT DEFAULT 1,          -- 0 禁用 1 正常
  dept_id     BIGINT,                      -- 数据权限用，见《数据权限》
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 角色表
CREATE TABLE sys_role (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  code        VARCHAR(64) NOT NULL,        -- 如 ROLE_ADMIN
  name        VARCHAR(64) NOT NULL,
  parent_id   BIGINT DEFAULT 0,            -- RBAC1 角色继承
  data_scope  TINYINT DEFAULT 1,           -- 数据范围，见《数据权限》
  sort        INT DEFAULT 0
);

-- 权限表（资源 + 操作）
CREATE TABLE sys_permission (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  parent_id   BIGINT DEFAULT 0,            -- 目录 → 菜单 → 按钮
  type        TINYINT,                     -- 1 目录 2 菜单 3 按钮 4 接口
  name        VARCHAR(64),
  perms       VARCHAR(128),                -- 权限码 user:add
  path        VARCHAR(128)                 -- 路由或接口路径
);

-- 用户-角色
CREATE TABLE sys_user_role (
  user_id BIGINT,
  role_id BIGINT,
  PRIMARY KEY (user_id, role_id)
);

-- 角色-权限
CREATE TABLE sys_role_permission (
  role_id       BIGINT,
  permission_id BIGINT,
  PRIMARY KEY (role_id, permission_id)
);
```

核心就是**五张表**：三张实体表 + 两张 N:M 关联表。要角色继承就加 `parent_id`，要部门数据范围就在 `sys_role` 上加 `data_scope`。

## 四、实现

### Java（Spring Security）

```java
// 自定义 UserDetailsService，登录时查出权限码集合
@Override
public UserDetails loadUserByUsername(String username) {
    User u = userMapper.selectByUsername(username);
    List<String> perms = permMapper.selectByUserId(u.getId()); // 权限码
    List<SimpleGrantedAuthority> auths = perms.stream()
        .map(SimpleGrantedAuthority::new).toList();
    return new org.springframework.security.core.userdetails.User(
        u.getUsername(), u.getPassword(), auths);
}

// 方法级校验
@PreAuthorize("hasAuthority('user:add')")
public void addUser(User u) { ... }
```

### Go（中间件校验）

```go
func Require(perm string) gin.HandlerFunc {
    return func(c *gin.Context) {
        perms := c.GetStringSlice("perms") // 登录时写入 token/redis
        for _, p := range perms {
            if p == perm || p == "*:*:*" {
                c.Next()
                return
            }
        }
        c.AbortWithStatusJSON(403, gin.H{"msg": "无权限"})
    }
}

r.POST("/user", auth, Require("user:add"), handler)
```

### 通用套路

1. 登录成功：查用户全部权限码 → 存 Redis（key = `perm:{userId}`）+ JWT 里只放 userId。
2. 请求进来：拦截器解析 token → 取权限集合 → 匹配接口要求的权限码。
3. 改权限（换角色 / 改角色绑定）：删缓存，下次请求回源。

## 五、常见坑

| 坑 | 现象 | 解法 |
|----|------|------|
| 角色爆炸 | 角色数量上百，每来个人就新建一个 | 角色收敛 + 权限细分，用"用户-权限"直绑做兜底 |
| 权限缓存不刷新 | 改了权限没生效 | 变更后主动 DEL，或加版本号短 TTL |
| 水平越权 | A 用户改到了 B 的订单 | RBAC 管不到行级，必须叠加 [数据权限](data_scope.md) |
| 菜单权限 ≠ 接口权限 | 前端藏了按钮但接口直连可调 | 前后端都要校验，后端是唯一可信点 |
| 权限码硬编码 | 改个字符串全项目搜 | 权限码常量类 / 枚举统一管理 |

## 六、适用与不适用

- ✅ 组织架构稳定、职责清晰的后台管理系统
- ❌ 权限跟上下文（时间、IP、设备、数据归属）相关的场景 → 用 [ABAC](ABAC.md)
- ❌ 资源需要"分享给某个具体的人" → 叠加 [ACL](ACL.md)
