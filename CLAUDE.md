<!-- OPENSPEC:START -->
# OpenSpec Instructions

These instructions are for AI assistants working in this project.

Always open `@/openspec/AGENTS.md` when the request:
- Mentions planning or proposals (words like proposal, spec, change, plan)
- Introduces new capabilities, breaking changes, architecture shifts, or big performance/security work
- Sounds ambiguous and you need the authoritative spec before coding

Use `@/openspec/AGENTS.md` to learn:
- How to create and apply change proposals
- Spec format and conventions
- Project structure and guidelines

Keep this managed block so 'openspec update' can refresh the instructions.

<!-- OPENSPEC:END -->

# B2B项目 - API对接说明

## 项目结构

- **后端**: `b2b-rental-backend/`
- **前端**: `lease-shelf-flow-01487/`

## 工作说明

当前文件夹用于处理前后端API对接工作，确保两边的接口信息保持一致。

## API对接流程

### 1. 对接前准备

- [ ] 确认后端API文档位置
- [ ] 确认前端API调用规范
- [ ] 准备测试环境和工具
- [ ] 同步后端和前端的环境配置

### 2. 后端接口梳理

- [ ] 列出所有后端API端点（Controller/Router）
- [ ] 记录每个接口的：
  - 请求方法（GET/POST/PUT/DELETE等）
  - 路径和参数
  - 请求体结构（Body/Query/Params）
  - 响应数据结构
  - 认证和权限要求
  - 错误码定义

### 3. 前端接口梳理

- [ ] 列出前端所有API调用位置（Service/API层）
- [ ] 记录每个调用的：
  - 调用的端点
  - 传参方式和数据结构
  - 响应处理逻辑
  - 错误处理机制

### 4. 接口对比验证

- [ ] 对比前后端接口清单，确保一一对应
- [ ] 验证请求参数命名和类型是否一致
- [ ] 验证响应数据结构是否匹配
- [ ] 检查是否有前端调用但后端未实现的接口
- [ ] 检查是否有后端提供但前端未使用的接口

### 5. 数据类型对齐

- [ ] TypeScript类型定义 vs 后端Model/DTO
- [ ] 日期时间格式（ISO 8601, Unix timestamp等）
- [ ] 枚举值定义
- [ ] 空值处理（null/undefined/empty）
- [ ] 数字精度（整数/浮点数）

### 6. 测试验证

- [ ] 使用Postman/curl测试后端接口
- [ ] 在开发环境联调前后端
- [ ] 测试各种边界情况：
  - 正常流程
  - 参数缺失/错误
  - 认证失败
  - 权限不足
  - 网络异常

### 7. 文档同步

- [ ] 更新API文档
- [ ] 记录已知问题和注意事项
- [ ] 更新前端TypeScript类型定义
- [ ] 记录版本变更历史

## 常见问题检查清单

- [ ] CORS配置是否正确
- [ ] 接口基础URL是否配置正确
- [ ] Token/Cookie传递是否正常
- [ ] 请求超时时间设置
- [ ] 文件上传大小限制
- [ ] 分页参数命名统一（page/pageSize vs pageNum/pageSize）

## 对接记录

### 已对接接口

#### 用户登录功能

**1. 后端接口（Payload CMS）**

后端使用 Payload CMS 3.x，认证系统基于 `Accounts` Collection。

- **技术栈**: Payload CMS + Next.js 15 + PostgreSQL
- **认证Collection**: `Accounts`（登录凭证）+ `Users`（业务身份）
- **设计特点**: 一个Account可以有多个User（不同业务身份）

**API端点：**

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 登录 | POST | `/api/accounts/login` | Payload CMS自动提供 |
| 注销 | POST | `/api/accounts/logout` | Payload CMS自动提供 |
| 获取当前用户 | GET | `/api/accounts/me` | Payload CMS自动提供 |
| 刷新Token | POST | `/api/accounts/refresh-token` | Payload CMS自动提供 |

