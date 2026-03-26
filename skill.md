---
name: openapi-auto-generator
description: Use when creating new API endpoints (Controller/Service), after implementation compiles successfully. Auto-generates OpenAPI 3.0 documentation. One file per API endpoint.
---

# OpenAPI 3.0 文档自动生成（单接口模式）

## Overview

新接口开发完成后，自动生成符合 OpenAPI 3.0 标准的文档（YAML 格式），**每个接口单独生成一份文件**，确保 API 文档与代码同步。

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
| 文档格式 | OpenAPI 3.0 (YAML) |
| 存放目录 | `docs/openapi/` |
| 文件命名 | `{operationId}.yaml` |
| 必需字段 | summary, description, requestBody, responses (200/400/401/500) |
| 响应示例 | 必须包含成功和失败场景 |

## 文档结构

每个接口生成独立的 OpenAPI 文档，文件命名规则：
- `docs/openapi/{operationId}.yaml`

例如：
- `docs/openapi/wechatLogin.yaml` - 微信登录接口
- `docs/openapi/getUserProfile.yaml` - 获取用户信息接口
- `docs/openapi/bindPhone.yaml` - 绑定手机号接口

## 单接口文档模板

### GET 接口示例

```yaml
openapi: 3.0.3
info:
  title: 获取用户个人信息
  description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码
  version: 1.0.0

servers:
  - url: http://localhost:8000
    description: 开发环境
  - url: https://api.competition.com
    description: 生产环境

tags:
  - name: 用户
    description: 用户信息相关接口

paths:
  /api/v1/user/profile:
    get:
      tags:
        - 用户
      summary: 获取当前用户个人信息
      description: 获取当前登录用户的基本信息、绑定状态以及自身的邀请码。从JWT Token中解析用户ID，返回用户的完整档案信息。
      operationId: getUserProfile
      security:
        - BearerAuth: []
      responses:
        '200':
          description: 获取成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/StandardResponse'
                  - type: object
                    properties:
                      data:
                        $ref: '#/components/schemas/UserProfileData'
              examples:
                success:
                  summary: 成功示例
                  value:
                    code: 200
                    msg: "success"
                    data:
                      id: 10086
                      area_code: 86
                      openid: "oGZUI0egBJY1zhBYw2KhdUfwVJJE"
                      phone: "13800138000"
                      nickname: "张三"
                      avatar: "https://..."
                      gender: 1
        '401':
          description: 未授权
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
              examples:
                unauthorized:
                  summary: Token无效或过期
                  value:
                    code: 401
                    msg: "无效的Token"
                    data: null
        '500':
          description: 服务器错误
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT Token鉴权，格式：Bearer {token}

  schemas:
    StandardResponse:
      type: object
      required: [code, msg, data]
      properties:
        code:
          type: integer
          example: 200
        msg:
          type: string
          example: "success"
        data:
          type: object
          nullable: true

    ErrorResponse:
      type: object
      required: [code, msg, data]
      properties:
        code:
          type: integer
          example: 400
        msg:
          type: string
          example: "参数错误"
        data:
          type: object
          nullable: true
          example: null

    UserProfileData:
      type: object
      properties:
        id:
          type: integer
          format: int64
          example: 10086
        area_code:
          type: integer
          format: int32
          minimum: 0
          example: 86
        nickname:
          type: string
          example: "张三"
```

### POST 接口示例

```yaml
openapi: 3.0.3
info:
  title: 绑定手机号
  description: 接收小程序端获取的手机号授权code，由后端向微信服务器换取真实手机号并更新至用户表
  version: 1.0.0

servers:
  - url: http://localhost:8000
    description: 开发环境
  - url: https://api.competition.com
    description: 生产环境

tags:
  - name: 用户
    description: 用户信息相关接口

paths:
  /api/v1/user/bind-phone:
    post:
      tags:
        - 用户
      summary: 绑定手机号
      description: 接收小程序端获取的手机号授权code，由后端向微信服务器换取真实手机号并更新至用户表。支持更改手机号，且需校验手机号是否已被其他用户绑定。同时保存微信返回的地区码。
      operationId: bindPhone
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [phone_code]
              properties:
                phone_code:
                  type: string
                  description: 小程序<button open-type="getPhoneNumber">返回的动态code
                  example: "071aBcDe23fGHi45678jKlMnO_pHxyz"
            examples:
              normal:
                summary: 正常绑定
                value:
                  phone_code: "071aBcDe23fGHi45678jKlMnO_pHxyz"
      responses:
        '200':
          description: 绑定成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/StandardResponse'
                  - type: object
                    properties:
                      data:
                        $ref: '#/components/schemas/BindPhoneData'
              examples:
                success:
                  summary: 绑定成功
                  value:
                    code: 200
                    msg: "绑定成功"
                    data:
                      phone: "13812341234"

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    StandardResponse:
      type: object
      required: [code, msg, data]
      properties:
        code: {type: integer, example: 200}
        msg: {type: string, example: "success"}
        data: {type: object, nullable: true}

    BindPhoneData:
      type: object
      properties:
        phone:
          type: string
          example: "13812341234"
```

### 字段类型映射

| Go 类型 | OpenAPI 类型 |
|---------|--------------|
| string | string |
| int, int64 | integer, format: int64 |
| uint32 | integer, format: int32, minimum: 0 |
| float64 | number |
| bool | boolean |
| time.Time | string, format: date-time |
| *T | type, nullable: true |
| []T | array, items: type: T |

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
   - 填充完整的 OpenAPI 规范（info, servers, tags, paths, components）
   - 添加 examples 示例

4. **验证文档**
   - 确保 YAML 格式正确
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
| 缺少 401 响应 | 需要鉴权的接口必须有 401 响应 |
| description 过于简单 | 必须描述接口功能和适用场景 |
| 参数没有 example | 每个参数必须有示例值 |
| *int64 没有标记 nullable | 指针类型必须加 nullable: true |
| 单接口文档缺少 components | 每个文档都必须是完整的 OpenAPI 规范 |

## Red Flags

- 写完接口就直接提交，没有更新文档
- 忘记添加 security: BearerAuth（需要鉴权的接口）
- 响应示例数据结构与实际不一致
- 单接口文档中引用了不存在的 schema
