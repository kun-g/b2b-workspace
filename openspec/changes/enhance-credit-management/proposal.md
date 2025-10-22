# Proposal: 完善授信管理逻辑

## Metadata
- **Change ID**: `enhance-credit-management`
- **Status**: Proposed
- **Created**: 2025-10-22
- **Author**: AI Assistant
- **Priority**: P0 (修复 BUG) + P1 (增强功能)

## Summary

完善 B2B 租赁平台的授信管理功能，包括：
1. **测试完善**：补充授信释放场景的集成测试和 E2E 测试
2. **BUG 修复**：修复商户授信管理页面的编译错误和 API 字段不匹配问题
3. **功能增强**：优化用户下单时的授信额度提示体验

## Context

### 当前状况

**后端实现**：
- ✅ `UserMerchantCredit` Collection 完整（包含 credit_limit, used_credit, available_credit, status, credit_history）
- ✅ `creditUtils.ts` 冻结/释放逻辑完整且有单元测试（5/5 通过）
- ✅ `Orders.ts` 已集成授信冻结/释放 Hooks
- ✅ E2E 测试覆盖订单创建时的授信冻结

**前端实现**：
- ✅ 用户授信查看页面 `MyCredit.tsx`
- ⚠️ 商户授信管理页面 `CreditManagement.tsx` 存在 BUG
- ⚠️ 下单页面缺少授信额度预检查

### 发现的问题

#### 问题 1：测试覆盖不完整
- 缺少订单完成/取消时释放授信的 E2E 测试
- 缺少商户授信管理功能的 E2E 测试
- 缺少授信额度不足场景的完整测试

#### 问题 2：商户授信管理页面 BUG（编译错误）
**位置**：`lease-shelf-flow-01487/src/pages/merchant/CreditManagement.tsx:256`

```typescript
// 错误代码
const available = creditService.getAvailableCredit(String(userId), currentUser.merchantId!);
//                                                                  ^^^^^^^^^^^ 未定义
```

**问题**：
1. `currentUser` 未定义，应该使用 `account`
2. `credit.enabled` 字段不存在（后端是 `credit.status`）
3. 已用额度未显示，用户体验不完整

#### 问题 3：用户下单体验可优化
**当前流程**：
1. 用户填写订单信息
2. 点击提交
3. **后端检查**授信额度
4. 额度不足时返回错误：`授信额度不足。可用: XXX元，需要: XXX元`
5. 前端 toast 显示错误

**改进方向**：
- 根据你的说明，设备押金显示为红色字体（已实现）
- 优化错误提示的可读性

## Motivation

1. **提升代码质量**：通过完善测试覆盖，确保授信逻辑在各种场景下正常工作
2. **修复阻塞性 BUG**：商户授信管理页面无法编译，阻止商户管理授信
3. **改善用户体验**：优化错误提示，帮助用户理解授信额度状态

## Goals

### 优先级 P0（必须完成）
1. ✅ 调查当前授信管理实现状况（已完成）
2. 补充授信释放场景的测试（订单完成/取消）
3. 修复 `CreditManagement.tsx` 编译错误
4. 修复 API 字段不匹配问题（`enabled` vs `status`）

### 优先级 P1（重要）
5. 显示已用额度和授信历史
6. 添加商户授信管理的 E2E 测试
7. 优化授信额度不足的错误提示

### 优先级 P2（可选）
8. 清理 TODO.md 中过时的授信任务
9. 优化授信查询性能（如果有必要）

## Non-Goals

以下内容**不在**本次变更范围内：
- ❌ 重新引入授信邀请码功能（已确认废弃）
- ❌ 修改授信数据模型结构
- ❌ 在下单页面添加额度展示（根据你的反馈，当前的红色字体方案已足够）
- ❌ 修改授信冻结/释放的核心逻辑（已验证正常工作）

## Success Criteria ✅ 已完成

1. **测试覆盖**：
   - [x] 集成测试覆盖订单完成释放授信 ✅
   - [x] 集成测试覆盖订单取消释放授信 ✅
   - [x] E2E 测试覆盖授信额度不足场景 ✅
   - [ ] E2E 测试覆盖商户授信管理流程（可选，建议后续添加）