**登录接口详情：**
```typescript
// 请求
POST /api/accounts/login
Content-Type: application/json

{
  "username": "string",  // 或 "email" 或 "phone"
  "password": "string"
}

// 响应
{
  "user": {
    "id": "string",
    "username": "string",
    "email": "string",
    "phone": "string",
    "status": "active" | "disabled",
    // ... 其他Accounts字段
  },
  "token": "string",  // JWT token
  "exp": number       // token过期时间戳
}
```

**认证配置（src/collections/Accounts.ts:71-81）:**
- Token有效期：7天
- 最大登录尝试：5次
- 锁定时间：2小时
- 支持的登录方式：用户名/邮箱/手机号 + 密码

**2. 前端实现**

- **位置**: `lease-shelf-flow-01487/src/services/api/accountApi.ts`
- **认证服务**: `lease-shelf-flow-01487/src/services/authService.ts`
- **认证上下文**: `lease-shelf-flow-01487/src/contexts/AuthContext.tsx`

**前端调用路径：**

| 功能 | 实际调用路径 | 期望后端路径 |
|------|-------------|-------------|
| 登录 | `/accounts/login` | `/api/accounts/login` |
| 注销 | `/accounts/logout` | `/api/accounts/logout` |
| 获取当前用户 | `/accounts/me` | `/api/accounts/me` |
| 刷新Token | `/accounts/refresh-token` | `/api/accounts/refresh-token` |

**Token处理：**
- 存储方式：`localStorage`
- Token键：`jwt_token`
- 认证头格式：`Authorization: JWT ${token}`

**环境配置（.env）：**
```
VITE_API_ENDPOINT=http://localhost:3000
```

**3. 接口对比分析**

✅ **已对齐的部分：**
- 请求体结构：`{ username, password }` 一致
- 认证头格式：都使用 `JWT ${token}`
- Token存储：都使用localStorage

⚠️ **存在的问题：**

1. **API路径不匹配** ⭐ 严重
   - 前端调用：`/accounts/login`
   - 后端实际：`/api/accounts/login`
   - **影响**：无法正常调用API
   - **解决方案**：需要配置路径前缀或调整前端API调用

2. **响应数据结构差异**
   - 后端返回完整的Accounts对象
   - 前端期望的User类型包含额外字段（如`role`, `merchant`等）
   - **问题**：前端的User类型来自Users Collection，而登录返回的是Accounts
   - **影响**：数据映射可能不完整

3. **角色权限系统不匹配**
   - 后端：Accounts（登录） + Users（业务身份和角色）
   - 前端：期望User对象直接包含role字段
   - **影响**：前端可能无法正确获取用户角色信息

#### 下单功能

**1. 后端接口（Payload CMS）**

**API端点：**

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 创建订单 | POST | `/api/orders` | 创建新订单 |
| 查询订单 | GET | `/api/orders` | 查询订单列表 |
| 订单详情 | GET | `/api/orders/:id` | 获取订单详情 |
| 更新订单 | PATCH | `/api/orders/:id` | 更新订单状态等 |

**创建订单接口详情：**
```typescript
// 请求
POST /api/orders
Content-Type: application/json
Authorization: JWT <token>

{
  "customer": "7",              // User ID（不是 Account ID）
  "merchant_sku": 4,
  "rent_start_date": "2025-10-23",
  "rent_end_date": "2025-10-29",
  "shipping_address": {
    "contact_name": "张三",
    "contact_phone": "13800138000",
    "province": "广东省",
    "city": "深圳市",
    "district": "南山区",       // 可以为空，系统会自动解析
    "address": "科技园南路15号",
    "region_code": "440305"
  }
}

// 响应
{
  "doc": {
    "id": "123",
    "order_no": "ORD-1760937480-ABC123",
    "status": "NEW",
    "customer": 7,
    "merchant": 1,
    "merchant_sku": 4,
    "rent_start_date": "2025-10-23",
    "rent_end_date": "2025-10-29",
    "daily_fee_snapshot": 100,
    "shipping_fee_snapshot": 15,
    "credit_hold_amount": 5000,
    "shipping_address": {
      "contact_name": "张三",
      "contact_phone": "13800138000",
      "province": "广东省",
      "city": "深圳市",
      "district": "南山区",     // 已自动解析补全
      "address": "科技园南路15号",
      "region_code": "440305"
    },
    "createdAt": "2025-10-20T05:30:00.000Z"
  }
}
```

