# FC Clearing Audit System

## Table Structure

### fc_clearing_audit

清分流程审计表，记录清分流程每一步的操作、异常和数据不一致情况。

**主要字段：**

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| audit_id | bigint | 主键 | 1 |
| flow_step | varchar(50) | 流程步骤代码 | AGGREGATE |
| flow_step_desc | varchar(50) | 流程步骤描述 | 聚合 |
| operation_type | varchar(50) | 操作类型代码 | SUCCESS |
| operation_type_desc | varchar(50) | 操作类型描述 | 成功 |
| status | varchar(20) | 状态代码 | COMPLETED |
| status_desc | varchar(50) | 状态描述 | 已完成 |
| period | varchar(20) | 业务期间(YYYYMM) | 202511 |
| source_period | varchar(20) | 源期间(用于ALLOCATE) | 202510 |
| target_period | varchar(20) | 目标期间(用于ALLOCATE) | 202511 |
| org_id | varchar(255) | 主体ID | ORG001 |
| org_name | varchar(255) | 主体名称 | 测试组织 |
| task_id | bigint | 清分任务ID | 1001 |
| input_params | text | 输入参数JSON | {"period":"202511","source_count":150} |
| output_summary | text | 输出结果JSON | {"fc_expense_count":12,"gl_total":50000.00} |
| validation_results | text | 验证结果JSON | {"amount_match":true,"gl_calculation_valid":true} |
| warning_messages | text | 警告信息JSON数组 | ["未找到期间 202511 的费用数据"] |
| error_message | text | 错误消息 | 占用金额必须大于0 |
| error_code | varchar(50) | 错误代码 | VALIDATION_FAILED |
| error_code_desc | varchar(50) | 错误代码描述 | 验证失败 |
| trigger_source | varchar(50) | 触发来源代码 | API |
| trigger_source_desc | varchar(50) | 触发来源描述 | API调用 |
| records_processed | int | 处理的记录数 | 150 |
| records_success | int | 成功处理的记录数 | 148 |
| amount_total | numeric(20,2) | 总金额 | 50000.00 |
| execution_time_ms | bigint | 执行耗时(毫秒) | 1250 |

## Enum Values

### FlowStep (流程步骤)
- `AGGREGATE` / `聚合` - 从interface_expense聚合到fc_expense
- `ALLOCATE` / `拆分` - 从fc_expense拆分到fc_expense_allocate
- `OCCUPY` / `占用` - 占用费用
- `CANCEL` / `撤销` - 撤销占用

### OperationType (操作类型)
- `SUCCESS` / `成功` - 操作成功完成
- `WARNING` / `警告` - 操作完成但有警告
- `ERROR` / `错误` - 操作失败
- `DATA_INCONSISTENCY` / `数据不一致` - 数据不一致

### Status (状态)
- `PENDING` / `待处理` - 待处理
- `COMPLETED` / `已完成` - 已完成
- `FAILED` / `失败` - 失败

### TriggerSource (触发来源)
- `API` / `API调用` - API调用
- `QUARTZ` / `定时任务` - 定时任务
- `MANUAL` / `手动触发` - 手动触发

### ErrorCode (错误代码)
- `INSUFFICIENT_FUNDS` / `资金不足` - 可用金额不足
- `VALIDATION_FAILED` / `验证失败` - 验证失败
- `MISSING_RECORDS` / `记录不存在` - 记录不存在
- `CALCULATION_ERROR` / `计算错误` - 计算错误
- `DATA_INCONSISTENCY` / `数据不一致` - 数据不一致

## Example Data

### Example 1: AGGREGATE - SUCCESS

```json
{
  "auditId": 1,
  "flowStep": "AGGREGATE",
  "flowStepDesc": "聚合",
  "operationType": "SUCCESS",
  "operationTypeDesc": "成功",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "period": "202511",
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"period\":\"202511\",\"source_count\":150}",
  "outputSummary": "{\"fc_expense_count\":12,\"gl_total\":50000.00,\"detail_sum\":50000.00}",
  "validationResults": "{\"amount_match\":true,\"gl_calculation_valid\":true,\"all_subjects_present\":true}",
  "recordsProcessed": 150,
  "recordsSuccess": 12,
  "amountTotal": 50000.00,
  "amountExpected": 50000.00,
  "amountActual": 50000.00,
  "executionTimeMs": 1250,
  "createTime": "2025-11-11 10:00:00"
}
```

### Example 2: AGGREGATE - WARNING

```json
{
  "auditId": 2,
  "flowStep": "AGGREGATE",
  "flowStepDesc": "聚合",
  "operationType": "WARNING",
  "operationTypeDesc": "警告",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "period": "202512",
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"period\":\"202512\",\"source_count\":0}",
  "outputSummary": "{\"gl_total\":0,\"fc_expense_count\":0,\"detail_sum\":0}",
  "validationResults": "{\"amount_match\":true,\"gl_calculation_valid\":true,\"all_subjects_present\":true}",
  "warningMessages": "[\"未找到期间 202512 的费用数据\"]",
  "recordsProcessed": 0,
  "recordsSuccess": 0,
  "recordsWarning": 1,
  "amountTotal": 0,
  "executionTimeMs": 50,
  "createTime": "2025-11-11 10:05:00"
}
```

