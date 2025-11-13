# fc_audit 审计日志设计

## 表结构设计

```sql
-- =============================================
-- 通用审计日志表（覆盖所有fc_*模块）
-- =============================================

DROP TABLE IF EXISTS fc_audit CASCADE;

CREATE TABLE fc_audit (
    id BIGSERIAL PRIMARY KEY,
    
    -- 业务标识
    module VARCHAR(50) NOT NULL,          -- 模块：EXPENSE/ORDER/TASK/CLEARING/ORG
    action VARCHAR(50) NOT NULL,          -- 操作动作：AGGREGATE/ALLOCATE/OCCUPY/CANCEL/EXECUTE/CREATE/UPDATE/DELETE
    subject_id VARCHAR(100),              -- 业务主键ID（通用字段，可存task_id/order_id/expense_id/period等）
    subject_name VARCHAR(200),            -- 业务对象名称（通用字段，可存task_name/order_no/org_name等）
    
    -- 关联标识（冗余字段，方便查询）
    task_id BIGINT,                       -- 清分任务ID
    order_id BIGINT,                      -- 订单ID
    org_id VARCHAR(50),                   -- 组织ID
    org_name VARCHAR(200),                -- 组织名称
    period VARCHAR(6),                    -- 期间（YYYYMM）
    
    -- 操作信息
    operation_type VARCHAR(50),           -- 操作类型：START/PROCESS/COMPLETE/VALIDATE
    status VARCHAR(20) NOT NULL,          -- 执行状态：SUCCESS/ERROR
    severity VARCHAR(20),                 -- 严重级别：INFO/SUCCESS/WARNING/ERROR/CRITICAL（UI根据此决定样式）
    
    -- 统计
    records_processed INT,                -- 处理记录总数
    records_success INT,                  -- 成功记录数
    records_failed INT,                   -- 失败记录数
    
    -- 金额
    amount_total NUMERIC(20, 2),          -- 总金额（期望处理的总金额）
    amount_actual NUMERIC(20, 2),         -- 实际金额（实际处理成功的金额）
    amount_difference NUMERIC(20, 2),     -- 差异金额（= amount_actual - amount_total）
    
    -- 错误信息
    error_code VARCHAR(50),               -- 错误代码（如：EXP_INSUFFICIENT_AMOUNT）
    error_message TEXT,                   -- 技术错误信息（供开发排查）
    user_message TEXT,                    -- 用户友好提示（前端直接显示）
    stack_trace TEXT,                     -- 异常堆栈（仅开发调试）
    
    -- 明细（JSON）
    detail_items TEXT,                    -- 明细列表JSON数组（前端直接渲染表格，仅失败记录）
    
    -- 执行信息
    execution_time_ms BIGINT,             -- 执行耗时（毫秒）
    start_time TIMESTAMP,                 -- 开始执行时间
    end_time TIMESTAMP,                   -- 结束执行时间
    
    -- 审计
    create_by VARCHAR(50) DEFAULT 'system',
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 索引
CREATE INDEX idx_audit_module_action ON fc_audit(module, action);
CREATE INDEX idx_audit_subject_id ON fc_audit(subject_id);
CREATE INDEX idx_audit_task_id ON fc_audit(task_id);
CREATE INDEX idx_audit_order_id ON fc_audit(order_id);
CREATE INDEX idx_audit_org_period ON fc_audit(org_id, period);
CREATE INDEX idx_audit_severity ON fc_audit(severity);
CREATE INDEX idx_audit_create_time ON fc_audit(create_time DESC);
```

---

## 模块和业务类型枚举

### Module（模块）

| 值 | 说明 |
|----|------|
| **EXPENSE** | 费用模块（归集/分摊/占用/撤销） |
| **ORDER** | 订单模块（创建/更新/删除/校验） |
| **TASK** | 清分任务模块（创建/执行/取消） |
| **CLEARING** | 清分结果模块（生成清分结果） |
| **ORG** | 组织模块（组织/规则/路由配置） |

### Action（操作动作）

| 值 | 说明 | 适用模块 |
|----|------|---------|
| **AGGREGATE** | 归集 | EXPENSE |
| **ALLOCATE** | 分摊 | EXPENSE |
| **OCCUPY** | 占用 | EXPENSE |
| **CANCEL** | 撤销/取消 | EXPENSE, TASK |
| **EXECUTE** | 执行 | TASK, CLEARING |
| **CREATE** | 创建 | ORDER, TASK, ORG |
| **UPDATE** | 更新 | ORDER, ORG |
| **DELETE** | 删除 | ORDER, ORG |
| **VALIDATE** | 校验 | ALL |