**后端特性：**
- ✅ **自动填充 customer 字段**：从当前登录用户的 User ID 自动填充，防止冒充
- ✅ **地址智能解析**：使用 `addressParser` 自动解析和补全地址字段
- ✅ **地址验证**：严格验证省市区必填，详细地址不能为空
- ✅ **自动计算运费**：根据运费模板和收货地址自动计算
- ✅ **授信额度冻结**：创建订单时自动冻结设备价值对应的授信额度
- ✅ **价格快照**：锁定下单时的日租金、运费等价格

**地址解析功能（addressParser）：**

系统会自动处理以下场景：

1. **补全缺失的 district**
   ```javascript
   // 前端发送
   {
     "province": "北京市",
     "city": "",
     "district": "",
     "address": "朝阳区望京SOHO"
   }

   // 后端自动解析补全
   {
     "province": "北京市",
     "city": "",
     "district": "朝阳区",  // ← 自动补全
     "address": "朝阳区望京SOHO"
   }
   ```

2. **修正重复的字段**
   ```javascript
   // 前端错误（city 和 district 重复）
   {
     "province": "广东省",
     "city": "深圳市",
     "district": "深圳市",  // ← 错误
     "address": "南山区科技园"
   }

   // 后端自动修正
   {
     "province": "广东省",
     "city": "深圳市",
     "district": "南山区",  // ← 自动修正
     "address": "南山区科技园"
   }
   ```

3. **严格验证**
   - ❌ 如果无法解析出完整的省市区，返回清晰的错误提示
   - ❌ 详细地址不能为空
   - ❌ 联系人和电话必填

**2. 前端实现**

- **位置**: `lease-shelf-flow-01487/src/services/api/orderApi.ts`
- **订单服务**: `lease-shelf-flow-01487/src/services/orderService.ts`
- **下单页面**: `lease-shelf-flow-01487/src/pages/customer/PlaceOrder.tsx`
- **地址解析工具**: `lease-shelf-flow-01487/src/utils/addressParser.ts`

**前端特性：**
- ✅ **智能地址解析**：用户可以粘贴完整地址，一键解析为省市区和详细地址
- ✅ **手动选择省市区**：支持传统的省市区级联选择器
- ✅ **实时运费计算**：根据选择的地区实时显示运费
- ✅ **友好错误提示**：地址格式错误时给出清晰的提示

**前端调用示例：**
```typescript
import { orderApi } from '@/services/api/orderApi'

// 创建订单
const result = await orderApi.createOrder({
  merchant_sku: "4",
  rent_start_date: "2025-10-23",
  rent_end_date: "2025-10-29",
  contact_name: "张三",
  contact_phone: "13800138000",
  region_code_path: ["广东省", "深圳市", "南山区"],
  address_detail: "科技园南路15号",
  remarks: "尽快发货"
})
```

**智能地址解析示例：**
```typescript
import { parseAddress, toRegionCodePath } from '@/utils/addressParser'

// 解析完整地址
const parsed = parseAddress('广东省深圳市南山区科技园南路15号')
// => {
//   province: '广东省',
//   city: '深圳市',
//   district: '南山区',
//   street: '科技园南路15号'
// }

// 转换为 regionCodePath
const path = toRegionCodePath(parsed)
// => ['广东省', '深圳市', '南山区']
```

**UI 功能**：

在下单页面的"收货地址"部分，用户可以：

1. **智能解析**（推荐）
   - 粘贴完整地址：`广东省深圳市南山区科技园南路15号`
   - 点击"解析"按钮
   - 系统自动填充省市区选择器和详细地址

2. **手动选择**
   - 使用省市区级联选择器
   - 手动输入详细地址

**3. 接口对比分析**

✅ **已解决的问题：**

1. ✅ **customer 字段自动填充**
   - 后端从当前登录用户自动获取正确的 User ID
   - 防止用户冒充其他用户下单

