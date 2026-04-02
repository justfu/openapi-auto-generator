---
name: openapi-auto-generator
description: Use when creating new API endpoints (Controller/Service), after implementation compiles successfully. Auto-generates OpenAPI 3.1 documentation. One file per API endpoint.
---

# OpenAPI 3.1 文档自动生成（单接口模式）

## Overview

新接口开发完成后，自动生成符合 OpenAPI 3.1 标准的文档（YAML 格式），**每个接口单独生成一份文件**，确保 API 文档与代码同步。

## When to Use

```
新接口开发完成
       │
       ├── Model 层 ✓
       ├── Service 层 ✓
       ├── Controller 层 ✓
       ├── 路由注册 ✓
       └── 编译通过 ✓
              │
              ▼
      立即生成 OpenAPI 文档
```

**使用时机：**
- ✅ 新增 POST/GET/PUT/DELETE 接口后
- ✅ 接口参数变更后
- ✅ 响应结构修改后

**不需要：**
- ❌ 仅修改内部逻辑（接口签名不变）
- ❌ 数据库 Model 变更（不影响 API）

## Quick Reference

| 项目 | 说明 |
|------|------|
| 文档格式 | OpenAPI 3.1 (YAML) |
| 存放目录 | `docs/openapi/` |
| 文件命名 | `{operationId}.yaml` |
| 必需字段 | summary, description, requestBody, responses (200/400/401/500) |
| Security | 根级别定义，不需要鉴权的接口单独 `security: []` 覆盖 |
| 可空字段 | 使用 `type: ['string', 'null']` 而非 `nullable: true` |

## 文档结构

每个接口生成独立的 OpenAPI 文档，文件命名规则：
- `docs/openapi/{operationId}.yaml`

例如：
- `docs/openapi/wechatLogin.yaml` - 微信登录接口
- `docs/openapi/getUserProfile.yaml` - 获取用户信息接口
- `docs/openapi/bindPhone.yaml` - 绑定手机号接口

## 文档格式规范

### 结构顺序

```yaml
openapi: '3.1.0'
info:
  version: '1.0.0'
  title: '接口标题'
  description: 接口描述
security:              # 鉴权方案，放在根级别
  - TokenAuth: []
paths:
  /api/v1/...:
    parameters:         # 路径参数放在 path 级别，用 $ref 引用 components/schemas
    get:
      summary: ...
      responses:
components:
  schemas:              # 所有数据模型
  securitySchemes:      # 鉴权方案定义
```

### 关键规则

1. **无 `servers` 段** - 不在文档中硬编码服务器地址
2. **无 `tags` 段** - 不在文档中使用标签分组
3. **Security 在根级别** - 需要鉴权的接口默认使用根级别 security；不需要鉴权的接口在 operation 中用 `security: []` 覆盖
4. **路径参数提升到 path 级别** - 路径参数（in: path）放在 path 对象下，通过 `$ref` 引用 components/schemas 中的类型定义
5. **无 `examples` 块** - 不在 schema 中添加 examples
6. **无 `allOf` 合并** - 响应直接用 `$ref` 引用 schema，不使用 allOf 合并 StandardResponse
7. **可空类型** - OpenAPI 3.1 中使用 `type: ['string', 'null']` 代替已废弃的 `nullable: true`
8. **字符串值加引号** - `openapi: '3.1.0'`、`version: '1.0.0'` 等版本号字符串必须加引号

## 单接口文档模板

### GET 接口示例（带路径参数）