### Example 3: ALLOCATE - SUCCESS

```json
{
  "auditId": 3,
  "flowStep": "ALLOCATE",
  "flowStepDesc": "拆分",
  "operationType": "SUCCESS",
  "operationTypeDesc": "成功",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "sourcePeriod": "202510",
  "targetPeriod": "202511",
  "period": "202511",
  "triggerSource": "QUARTZ",
  "triggerSourceDesc": "定时任务",
  "inputParams": "{\"source_period\":\"202510\",\"target_period\":\"202511\",\"days_count\":30}",
  "outputSummary": "{\"allocate_records\":372,\"total_amount\":50000.00,\"daily_sum\":50000.00,\"deleted_existing\":0}",
  "validationResults": "{\"daily_sum_match\":true,\"precision_loss\":false,\"expected_days\":30,\"actual_records\":372}",
  "recordsProcessed": 372,
  "recordsSuccess": 372,
  "amountTotal": 50000.00,
  "amountExpected": 50000.00,
  "amountActual": 50000.00,
  "executionTimeMs": 850,
  "createTime": "2025-11-11 11:00:00"
}
```

### Example 4: OCCUPY - ERROR

```json
{
  "auditId": 4,
  "flowStep": "OCCUPY",
  "flowStepDesc": "占用",
  "operationType": "ERROR",
  "operationTypeDesc": "错误",
  "status": "FAILED",
  "statusDesc": "失败",
  "orgId": "ORG001",
  "orgName": "测试组织",
  "taskId": 1001,
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"task_id\":1001,\"org_id\":\"ORG001\",\"amount\":10000.00}",
  "dataSnapshot": "{\"before\":{\"total_available\":5000.00}}",
  "validationResults": "{\"sufficient_funds\":false,\"amount_match\":false}",
  "errorMessage": "可用金额不足，总可用: 5000.00, 需要: 10000.00",
  "errorCode": "INSUFFICIENT_FUNDS",
  "errorCodeDesc": "资金不足",
  "amountTotal": 10000.00,
  "amountExpected": 10000.00,
  "amountActual": 0,
  "executionTimeMs": 15,
  "createTime": "2025-11-11 12:00:00"
}
```

### Example 5: OCCUPY - SUCCESS

```json
{
  "auditId": 5,
  "flowStep": "OCCUPY",
  "flowStepDesc": "占用",
  "operationType": "SUCCESS",
  "operationTypeDesc": "成功",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "orgId": "ORG001",
  "orgName": "测试组织",
  "taskId": 1002,
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"task_id\":1002,\"org_id\":\"ORG001\",\"amount\":3000.00}",
  "dataSnapshot": "{\"before\":{\"total_available\":5000.00}}",
  "outputSummary": "{\"usage_records\":3,\"actual_occupied\":3000.00}",
  "validationResults": "{\"sufficient_funds\":true,\"amount_match\":true}",
  "recordsProcessed": 3,
  "recordsSuccess": 3,
  "amountTotal": 3000.00,
  "amountExpected": 3000.00,
  "amountActual": 3000.00,
  "executionTimeMs": 120,
  "createTime": "2025-11-11 12:05:00"
}
```

### Example 6: CANCEL - SUCCESS

```json
{
  "auditId": 6,
  "flowStep": "CANCEL",
  "flowStepDesc": "撤销",
  "operationType": "SUCCESS",
  "operationTypeDesc": "成功",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "taskId": 1002,
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"task_id\":1002}",
  "outputSummary": "{\"cancelled_records\":3,\"restored_amount\":3000.00}",
  "recordsProcessed": 3,
  "recordsSuccess": 3,
  "executionTimeMs": 80,
  "createTime": "2025-11-11 12:10:00"
}
```

### Example 7: CANCEL - WARNING

```json
{
  "auditId": 7,
  "flowStep": "CANCEL",
  "flowStepDesc": "撤销",
  "operationType": "WARNING",
  "operationTypeDesc": "警告",
  "status": "COMPLETED",
  "statusDesc": "已完成",
  "taskId": 9999,
  "triggerSource": "API",
  "triggerSourceDesc": "API调用",
  "inputParams": "{\"task_id\":9999}",
  "outputSummary": "{\"cancelled_records\":0,\"restored_amount\":0}",
  "warningMessages": "[\"没有找到需要撤销的占用记录\"]",
  "recordsProcessed": 0,
  "recordsSuccess": 0,
  "executionTimeMs": 10,
  "createTime": "2025-11-11 12:15:00"
}
```

## Controller API Design

### Base URL
```
/clearing/audit
```

### 1. Query Audit List

**Endpoint:** `GET /clearing/audit/list`

**Description:** 查询审计记录列表，支持分页和多种过滤条件