2. ✅ **Account ID vs User ID 混淆**
   - 前端正确提取 User ID（不是 Account ID）
   - 多角色用户按优先级自动选择（platform_admin > merchant_admin > customer）

3. ✅ **地址解析和验证**
   - 自动补全缺失的地址字段
   - 修正前端地址数据错误
   - 提供清晰的错误提示

4. ✅ **同角色重复创建防护**
   - 后端防止同一个 Account 创建多个同角色的 User
   - 避免数据混乱

5. ✅ **前端智能地址解析**
   - 用户可以粘贴完整地址一键解析
   - 自动识别省市区并填充
   - 减少用户输入错误，提升体验

#### 地址解析功能

**1. 后端接口（Next.js API Route）**

**API端点：**

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 解析地址 | POST | `/api/parse-address` | 将完整地址解析为省市区街道组件 |

**地址解析接口详情：**
```typescript
// 请求
POST /api/parse-address
Content-Type: application/json

{
  "address": "广东省深圳市南山区科技园南路15号"
}

// 响应
{
  "success": true,
  "data": {
    "province": "广东省",
    "city": "深圳市",
    "district": "南山区",
    "street": "科技园南路15号"
  },
  "original": "广东省深圳市南山区科技园南路15号"
}
```

**后端特性：**
- ✅ **精确解析**：使用 `china-division` 库，基于完整的中国省市区数据库
- ✅ **支持简写**：自动识别省市区简写（如"安徽宿州灵璧"）
- ✅ **直辖市处理**：正确处理北京/上海/天津/重庆的特殊结构
- ✅ **容错能力**：支持多种地址格式

**实现位置：**
- **API Route**: `b2b-rental-backend/src/app/api/parse-address/route.ts`
- **解析工具**: `b2b-rental-backend/src/utils/addressParser.ts`

**2. 前端实现**

**前端API调用：**
- **位置**: `lease-shelf-flow-01487/src/services/api/addressApi.ts`
- **使用页面**: `lease-shelf-flow-01487/src/pages/customer/PlaceOrder.tsx`

**前端调用示例：**
```typescript
import { addressApi } from '@/services/api/addressApi'

// 解析地址
try {
  const parsed = await addressApi.parseAddress('广东省深圳市南山区科技园南路15号')
  console.log(parsed)
  // => { province: '广东省', city: '深圳市', district: '南山区', street: '科技园南路15号' }
} catch (error) {
  console.error('地址解析失败', error)
}
```

**前端特性：**
- ✅ **异步调用**：使用后端API进行解析，保证准确性
- ✅ **加载状态**：显示"解析中..."提示
- ✅ **错误处理**：友好的错误提示
- ✅ **自动填充**：解析成功后自动填充省市区选择器和详细地址

**3. 接口对比分析**

✅ **已解决的问题：**

1. ✅ **统一解析能力**
   - 前端不再使用本地的简化版解析工具
   - 统一使用后端的精确解析引擎
   - 提升解析准确率

2. ✅ **用户体验提升**
   - 用户粘贴地址后立即获得准确反馈
   - 解析失败时给出清晰错误提示
   - 避免提交订单时才发现地址错误

3. ✅ **代码维护性**
   - 单一数据源（china-division）
   - 前端无需维护地址解析逻辑
   - 解析规则更新只需修改后端

**相关文档：**
- [后端地址解析功能说明](b2b-rental-backend/docs/ADDRESS_PARSING_GUIDE.md)
- [前端地址解析功能说明](lease-shelf-flow-01487/docs/ADDRESS_PARSER_FEATURE.md)
- [Orders Collection](b2b-rental-backend/src/collections/Orders.ts)
- [addressParser 工具 - 后端](b2b-rental-backend/src/utils/addressParser.ts)
- [addressApi - 前端](lease-shelf-flow-01487/src/services/api/addressApi.ts)

#### 授信管理功能

**1. 后端接口（Payload CMS）**

