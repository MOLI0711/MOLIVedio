# API 设计规范
## RESTful 规范
### HTTP 方法使用
| 方法   | 用途             | 示例                                       |
| ------ | ---------------- | ------------------------------------------ |
| GET    | 查询资源         | `GET /api/users/{id}` - 查询用户信息       |
| POST   | 创建资源         | `POST /api/users` - 创建用户               |
| PUT    | 更新资源（全量） | `PUT /api/users/{id}` - 更新用户全部信息   |
| PATCH  | 更新资源（部分） | `PATCH /api/users/{id}` - 更新用户部分信息 |
| DELETE | 删除资源         | `DELETE /api/users/{id}` - 删除用户        |
### URL 设计规范
#### 基本原则
1. **使用名词复数**：`/api/users` 而不是 `/api/user`
2. **使用小写字母**：`/api/order-items` 而不是 `/api/OrderItems`
3. **使用连字符**：`/api/order-items` 而不是 `/api/order_items`
4. **避免动词**：`POST /api/users` 而不是 `/api/createUser`
5. **资源嵌套不超过2层**：`/api/users/{userId}/orders` ✅，`/api/a/b/c/d` ❌
#### URL 示例
```
# 用户相关
GET    /api/users              # 查询用户列表
GET    /api/users/{id}         # 查询单个用户
POST   /api/users              # 创建用户
PUT    /api/users/{id}         # 更新用户
DELETE /api/users/{id}         # 删除用户
# 用户订单（嵌套资源）
GET    /api/users/{userId}/orders        # 查询用户的订单列表
GET    /api/users/{userId}/orders/{orderId}  # 查询用户的某个订单
# 特殊操作（必须用动词时）
POST   /api/users/{id}/activate   # 激活用户
POST   /api/orders/{id}/pay       # 支付订单
```
### 查询参数规范
```
# 分页
GET /api/users?page=1&size=20
# 排序
GET /api/users?sort=created_at&order=desc
# 过滤
GET /api/users?status=active&role=admin
# 搜索
GET /api/users?keyword=张三
# 指定返回字段
GET /api/users?fields=id,name,email
```
## 请求格式
### 请求头（Header）
```http
Content-Type: application/json
Authorization: Bearer {access_token}
X-Request-ID: {唯一请求ID，用于链路追踪}
```
### 请求体（Body）
```json
{
"name": "张三",
"email": "zhangsan@example.com",
"age": 25,
"roles": ["user", "admin"]
}
```
**规范要求**：
- 使用 **camelCase** 命名（与 Java 一致）
- 日期格式统一使用 **ISO 8601**：`2024-01-07T10:00:00Z`
- 金额使用 **分**（整数），不使用浮点数
## 响应格式
### 统一响应结构
```json
{
"code": 200,
"message": "success",
"data": {
"id": 123,
"name": "张三"
},
"timestamp": "2024-01-07T10:00:00Z",
"requestId": "abc-123-def"
}
```
### 字段说明
| 字段      | 类型   | 说明        | 必需 |
| --------- | ------ | ----------- | ---- |
| code      | number | 业务状态码  | ✅   |
| message   | string | 业务消息    | ✅   |
| data      | any    | 业务数据    | ❌   |
| timestamp | string | 响应时间戳  | ✅   |
| requestId | string | 请求追踪 ID | ✅   |
### 分页响应格式
```json
{
"code": 200,
"message": "success",
"data": {
"items": [
{ "id": 1, "name": "用户1" },
{ "id": 2, "name": "用户2" }
],
"pagination": {
"page": 1,
"size": 20,
"total": 100,
"totalPages": 5
}
}
}
```
## 状态码规范
### HTTP 状态码
| 状态码 | 含义                  | 使用场景               |
| ------ | --------------------- | ---------------------- |
| 200    | OK                    | 请求成功               |
| 201    | Created               | 创建成功               |
| 204    | No Content            | 删除成功（无返回内容） |
| 400    | Bad Request           | 参数错误               |
| 401    | Unauthorized          | 未登录或 Token 无效    |
| 403    | Forbidden             | 无权限访问             |
| 404    | Not Found             | 资源不存在             |
| 409    | Conflict              | 资源冲突（如重复创建） |
| 429    | Too Many Requests     | 请求过于频繁           |
| 500    | Internal Server Error | 服务器内部错误         |
| 503    | Service Unavailable   | 服务不可用             |
### 业务状态码（code 字段）
| Code | 说明         | HTTP Status |
| ---- | ------------ | ----------- |
| 200  | 成功         | 200         |
| 400  | 参数错误     | 400         |
| 401  | 未登录       | 401         |
| 403  | 无权限       | 403         |
| 404  | 资源不存在   | 404         |
| 409  | 资源冲突     | 409         |
| 1001 | 用户名已存在 | 400         |
| 1002 | 密码错误     | 400         |
| 2001 | 余额不足     | 400         |
| 2002 | 库存不足     | 400         |
| 5000 | 系统错误     | 500         |
**业务状态码规则**：
- `200-599`：标准 HTTP 状态码
- `1000-1999`：用户相关错误
- `2000-2999`：订单/交易相关错误
- `3000-3999`：商品相关错误
- `5000-5999`：系统错误
## 错误响应
### 错误响应格式
```json
{
"code": 400,
"message": "参数错误",
"data": null,
"error": {
"type": "ValidationError",
"details": [
{
"field": "email",
"message": "邮箱格式不正确"
},
{
"field": "age",
"message": "年龄必须大于0"
}
]
},
"timestamp": "2024-01-07T10:00:00Z",
"requestId": "abc-123-def"
}
```
### 常见错误示例
```json
[
{
"code": 400,
"message": "参数错误：email 不能为空"
},
{
"code": 401,
"message": "未登录，请先登录"
},
{
"code": 403,
"message": "无权限访问该资源"
},
{
"code": 404,
"message": "用户不存在"
},
{
"code": 5000,
"message": "系统错误，请稍后重试"
}
]
```
## 认证和鉴权
### 认证方式
使用 **JWT Token** 进行认证：
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
### Token 刷新
```
POST /api/auth/refresh
Content-Type: application/json
{
  "refreshToken": "xxx"
}
```
响应：
```json
{
"code": 200,
"message": "success",
"data": {
"accessToken": "新的访问Token",
"refreshToken": "新的刷新Token",
"expiresIn": 3600
}
}
```
## API 版本管理
### 版本号规范
- 使用 URL 路径携带版本号：`/api/v1/users`
- 大版本变更时升级版本号：`v1` → `v2`
- 向后兼容的小改动不升级版本
### 版本示例
```
# V1 版本
GET /api/v1/users
# V2 版本（重大变更）
GET /api/v2/users
```
## 文档规范
### 必须提供的文档内容
1. **接口描述**：接口用途和业务场景
2. **请求参数**：字段名称、类型、是否必需、说明
3. **响应字段**：字段名称、类型、说明
4. **错误码**：可能的错误码和含义
5. **示例**：完整的请求和响应示例
### Swagger / OpenAPI 规范
建议使用 Swagger 注解自动生成 API 文档
---
**重要提示**：
- 本文件定义的是团队统一的 API 规范
- 所有后端接口必须严格遵守
- 新增业务错误码需在团队内评审
- API 设计时优先考虑前端使用便利性