**Query Parameters:**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| pageNum | int | 否 | 页码，默认1 | 1 |
| pageSize | int | 否 | 每页数量，默认10 | 10 |
| flowStep | string | 否 | 流程步骤过滤 | AGGREGATE |
| operationType | string | 否 | 操作类型过滤 | SUCCESS |
| status | string | 否 | 状态过滤 | COMPLETED |
| period | string | 否 | 期间过滤(YYYYMM) | 202511 |
| orgId | string | 否 | 主体ID过滤 | ORG001 |
| taskId | long | 否 | 任务ID过滤 | 1001 |
| startTime | string | 否 | 开始时间 | 2025-11-01 00:00:00 |
| endTime | string | 否 | 结束时间 | 2025-11-11 23:59:59 |

**Response:**

```json
{
  "code": 200,
  "msg": "查询成功",
  "total": 50,
  "rows": [
    {
      "auditId": "1",
      "flowStep": "AGGREGATE",
      "flowStepDesc": "聚合",
      "operationType": "SUCCESS",
      "operationTypeDesc": "成功",
      "status": "COMPLETED",
      "statusDesc": "已完成",
      "period": "202511",
      "triggerSource": "API",
      "triggerSourceDesc": "API调用",
      "recordsProcessed": 150,
      "recordsSuccess": 12,
      "amountTotal": 50000.00,
      "executionTimeMs": 1250,
      "createTime": "2025-11-11 10:00:00"
    }
  ]
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:8080/clearing/audit/list?pageNum=1&pageSize=10&flowStep=AGGREGATE&operationType=SUCCESS"
```

### 2. Get Audit Detail

**Endpoint:** `GET /clearing/audit/{auditId}`

**Description:** 获取审计记录详细信息

**Path Parameters:**

| 参数名 | 类型 | 必填 | 说明 |
|--------|------|------|------|
| auditId | long | 是 | 审计记录ID |

**Response:**

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "auditId": "1",
    "flowStep": "AGGREGATE",
    "flowStepDesc": "聚合",
    "operationType": "SUCCESS",
    "operationTypeDesc": "成功",
    "status": "COMPLETED",
    "statusDesc": "已完成",
    "period": "202511",
    "triggerSource": "API",
    "triggerSourceDesc": "API调用",
    "inputParams": "{\"period\":\"202511\",\"source_count\":150}",
    "outputSummary": "{\"fc_expense_count\":12,\"gl_total\":50000.00,\"detail_sum\":50000.00}",
    "validationResults": "{\"amount_match\":true,\"gl_calculation_valid\":true}",
    "recordsProcessed": 150,
    "recordsSuccess": 12,
    "amountTotal": 50000.00,
    "amountExpected": 50000.00,
    "amountActual": 50000.00,
    "executionTimeMs": 1250,
    "createTime": "2025-11-11 10:00:00",
    "createBy": "system"
  }
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:8080/clearing/audit/1"
```

### 3. Get Audit Statistics

**Endpoint:** `GET /clearing/audit/statistics`

**Description:** 获取审计统计信息

**Query Parameters:**

| 参数名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| period | string | 否 | 期间过滤 | 202511 |
| flowStep | string | 否 | 流程步骤过滤 | AGGREGATE |
| operationType | string | 否 | 操作类型过滤 | SUCCESS |
| orgId | string | 否 | 主体ID过滤 | ORG001 |
| taskId | long | 否 | 任务ID过滤 | 1001 |
| status | string | 否 | 状态过滤 | COMPLETED |
| startTime | string | 否 | 开始时间 | 2025-11-01 00:00:00 |
| endTime | string | 否 | 结束时间 | 2025-11-11 23:59:59 |

**Response:**

```json
{
  "code": 200,
  "msg": "操作成功",
  "data": {
    "total_count": 50,
    "success_count": 30,
    "warning_count": 15,
    "error_count": 5,
    "inconsistency_count": 0,
    "aggregate_count": 20,
    "allocate_count": 15,
    "occupy_count": 10,
    "cancel_count": 5
  }
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:8080/clearing/audit/statistics?period=202511"
```

## Usage Examples

### Example 1: Query all AGGREGATE operations with WARNING

```bash
curl -X GET "http://localhost:8080/clearing/audit/list?flowStep=AGGREGATE&operationType=WARNING&pageNum=1&pageSize=20"
```

### Example 2: Query all ERROR operations for a specific period

```bash
curl -X GET "http://localhost:8080/clearing/audit/list?operationType=ERROR&period=202511&pageNum=1&pageSize=10"
```

### Example 3: Query all OCCUPY operations for a specific task

```bash
curl -X GET "http://localhost:8080/clearing/audit/list?flowStep=OCCUPY&taskId=1001"
```

### Example 4: Get statistics for a specific period

```bash
curl -X GET "http://localhost:8080/clearing/audit/statistics?period=202511"
```

### Example 5: Query audit records by time range

```bash
curl -X GET "http://localhost:8080/clearing/audit/list?startTime=2025-11-01%2000:00:00&endTime=2025-11-11%2023:59:59"
```