**API端点：**

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取商户授信列表 | GET | `/api/user-merchant-credit?where[merchant][equals]={merchantId}` | 商户查看授权给用户的授信列表 |
| 获取用户授信列表 | GET | `/api/user-merchant-credit?where[user][equals]={userId}&where[status][equals]=active` | 用户查看自己的授信列表 |
| 创建授信 | POST | `/api/user-merchant-credit` | 商户为用户创建授信 |
| 更新授信 | PATCH | `/api/user-merchant-credit/{id}` | 更新授信额度或状态 |
| 删除授信 | DELETE | `/api/user-merchant-credit/{id}` | 删除授信 |

**数据模型：**
```typescript
{
  id: string
  user: number                    // User ID
  merchant: number                // Merchant ID
  credit_limit: number            // 授信额度
  used_credit: number             // 已用额度（自动计算）
  available_credit: number        // 可用额度（自动计算）
  status: 'active' | 'disabled' | 'frozen'  // 授信状态
  credit_history: Array<{        // 额度调整历史（自动记录）
    date: string
    old_limit: number
    new_limit: number
    reason: string
    operator: number
  }>
  createdAt: string
  updatedAt: string
}
```

**后端特性：**
- ✅ **自动计算额度**：`available_credit = credit_limit - used_credit`
- ✅ **授信冻结/释放**：订单创建时冻结，完成/取消时释放
- ✅ **历史记录**：每次额度调整自动记录到 `credit_history` 数组
- ✅ **用户角色验证**：创建授信时验证 user 必须是 `customer` 角色，防止为其他角色创建授信
- ✅ **权限控制**：
  - 商户只能管理自己的授信
  - 用户只能查看自己的授信
  - 禁用/冻结的授信无法用于下单

**2. 前端实现**

**商户端页面**：
- **位置**: `lease-shelf-flow-01487/src/pages/merchant/CreditManagement.tsx`
- **路由**: `/merchant/credit`
- **功能**：
  - ✅ 查看授信列表（用户、额度、已用、可用、状态）
  - ✅ 添加新授信（**输入 customer 的 User ID**、输入额度）
  - ✅ 更新授信额度（直接编辑输入框，失焦保存）
  - ✅ 切换授信状态（开关：active/disabled）
  - ✅ 删除授信
  - ✅ 显示已用额度进度条（>80% 红色警告）
  - ✅ 查看授信历史记录（对话框展示调整历史）

**用户端页面**：
- **位置**: `lease-shelf-flow-01487/src/pages/customer/MyCredit.tsx`
- **路由**: `/customer/my-credit`
- **功能**：
  - ✅ 查看自己的授信列表
  - ✅ 显示每个商户的授信额度和可用额度

**API服务层**：
- **位置**: `lease-shelf-flow-01487/src/services/creditService.ts`
- **类型定义**: `lease-shelf-flow-01487/src/services/api/creditApi.ts`

**3. 接口对比分析**

✅ **已完全对齐**：
- 数据模型：前后端使用相同的字段结构
- 状态字段：统一使用 `status` ('active' | 'disabled' | 'frozen')
- API 调用：前端正确调用后端 REST API
- 权限控制：商户和用户权限正确分离

**4. 测试覆盖**

✅ **后端测试**：
- 单元测试：`creditUtils.test.ts` (5/5 通过) - 测试冻结/释放逻辑
- 集成测试：`Orders.test.ts` (4/4 通过) - 测试订单完成/取消时释放授信
- E2E 测试：`orders-api.e2e.spec.ts` - 测试授信额度不足阻止下单

✅ **前端测试**：
- E2E 测试：`credit-scenarios.spec.ts` - 测试客户授信场景（6种角色）
- E2E 测试：`merchant-credit-management.spec.ts` - 测试商户授信管理流程

**相关文档：**
- [UserMerchantCredit Collection](b2b-rental-backend/src/collections/UserMerchantCredit.ts)
- [creditUtils 工具](b2b-rental-backend/src/utils/creditUtils.ts)
- [OpenSpec 变更记录](openspec/changes/enhance-credit-management/)

#### 归还地址管理功能

**1. 后端实现（Payload CMS）**