2. **BUG 修复**：
   - [x] `CreditManagement.tsx` 可以正常编译 ✅
   - [x] 商户可以查看、添加、更新、禁用、删除授信 ✅
   - [x] 授信状态切换正常工作（active/disabled） ✅

3. **用户体验**：
   - [x] 商户授信管理页面显示已用额度 ✅
   - [x] 显示使用进度条和百分比 ✅
   - [x] 超过 80% 使用率时红色警告 ✅
   - [x] 授信额度不足时，错误提示清晰易懂 ✅

## Affected Systems

### 后端
- `b2b-rental-backend/tests/int/` - 添加集成测试
- `b2b-rental-backend/tests/e2e/` - 添加 E2E 测试
- `b2b-rental-backend/TODO.md` - 清理过时任务

### 前端
- `lease-shelf-flow-01487/src/pages/merchant/CreditManagement.tsx` - 修复 BUG，显示已用额度
- `lease-shelf-flow-01487/src/services/creditService.ts` - 可能需要调整（取决于 API 字段）
- `lease-shelf-flow-01487/e2e/tests/` - 添加授信管理 E2E 测试

## Dependencies

- 依赖现有的 `creditUtils.ts`（无需修改）
- 依赖 `UserMerchantCredit` Collection（无需修改）
- 依赖 `Orders` Collection 的 Hooks（无需修改）

## Alternatives Considered

### 方案 1：保持现状，不做修复
**优点**：无需工作量
**缺点**：商户无法使用授信管理功能（编译错误），测试覆盖不足

### 方案 2：重构整个授信模块
**优点**：彻底优化
**缺点**：工作量巨大，风险高，不符合当前需求（核心逻辑已正常工作）

### 方案 3：仅修复 BUG，不完善测试
**优点**：快速上线
**缺点**：无法保证质量，可能引入回归 BUG

### 选择的方案：渐进式完善（测试优先 -> BUG 修复 -> 体验优化）
**优点**：
- 测试优先，确保代码质量
- 修复阻塞性 BUG，恢复商户功能
- 渐进式优化，风险可控

**缺点**：需要分阶段完成（但这正是我们想要的）

## Risks and Mitigations

### 风险 1：测试可能发现现有逻辑的 BUG
**影响**：中等
**缓解措施**：先完善测试，发现问题再修复

### 风险 2：前端 API 字段修改可能影响其他页面
**影响**：低（仅 `CreditManagement.tsx` 使用 `enabled` 字段）
**缓解措施**：全局搜索 `enabled` 字段使用，确保没有遗漏

### 风险 3：E2E 测试可能不稳定
**影响**：低
**缓解措施**：使用现有 E2E 测试框架的最佳实践，添加重试机制

## Open Questions

1. ✅ **授信方法确认**：当前是否还使用邀请码？
   - **答案**：不使用，已移除

2. ✅ **商户授信管理 UI**：保留开关还是改为下拉？
   - **答案**：保留开关（active/disabled）

3. ⚠️ **授信历史显示**：是否需要在管理页面显示授信调整历史？
   - **待确认**：后端已有 `credit_history` 字段，是否需要前端展示？

4. ⚠️ **frozen 状态处理**：后端有 `frozen` 状态，前端是否需要支持？
   - **待确认**：当前前端只支持 active/disabled

## Timeline Estimate

- **Phase 1 - 测试完善**（优先级 C）：2-3 小时
  - 集成测试：1 小时
  - E2E 测试：1-2 小时

- **Phase 2 - BUG 修复**（优先级 A）：1-2 小时
  - 修复编译错误：30 分钟
  - 修复 API 字段：30 分钟
  - 显示已用额度：30 分钟
  - 测试验证：30 分钟

- **Phase 3 - 体验优化**（优先级 B）：1 小时
  - 优化错误提示：30 分钟
  - 测试验证：30 分钟

**总计**：4-6 小时

## Approval

- [ ] Product Owner / 用户确认
- [ ] Tech Lead 确认
- [ ] 架构师确认（如适用）

## References

- PRD: `b2b-rental-backend/docs/prd_update.md` - 第 4 节（授信与可见性）
- 前后端对接文档: `CLAUDE.md` - 授信管理部分
- 单元测试: `b2b-rental-backend/src/utils/creditUtils.test.ts`
- E2E 测试: `b2b-rental-backend/tests/e2e/orders-api.e2e.spec.ts`
