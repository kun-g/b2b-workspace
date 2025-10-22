# Spec Delta: 授信管理功能

本文档描述授信管理功能的**增量变更**（Delta），而非完整规格。

## MODIFIED Requirements

### Requirement: 商户授信管理 - 查看授信列表

商户管理员 SHALL 能够查看发出的授信列表，包括每个用户的授信额度、已用额度、可用额度和状态。系统 MUST 实时反映授信的使用情况。

#### Scenario: 商户查看授信使用情况

**Given** 商户管理员登录并访问授信管理页面
**And** 商户已对用户 A 授信 10,000 元
**And** 用户 A 当前有订单占用 3,000 元授信

**When** 商户查看授信列表

**Then** 应显示用户 A 的授信记录，包含以下信息：
- 授信额度：¥10,000
- **已用额度：¥3,000**（新增）
- 可用额度：¥7,000
- **使用进度条**（可选，显示 30% 使用率）
- 状态：启用/禁用（开关）

**And** 已用额度应实时反映当前订单的授信占用情况

---

### Requirement: 商户授信管理 - 启用/禁用授信

商户管理员 SHALL 能够通过开关切换用户的授信状态。系统 MUST 使用 `status` 字段（值为 `'active'` 或 `'disabled'`）而非 `enabled` 布尔字段。被禁用的授信 MUST 阻止用户查看 SKU 和下单。

#### Scenario: 商户禁用用户授信

**Given** 商户管理员登录并访问授信管理页面
**And** 用户 A 的授信状态为"启用"（`status: 'active'`）

**When** 商户点击用户 A 的状态开关，将其切换为"禁用"

**Then** 应发送 PATCH 请求到 `/api/user-merchant-credit/{id}`
**And** 请求体包含 `{ status: 'disabled' }`（而非 `{ enabled: false }`）
**And** 前端显示授信状态为"禁用"
**And** 用户 A 在前端无法看到该商户的 SKU
**And** 用户 A 无法对该商户下单

**Note**：后端授信状态为 `'active' | 'disabled' | 'frozen'`，前端需适配。

---

### Requirement: 商户授信管理 - 调整授信额度

商户管理员 SHALL 能够调整用户的授信额度。系统 MUST 在 `credit_history` 数组中记录每次调整，包括日期、旧额度、新额度、原因和操作人。

#### Scenario: 商户提高用户授信额度

**Given** 商户管理员登录并访问授信管理页面
**And** 用户 A 当前授信额度为 10,000 元

**When** 商户将用户 A 的授信额度修改为 15,000 元并保存

**Then** 应发送 PATCH 请求到 `/api/user-merchant-credit/{id}`
**And** 请求体包含 `{ credit_limit: 15000 }`
**And** 后端在 `credit_history` 数组中记录：
- `date`: 当前时间
- `old_limit`: 10000
- `new_limit`: 15000
- `reason`: "额度调整"
- `operator`: 当前操作人的 User ID

**And** 前端显示更新后的授信额度
**And** （可选）前端可查看授信调整历史

---

## ADDED Requirements

### Requirement: 授信额度不足提示优化

当用户下单时授信额度不足，系统 MUST 返回 400 错误，并提供清晰的错误消息，消息 SHALL 包含可用额度和所需额度。前端 MUST 以 toast 形式显示错误消息，并将设备押金金额显示为红色字体。

#### Scenario: 用户下单时授信额度不足

**Given** 用户登录并访问下单页面
**And** 用户在商户 A 有 5,000 元授信
**And** 用户已用 3,000 元授信（可用 2,000 元）
**And** 用户选择的 SKU 设备价值为 5,000 元

**When** 用户点击"提交订单"

**Then** 后端应返回 400 错误，错误消息为：
`授信额度不足。可用: 2000元，需要: 5000元`

**And** 前端应显示 toast 提示：
`授信额度不足。可用: 2000元，需要: 5000元`

**And** （已实现）下单页面的设备押金金额显示为红色字体

**Note**：当前提示已较清晰，如需进一步优化，可在 Phase 3 调整。

---

### Requirement: 授信释放的测试覆盖