```yaml
openapi: '3.1.0'
info:
  version: '1.0.0'
  title: '获取用户个人信息'
  description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码

security:
  - TokenAuth: []

paths:
  /api/v1/user/{userId}/profile:
    parameters:
      - name: userId
        description: The unique identifier of the user
        in: path
        required: true
        schema:
          $ref: '#/components/schemas/UserId'
    get:
      summary: 获取当前用户个人信息
      description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码。从JWT Token中解析用户ID，返回用户的完整档案信息。
      responses:
        '200':
          description: 获取成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserProfile'
        '401':
          description: 未授权，Token无效或过期
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: 服务器错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    UserId:
      description: The unique identifier of a user
      type: integer
      format: int64

    UserProfile:
      description: 用户个人信息
      type: object
      required:
        - id
        - nickname
      properties:
        id:
          $ref: '#/components/schemas/UserId'
        area_code:
          description: 手机国际区号
          type: integer
          format: int32
          minimum: 0
        openid:
          description: 微信OpenID
          type: string
        phone:
          description: 手机号
          type:
            - string
            - 'null'
        nickname:
          description: 昵称
          type: string
        avatar:
          description: 头像URL
          type:
            - string
            - 'null'
        gender:
          description: 性别 1-男 2-女
          type:
            - integer
            - 'null'

    Error:
      description: 错误响应
      type: object
      required:
        - code
        - msg
      properties:
        code:
          description: 错误码
          type: integer
        msg:
          description: 错误信息
          type: string

  securitySchemes:
    TokenAuth:
      type: apiKey
      in: header
      name: token
      description: 在请求头中直接传递 token 参数进行鉴权
```

### GET 接口示例（无路径参数）

```yaml
openapi: '3.1.0'
info:
  version: '1.0.0'
  title: '获取用户个人信息'
  description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码

security:
  - TokenAuth: []

paths:
  /api/v1/user/profile:
    get:
      summary: 获取当前用户个人信息
      description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码。从JWT Token中解析用户ID，返回用户的完整档案信息。
      responses:
        '200':
          description: 获取成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserProfile'
        '401':
          description: 未授权，Token无效或过期
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: 服务器错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    UserProfile:
      description: 用户个人信息
      type: object
      required:
        - id
        - nickname
      properties:
        id:
          description: 用户ID
          type: integer
          format: int64
        nickname:
          description: 昵称
          type: string

    Error:
      description: 错误响应
      type: object
      required:
        - code
        - msg
      properties:
        code:
          description: 错误码
          type: integer
        msg:
          description: 错误信息
          type: string

  securitySchemes:
    TokenAuth:
      type: apiKey
      in: header
      name: token
      description: 在请求头中直接传递 token 参数进行鉴权
```

### POST 接口示例

```yaml
openapi: '3.1.0'
info:
  version: '1.0.0'
  title: '绑定手机号'
  description: 接收小程序端获取的手机号授权code，由后端向微信服务器换取真实手机号并更新至用户表

security:
  - TokenAuth: []

paths:
  /api/v1/user/bind-phone:
    post:
      summary: 绑定手机号
      description: 接收小程序端获取的手机号授权code，由后端向微信服务器换取真实手机号并更新至用户表。支持更改手机号，且需校验手机号是否已被其他用户绑定。
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/BindPhoneRequest'
      responses:
        '200':
          description: 绑定成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BindPhoneResponse'
        '401':
          description: 未授权，Token无效或过期
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: 服务器错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    BindPhoneRequest:
      description: 绑定手机号请求
      type: object
      required:
        - phone_code
      properties:
        phone_code:
          description: 小程序<button open-type="getPhoneNumber">返回的动态code
          type: string

    BindPhoneResponse:
      description: 绑定手机号响应
      type: object
      required:
        - phone
      properties:
        phone:
          description: 绑定的手机号
          type: string

    Error:
      description: 错误响应
      type: object
      required:
        - code
        - msg
      properties:
        code:
          description: 错误码
          type: integer
        msg:
          description: 错误信息
          type: string

  securitySchemes:
    TokenAuth:
      type: apiKey
      in: header
      name: token
      description: 在请求头中直接传递 token 参数进行鉴权
```

### 无需鉴权接口示例