**API端点：**

| 功能 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取归还信息列表 | GET | `/api/return-info` | 商户/平台管理员查看归还信息 |
| 创建归还信息 | POST | `/api/return-info` | 商户管理员创建归还地址 |
| 更新归还信息 | PATCH | `/api/return-info/{id}` | 更新归还地址 |
| 删除归还信息 | DELETE | `/api/return-info/{id}` | 删除归还地址 |

**数据模型：**
```typescript
{
  id: string
  merchant: number                  // Merchant ID
  return_contact_name: string       // 归还联系人
  return_contact_phone: string      // 联系电话
  return_address: {                // 归还地址
    province: string
    city: string
    district: string
    address: string
    postal_code: string
  }
  status: 'active' | 'disabled'    // 状态
  is_default: boolean               // 是否默认地址
  notes: string                     // 备注
}
```

**后端特性：**
- ✅ **自动设置默认地址**：每个商户只能有一个默认归还地址，设置新默认时自动取消其他默认
- ✅ **订单自动填充**：创建订单时自动从商户默认归还信息中填充归还地址
- ✅ **商户可修改订单归还地址**：在订单状态为 NEW/PAID/TO_SHIP/SHIPPED/IN_RENT 时，商户可以修改订单的归还地址
- ✅ **归还中后锁定**：订单进入 RETURNING 状态后，归还地址不可修改
- ✅ **一键复制功能**：订单详情页面提供格式化的归还地址，方便用户复制到快递单

**订单中的归还地址字段：**
- `return_address` (group): 订单的归还地址
  - 在 RETURNING 之前可编辑（通过 `readOnly` 条件控制）
  - 在 RETURNING 及之后只读
- `return_address_display` (UI field): 显示格式化的归还地址，提供一键复制功能

**2. 用户体验增强**

**商户端功能**：
- 在订单详情页面可以看到"归还地址（便于复制）"卡片
- 点击"复制"按钮一键复制完整归还地址信息
- 复制格式示例：
  ```
  收件人：张三
  电话：13800138000
  地址：广东省深圳市南山区科技园南路15号
  邮编：518000
  ```

**用户端功能**：
- 在订单详情中可以看到归还地址
- 准备归还设备时，可以直接复制归还地址到快递单

**3. 权限控制**

- **商户管理员**：
  - 可以创建/更新/删除自己商户的归还信息
  - 可以在订单 RETURNING 之前修改订单归还地址
- **平台管理员**：
  - 可以查看/修改/删除所有归还信息
- **普通用户**：
  - 可以查看归还信息（用于下单）
  - 不能修改归还信息

**相关文件（后端）：**
- [ReturnInfo Collection](b2b-rental-backend/src/collections/ReturnInfo.ts)
- [Orders Collection - return_address 字段](b2b-rental-backend/src/collections/Orders.ts:352-439)
- [ReturnAddressDisplay 组件](b2b-rental-backend/src/components/ReturnAddressDisplay.tsx)

**相关文件（前端）：**
- [归还地址管理页面](lease-shelf-flow-01487/src/pages/merchant/ReturnAddressManagement.tsx)
- [ReturnAddressSelector 组件](lease-shelf-flow-01487/src/components/ReturnAddressSelector.tsx)
- [returnInfoApi](lease-shelf-flow-01487/src/services/api/returnInfoApi.ts)

**4. 前端实现**

**归还地址管理页面**（`/merchant/return-address`）：
- ✅ 查看商户的所有归还地址
- ✅ 添加新归还地址（联系人、电话、完整地址、邮编、备注）
- ✅ 编辑现有归还地址
- ✅ 删除归还地址
- ✅ 设置默认地址（每个商户只能有一个默认地址）
- ✅ 启用/禁用地址

**订单中的归还地址选择**（`ReturnAddressSelector` 组件）：
- ✅ **从列表选择**：显示商户所有启用的归还地址，支持单选
- ✅ **输入新地址**：手动输入新的归还地址
- ✅ **智能地址解析**：粘贴完整地址后一键解析，自动填充省市区和详细地址
- ✅ **可选保存**：勾选"保存到归还地址列表"，可将新地址保存（方便下次使用）
- ✅ **立即保存**：勾选保存后，点击"立即保存"按钮将地址保存到列表
- ✅ **状态限制**：订单进入 RETURNING 状态后，组件自动变为只读模式