系统 MUST 通过自动化测试验证授信额度在订单完成或取消时正确释放。测试 SHALL 覆盖集成测试和 E2E 测试两个层级。

#### Scenario: 订单完成时释放授信（集成测试）

**Given** 用户有 10,000 元授信
**And** 创建订单冻结了 5,000 元授信（已用额度 = 5,000）
**And** 订单状态为 `RETURNED`

**When** 商户确认完成订单（状态流转为 `COMPLETED`）

**Then** 授信的 `used_credit` 应减少 5,000 元
**And** 授信的 `available_credit` 应增加 5,000 元
**And** 订单的 `credit_hold_amount` 保持不变（记录快照）

---

#### Scenario: 订单取消时释放授信（集成测试）

**Given** 用户有 10,000 元授信
**And** 创建订单冻结了 5,000 元授信（已用额度 = 5,000）
**And** 订单状态为 `PAID`

**When** 用户取消订单（状态流转为 `CANCELED`）

**Then** 授信的 `used_credit` 应减少 5,000 元
**And** 授信的 `available_credit` 应增加 5,000 元
**And** 订单的 `credit_hold_amount` 保持不变（记录快照）

---

#### Scenario: 授信额度不足时阻止下单（E2E 测试）

**Given** 用户在商户 A 有 2,000 元授信（全部可用）
**And** SKU 设备价值为 5,000 元

**When** 用户尝试创建订单

**Then** 应返回 400 错误
**And** 错误消息包含"授信额度不足"
**And** 订单未创建
**And** 授信的 `used_credit` 保持不变

---

#### Scenario: 商户授信管理完整流程（E2E 测试）

**Given** 商户管理员登录

**When** 执行以下操作：
1. 访问授信管理页面
2. 点击"新增授信"，选择用户 B，输入额度 8,000 元
3. 查看授信列表，确认用户 B 的授信已添加
4. 修改用户 B 的授信额度为 10,000 元
5. 禁用用户 B 的授信（切换开关）
6. 启用用户 B 的授信（切换开关）
7. 删除用户 B 的授信

**Then** 每个操作应成功
**And** UI 应实时反映数据变化
**And** API 调用应正确（通过网络面板验证）

---

## REMOVED Requirements

无移除的需求。

---

## RENAMED Requirements

无重命名的需求。

---

## Implementation Notes

### API 字段映射（前端适配）

**后端字段**：
```typescript
interface UserMerchantCredit {
  id: string
  user: number
  merchant: number
  credit_limit: number
  used_credit: number
  available_credit: number  // 计算字段
  status: 'active' | 'disabled' | 'frozen'
  credit_history: Array<{
    date: string
    old_limit: number
    new_limit: number
    reason: string
    operator: number
  }>
}
```

**前端适配**：
- 使用 `credit.status === 'active'` 替代 `credit.enabled`
- Switch 组件切换时，更新 `status` 为 `'active'` 或 `'disabled'`
- 暂时不支持 `'frozen'` 状态（可在未来扩展）

### 测试策略

1. **单元测试**（已完成）：
   - `creditUtils.test.ts` - 冻结/释放逻辑（5/5 通过）

2. **集成测试**（本次新增）：
   - `Orders.test.ts` - 订单完成/取消释放授信

3. **E2E 测试**（本次新增和完善）：
   - `orders-api.e2e.spec.ts` - 完善额度不足测试（当前跳过）
   - `merchant-credit-management.spec.ts` - 新增商户授信管理测试

### 错误处理

**授信额度不足**：
- HTTP 状态码：400 Bad Request
- 错误消息格式：`授信额度不足。可用: {available}元，需要: {required}元`
- 前端处理：`toast.error(error.message)`

**授信状态不可用**：
- HTTP 状态码：400 Bad Request
- 错误消息格式：`授信状态不可用: {status}`

---

## Cross-References

- Related to: `order-management` spec（订单状态机）
- Related to: `merchant-management` spec（商户管理功能）
- Related to: `user-management` spec（用户管理功能）

---

## Versioning

- **Version**: 1.1.0
- **Previous Version**: 1.0.0（初始实现）
- **Changes**:
  - 新增已用额度显示
  - 修复 API 字段不匹配
  - 完善测试覆盖
  - 优化错误提示