```yaml
openapi: '3.1.0'
info:
  version: '1.0.0'
  title: '微信登录'
  description: 通过微信小程序code获取或创建用户，返回JWT Token

paths:
  /api/v1/auth/wechat-login:
    post:
      summary: 微信登录
      description: 通过微信小程序的登录code换取openid，查询或创建用户，返回JWT Token用于后续接口鉴权。
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/WechatLoginRequest'
      responses:
        '200':
          description: 登录成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/WechatLoginResponse'
        '400':
          description: 参数错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: 服务器错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

components:
  schemas:
    WechatLoginRequest:
      description: 微信登录请求
      type: object
      required:
        - code
      properties:
        code:
          description: 微信小程序wx.login()获取的临时登录凭证
          type: string

    WechatLoginResponse:
      description: 微信登录响应
      type: object
      required:
        - token
      properties:
        token:
          description: JWT Token，用于后续接口鉴权
          type: string

    Error:
      description: 错误响应
      type: object
      required:
        - code
        - msg
      properties:
        code:
          description: 错误码
          type: integer
        msg:
          description: 错误信息
          type: string
```

### 字段类型映射

#### Go 类型映射

| Go 类型 | OpenAPI 3.1 类型 |
|---------|-------------------|
| string | `type: string` |
| int, int64 | `type: integer`, `format: int64` |
| int32 | `type: integer`, `format: int32` |
| uint32 | `type: integer`, `format: int32`, `minimum: 0` |
| float64 | `type: number` |
| float32 | `type: number`, `format: float` |
| bool | `type: boolean` |
| time.Time | `type: string`, `format: date-time` |
| *T | `type: [T, 'null']` （数组类型表示可空） |
| []T | `type: array`, `items: { type: T }` |
| map[string]T | `type: object`, `additionalProperties: { type: T }` |
| interface{} | `type: object` |

#### Java 类型映射

| Java 类型 | OpenAPI 3.1 类型 |
|-----------|-------------------|
| String | `type: string` |
| int, Integer | `type: integer`, `format: int32` |
| long, Long | `type: integer`, `format: int64` |
| short, Short | `type: integer`, `format: int32` |
| double, Double | `type: number`, `format: double` |
| float, Float | `type: number`, `format: float` |
| BigDecimal | `type: number` |
| boolean, Boolean | `type: boolean` |
| LocalDate | `type: string`, `format: date` |
| LocalTime | `type: string`, `format: time` |
| LocalDateTime | `type: string`, `format: date-time` |
| Date | `type: string`, `format: date-time` |
| OffsetDateTime | `type: string`, `format: date-time` |
| ZonedDateTime | `type: string`, `format: date-time` |
| UUID | `type: string`, `format: uuid` |
| List\<T\> | `type: array`, `items: { type: T }` |
| Set\<T\> | `type: array`, `items: { type: T }`, `uniqueItems: true` |
| Map\<String, T\> | `type: object`, `additionalProperties: { type: T }` |
| Object | `type: object` |
| Optional\<T\> | `type: [T, 'null']` |
| @Nullable T | `type: [T, 'null']` |
| Enum | `type: string`, `enum: [value1, value2, ...]` |

#### PHP 类型映射

| PHP 类型 | OpenAPI 3.1 类型 |
|----------|-------------------|
| string | `type: string` |
| int | `type: integer`, `format: int64` |
| float | `type: number`, `format: double` |
| bool | `type: boolean` |
| array | `type: array`, `items: { type: T }` |
| object | `type: object` |
| null | `type: 'null'` |
| mixed | 不映射，需根据上下文确定 |
| DateTime | `type: string`, `format: date-time` |
| DateTimeInterface | `type: string`, `format: date-time` |
| Carbon | `type: string`, `format: date-time` |
| ?T（可空类型） | `type: [T, 'null']` |
| Collection | `type: array`, `items: { type: T }` |
| enum | `type: string`, `enum: [value1, value2, ...]`（PHP 8.1+） |
| backed enum (int) | `type: integer`, `enum: [1, 2, ...]` |

#### Python 类型映射