**订单页面集成**（`MyOrders.tsx` - 商户端）：
- ✅ **编辑按钮显示**：在以下订单状态显示"设置归还地址"按钮：
  - PAID（已支付）
  - TO_SHIP（待发货）
  - SHIPPED（已发货）
  - IN_RENT（租赁中）
- ✅ **编辑对话框**：点击按钮打开对话框，集成 `ReturnAddressSelector` 组件
- ✅ **自动加载**：对话框打开时自动加载当前订单的归还地址
- ✅ **保存更新**：通过 `orderApi.updateOrderStatus` 更新订单归还地址
- ✅ **验证逻辑**：必填字段：联系人、电话、省份、城市、详细地址

**订单卡片显示**（`OrderCard.tsx`）：
- ✅ **显示优先级**：
  1. 优先显示 `order.return_address`（订单设置的归还地址）
  2. 降级显示 `sku.return_address`（SKU 默认归还地址）
- ✅ **完整信息展示**：联系人、电话、省市区、详细地址、邮编
- ✅ **悬浮卡片**：使用 HoverCard 展示完整地址
- ✅ **一键复制**：点击复制完整地址信息

**使用场景示例**：

1. **商户首次使用**：
   - 进入"归还地址管理"页面
   - 添加第一个归还地址，勾选"设为默认"
   - 以后创建的订单会自动使用该默认地址

2. **订单修改归还地址（从列表选择）**：
   - 在订单详情中，点击"从列表选择"
   - 选择其他已保存的归还地址
   - 订单归还地址立即更新

3. **订单修改归还地址（使用智能解析）**：
   - 在订单详情中，点击"输入新地址"
   - 在"智能解析地址"区域粘贴完整地址，如：`广东省深圳市南山区科技园南路15号`
   - 点击"解析"按钮，系统自动填充省市区和详细地址
   - 手动输入联系人和电话
   - 可选勾选"保存到归还地址列表"

4. **订单修改归还地址（手动输入）**：
   - 在订单详情中，点击"输入新地址"
   - 手动填写所有字段（联系人、电话、省市区、详细地址、邮编）
   - **不勾选"保存"**：该地址仅用于当前订单
   - **勾选"保存"**：点击"立即保存"后，地址同时保存到列表和应用到订单

### 待对接接口

#### 需要实现的接口

1. **统一的登录响应格式**
   - 后端需要在登录时返回关联的Users信息（含角色）
   - 或提供额外的端点获取业务身份信息

2. **API路径前缀配置**
   - 方案A：前端统一添加`/api`前缀
   - 方案B：后端配置路由重写（推荐）

### 问题记录

#### [严重] API路径前缀不匹配

**发现时间**: 2025-10-19
**问题描述**: 前端调用 `/accounts/login`，但Payload CMS的标准路径是 `/api/accounts/login`

**影响范围**: 所有认证相关接口
**状态**: 待解决

**可能的解决方案**:

1. **后端方案 - 配置路由重写（推荐）**
   ```typescript
   // src/middleware.ts 或 next.config.mjs
   // 添加路由重写规则，将 /accounts/* 重写到 /api/accounts/*
   ```

2. **前端方案 - 修改API配置**
   ```typescript
   // src/config/api.ts
   export const getApiUrl = (path: string) => {
     // 统一添加 /api 前缀
     return `${API_CONFIG.baseURL}/api${path}`;
   };
   ```

#### [中等] 用户数据结构不一致

**问题描述**:
- 后端登录返回Accounts对象（不含role）
- 前端期望User对象（含role、merchant等业务信息）

**影响**: 前端可能无法正确识别用户角色和权限

**建议方案**:
1. 登录接口额外返回关联的Users信息
2. 登录后前端额外调用 `/api/users/me` 获取业务身份