### Severity（严重级别）

| 值 | 说明 | UI建议颜色 |
|----|------|-----------|
| **INFO** | 信息 | 蓝色 |
| **SUCCESS** | 成功 | 绿色 |
| **WARNING** | 警告 | 橙色 |
| **ERROR** | 错误 | 红色 |
| **CRITICAL** | 严重错误 | 深红色 |

---

## API响应示例

### 1. 查询列表
`GET /clearing/audit/list`

**参数**: module, action, taskId, orgId, period, severity, status

**响应**:
```json
{
  "code": 200,
  "total": 156,
  "rows": [
    {
      "id": 12346,
      "module": "EXPENSE",
      "action": "OCCUPY",
      "subjectId": "1001",
      "subjectName": "202404期间费用占用",
      "taskId": 1001,
      "orgId": "0001A21000000000LZD6",
      "orgName": "鲜道源（自营）",
      "period": "202404",
      "status": "ERROR",
      "severity": "ERROR",
      "userMessage": "可用金额不足，缺口5000.00元",
      "recordsProcessed": 31,
      "recordsFailed": 3,
      "executionTimeMs": 125,
      "createTime": "2024-10-10 14:30:15"
    }
  ]
}
```

### 2. 费用占用失败详情
`GET /clearing/audit/{id}`

```json
{
  "code": 200,
  "data": {
    "id": 12346,
    
    "module": "EXPENSE",
    "action": "OCCUPY",
    "subjectId": "1001",
    "subjectName": "202404期间费用占用",
    
    "taskId": 1001,
    "orderId": 5001,
    "orgId": "0001A21000000000LZD6",
    "orgName": "鲜道源（自营）",
    "period": "202404",
    
    "operationType": "PROCESS",
    "status": "ERROR",
    "severity": "ERROR",
    
    "recordsProcessed": 31,
    "recordsSuccess": 28,
    "recordsFailed": 3,
    
    "amountTotal": 10000.00,
    "amountActual": 8387.10,
    "amountDifference": -1612.90,
    
    "errorCode": "EXP_INSUFFICIENT_AMOUNT",
    "errorMessage": "Available amount insufficient: need 10000.00, available 8387.10, gap 1612.90",
    "userMessage": "可用金额不足。需要：10000.00元，可用：8387.10元，缺口：1612.90元",
    "stackTrace": "com.yzt.clearing.exception.InsufficientAmountException...",
    
    "detailItems": [
      {
        "date": "2024-04-03",
        "expenseType": "GL",
        "allocateId": 1003,
        "amount": 1451.61,
        "availableAmount": 0.00,
        "status": "error",
        "message": "可用金额不足"
      },
      {
        "date": "2024-04-04",
        "expenseType": "GL",
        "allocateId": 1004,
        "amount": 1451.61,
        "availableAmount": 500.00,
        "status": "error",
        "message": "可用金额不足"
      }
    ],
    
    "executionTimeMs": 125,
    "startTime": "2024-10-10 14:30:15",
    "endTime": "2024-10-10 14:30:15",
    
    "createBy": "admin",
    "createTime": "2024-10-10 14:30:15"
  }
}
```

### 3. 清分任务执行失败详情
`GET /clearing/audit/{id}`

```json
{
  "code": 200,
  "data": {
    "id": 12350,
    
    "module": "CLEARING",
    "action": "EXECUTE",
    "subjectId": "1005",
    "subjectName": "PT202240114001",
    
    "taskId": 1005,
    "orgId": "0001A21000000000LZD6",
    "orgName": "鲜道源（自营）",
    "period": "202401",
    
    "operationType": "PROCESS",
    "status": "ERROR",
    "severity": "CRITICAL",
    
    "recordsProcessed": 15,
    "recordsSuccess": 10,
    "recordsFailed": 5,
    
    "amountTotal": 150000.00,
    "amountActual": 0.00,
    "amountDifference": -150000.00,
    
    "errorCode": "TASK_EXECUTE_FAILED",
    "errorMessage": "Multiple calculation rules failed: CALC_ERROR_002, DB_TIMEOUT_001, CALC_RRHCM_002",
    "userMessage": "清分任务执行失败，5条规则计算失败，请检查规则配置和数据完整性",
    "stackTrace": "com.yzt.clearing.exception.TaskExecutionException...",
    
    "detailItems": [
      {
        "ruleId": "RULE_001",
        "ruleName": "费用分摊规则",
        "ruleType": "CALC_ERROR_002",
        "status": "error",
        "time": "2024-01-14 10:30:12",
        "message": "计算超时：规则执行超过30秒限制",
        "affectedOrders": 25
      },
      {
        "ruleId": "RULE_003",
        "ruleName": "订单金额校验",
        "ruleType": "DB_TIMEOUT_001",
        "status": "error",
        "time": "2024-01-14 10:30:15",
        "message": "数据库连接超时：查询订单金额时超时",
        "affectedOrders": 10
      },
      {
        "ruleId": "RULE_007",
        "ruleName": "路由计算规则",
        "ruleType": "CALC_RRHCM_002",
        "status": "error",
        "time": "2024-01-14 10:30:20",
        "message": "路由配置缺失：组织0001A21000000000LZD6缺少路由规则",
        "affectedOrders": 8
      }
    ],
    
    "executionTimeMs": 35200,
    "startTime": "2024-01-14 10:30:00",
    "endTime": "2024-01-14 10:30:35",
    
    "createBy": "system",
    "createTime": "2024-01-14 10:30:35"
  }
}
```