| Python 类型 | OpenAPI 3.1 类型 |
|-------------|-------------------|
| str | `type: string` |
| int | `type: integer`, `format: int64` |
| float | `type: number`, `format: double` |
| bool | `type: boolean` |
| bytes | `type: string`, `format: byte` |
| decimal.Decimal | `type: number` |
| datetime.date | `type: string`, `format: date` |
| datetime.time | `type: string`, `format: time` |
| datetime.datetime | `type: string`, `format: date-time` |
| uuid.UUID | `type: string`, `format: uuid` |
| list\[T\] | `type: array`, `items: { type: T }` |
| tuple\[T, ...\] | `type: array`, `items: { type: T }` |
| set\[T\] | `type: array`, `items: { type: T }`, `uniqueItems: true` |
| dict\[str, T\] | `type: object`, `additionalProperties: { type: T }` |
| Optional\[T\] | `type: [T, 'null']` |
| T \| None | `type: [T, 'null']` |
| Union\[T1, T2\] | `oneOf: [{ type: T1 }, { type: T2 }]` |
| Literal\["a", "b"\] | `type: string`, `enum: ["a", "b"]` |
| Enum | `type: string` 或 `type: integer`, `enum: [...]` |

### 可空类型说明（OpenAPI 3.1）

OpenAPI 3.1 移除了 `nullable: true`，改用类型数组：

```yaml
# 旧写法（3.0）
phone:
  type: string
  nullable: true

# 新写法（3.1）
phone:
  type:
    - string
    - 'null'
```

## Implementation Steps

1. **分析接口签名**
   - 读取 Controller 中的路由定义
   - 确认请求方法（GET/POST/PUT/DELETE）
   - 确认请求参数（Query/Path/Body）
   - 确认是否需要鉴权

2. **提取请求/响应模型**
   - 读取 requestModel 中的请求体结构
   - 读取 responseModel 中的响应体结构

3. **生成单接口文档**
   - 创建 `docs/openapi/{operationId}.yaml`
   - 按照格式规范填充：openapi, info, security, paths, components
   - 路径参数放在 path 级别并用 `$ref` 引用 components/schemas
   - 请求体和响应体也用 `$ref` 引用 components/schemas
   - 可空字段使用 `type: [T, 'null']` 格式

4. **验证文档**
   - 确保 YAML 格式正确
   - 确保 schema 引用完整（所有 `$ref` 指向的 schema 都在 components/schemas 中定义）
   - 导入 Apifox 或其他工具测试

## 目录结构

```
docs/
└── openapi/
    ├── wechatLogin.yaml
    ├── getUserProfile.yaml
    ├── bindPhone.yaml
    └── updateUserProfile.yaml
```

## Common Mistakes

| 错误 | 正确做法 |
|------|----------|
| 使用 `openapi: 3.0.3` | 必须使用 `openapi: '3.1.0'` |
| 使用 `nullable: true` | 使用 `type: [T, 'null']` 数组格式 |
| 使用 `servers` 段 | OpenAPI 3.1 文档不包含 servers |
| 使用 `tags` 段 | OpenAPI 3.1 文档不包含 tags |
| Security 放在 operation 级别 | 需要鉴权的接口使用根级别 security |
| 路径参数放在 operation 内 | 路径参数必须放在 path 级别 |
| 使用 `allOf` 合并响应 | 直接用 `$ref` 引用完整 schema |
| 添加 `examples` 块 | 不添加 examples，保持简洁 |
| 指针类型没有标记可空 | Go 的 `*T` 类型必须用 `type: [T, 'null']` |
| 单接口文档缺少 components | 每个文档都必须包含完整的 components |
| $ref 引用的 schema 未定义 | 所有引用的 schema 必须在 components/schemas 中定义 |

## Red Flags

- 写完接口就直接提交，没有更新文档
- 需要鉴权的接口没有在根级别配置 security
- 不需要鉴权的接口没有在 operation 中用 `security: []` 覆盖
- 响应 schema 与实际代码中的响应结构不一致
- 单接口文档中引用了不存在的 schema
