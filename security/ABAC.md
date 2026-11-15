# ABAC（Attribute-Based Access Control）基于属性的访问控制

## 一、全称与定义

- **A**ttribute-**B**ased **A**ccess **C**ontrol，基于属性的访问控制
- 不看"你是谁、你什么角色"，看**属性**：主体、资源、操作、环境
- 决策 = 对一组**策略（Policy）**求值

## 二、原理

### 四要素

| 要素 | 英文 | 属性举例 |
|------|------|----------|
| 主体 | Subject | 部门、职级、安全等级、是否在职 |
| 资源 | Resource | 所属部门、密级、创建人、文档类型 |
| 操作 | Action | read / write / delete / share |
| 环境 | Environment | 时间、IP、设备指纹、地理位置、风控分 |

### 决策模型

```
请求 (subject, resource, action, environment)
        ↓
   策略求值引擎 (PDP)
        ↓
   Permit / Deny / NotApplicable / Indeterminate
```

### 策略示例

```text
permit when
  subject.dept == resource.dept
  AND subject.level >= resource.classification
  AND environment.time BETWEEN 09:00 AND 18:00
  AND environment.ip IN corporate_network
```

经典公式：**"谁能（subject）在什么条件下（environment）对什么（resource）做什么（action）"**。

## 三、表设计

ABAC 的属性大多来自已有实体（用户表、资源表），**核心是策略表**：

```sql
CREATE TABLE abac_policy (
  id          BIGINT PRIMARY KEY AUTO_INCREMENT,
  name        VARCHAR(128),
  effect      VARCHAR(8),        -- permit / deny
  priority    INT DEFAULT 0,     -- 冲突时优先级
  target      VARCHAR(256),      -- 适用范围表达式（可选）
  rule_expr   TEXT,              -- 策略表达式
  status      TINYINT DEFAULT 1,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 可选：策略与资源/主体绑定（避免 target 表达式太重）
CREATE TABLE abac_policy_binding (
  policy_id    BIGINT,
  subject_type VARCHAR(32),      -- role / user / dept
  subject_id   BIGINT,
  resource_type VARCHAR(32)
);
```

属性本身不建表，从 `sys_user`、业务表、请求上下文里取，组装成**属性包**交给引擎。

## 四、实现

### 方案 1：OPA（Open Policy Agent）

```rego
package authz

default allow = false

allow {
    input.action == "read"
    input.subject.dept == input.resource.dept
    input.subject.level >= input.resource.classification
}
```

- 独立部署（sidecar / DaemonSet），通过 REST 或 gRPC 调用
- 策略热更新，无需发版
- K8s 准入、API 网关、微服务都能接

### 方案 2：Casbin

```text
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub_rule, obj, act, eft

[matchers]
m = eval(p.sub_rule) && r.obj == p.obj && r.act == p.act
```

- 语言无关（Go / Java / Node 都有 SDK），可内嵌进程内，性能好
- 支持 ABAC 的 `eval(p.sub_rule)`，属性直接来自 `r.sub` 对象字段

### 方案 3：自研表达式引擎

用 SpEL（Java）、expr（Go）、cel-go 解析 `rule_expr`，属性包组装后求值。

```go
env := map[string]any{
    "subject":  sub, "resource": res,
    "action":   act, "env":      ctx,
}
program, _ := expr.Compile(policy.RuleExpr, expr.Env(env))
out, _ := expr.Run(program, env)   // bool
```

规则可控、简单场景够用，但要自己做策略管理界面 + 审计。

### 性能要点

- 属性来源多，必须**缓存**（用户属性、资源属性、策略表达式编译结果）
- 策略求值一般在网关 / 服务层做，别每次查库

## 五、常见坑

| 坑 | 现象 | 解法 |
|----|------|------|
| 策略黑洞 | 规则多了没人知道最终结论 | 策略管理界面 + 冲突分析 + 优先级 |
| 属性不一致 | 同一属性从三张表取出来不一样 | 属性标准化，统一 attribute provider |
| 调试困难 | 拒绝了不知道为什么 | 引擎返回决策理由（explain） |
| 表达式注入 | 用户输入进了规则表达式 | 规则与数据严格分离，属性只做数据 |
| 性能 | 每请求查库取属性 | 属性缓存 + 策略编译缓存 |

## 六、适用与不适用

- ✅ 策略多变、维度多（时间 / IP / 风控 / 数据密级）
- ✅ 零信任架构、API 网关统一鉴权
- ❌ 规则简单稳定的后台管理 —— 上 RBAC 就够，别过度设计