### 4. 费用归集成功
`GET /clearing/audit/{id}`

```json
{
  "code": 200,
  "data": {
    "id": 12347,
    
    "module": "EXPENSE",
    "action": "AGGREGATE",
    "subjectId": "202511",
    "subjectName": "202511期间费用归集",
    
    "taskId": null,
    "orderId": null,
    "orgId": "0001A21000000000LZD6",
    "orgName": "鲜道源（自营）",
    "period": "202511",
    
    "operationType": "COMPLETE",
    "status": "SUCCESS",
    "severity": "SUCCESS",
    
    "recordsProcessed": 6,
    "recordsSuccess": 6,
    "recordsFailed": 0,
    
    "amountTotal": 45000.00,
    "amountActual": 45000.00,
    "amountDifference": 0.00,
    
    "errorCode": null,
    "errorMessage": null,
    "userMessage": "费用归集成功，共处理6条记录，生成GL总账45000.00元",
    "stackTrace": null,
    
    "detailItems": null,
    
    "executionTimeMs": 280,
    "startTime": "2024-11-13 10:15:30",
    "endTime": "2024-11-13 10:15:31",
    
    "createBy": "system",
    "createTime": "2024-11-13 10:15:31"
  }
}
```

---

## 错误代码（基于真实业务场景）

### 费用模块（EXPENSE）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `EXP_INSUFFICIENT_AMOUNT` | ERROR | 费用可用金额不足 | 可用金额不足，需要10000.00元，可用8387.10元，缺口1612.90元 |
| `EXP_PERIOD_NOT_FOUND` | ERROR | 期间不存在或未归集 | 期间202411不存在，请先执行费用归集 |

### 订单模块（ORDER）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `ORD_NOT_FOUND` | ERROR | 订单不存在 | 订单1001不存在或已删除 |
| `ORD_ALREADY_OCCUPIED` | ERROR | 订单已被占用 | 订单P20241115001已被任务100占用，无法重复清分 |

### 清分任务模块（TASK）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `TASK_ROUTE_MISSING` | ERROR | 任务缺少路由配置 | 组织0001A21000000000LZD6缺少清分路由配置 |
| `TASK_EXECUTE_FAILED` | CRITICAL | 任务执行失败 | 清分任务执行失败：5个节点计算失败 |

### 组织路由/规则模块（ORG）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `ORG_NOT_FOUND` | ERROR | 组织不存在 | 组织0001A21000000000LZD6不存在 |
| `ORG_RULE_NOT_FOUND` | ERROR | 组织规则不存在 | 组织0001A21000000000LZD6未配置清分规则 |

### 清分结果模块（CLEARING）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `CLR_AMOUNT_MISMATCH` | CRITICAL | 清分金额不一致 | 清分结果金额校验失败：订单金额50000.00元，清分金额55000.00元，差异5000.00元 |
| `CLR_RESULT_DUPLICATE` | ERROR | 清分结果重复 | 订单1001已存在清分结果 |

### 系统级错误（SYS）

| 错误代码 | Severity | 说明 | 用户提示示例 |
|---------|----------|------|------------|
| `SYS_DATABASE_ERROR` | CRITICAL | 数据库异常 | 系统错误：数据库连接失败，请联系管理员 |
| `SYS_CONCURRENT_UPDATE` | ERROR | 并发更新冲突 | 数据已被其他用户修改，请刷新后重试 |


