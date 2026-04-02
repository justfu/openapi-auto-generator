# OpenAPI 3.1 Auto Generator Skill

OpenAPI 3.1 文档自动生成 Claude Code Skill，用于在新接口开发完成后自动生成符合 OpenAPI 3.1 标准的 API 文档。

## 功能特性

- **单接口模式** - 每个接口独立生成一份 YAML 文档，粒度精细，便于管理
- **OpenAPI 3.1 标准** - 使用最新的 OpenAPI 3.1 规范，采用类型数组表示可空字段
- **自动同步** - 接口开发完成后立即生成，确保文档与代码一致
- **完整类型映射** - 自动将 Go 类型转换为 OpenAPI 类型定义
- **Apifox 兼容** - 生成的文档可直接导入 Apifox 进行测试

## 使用时机

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

**适用于：**
- 新增 POST / GET / PUT / DELETE 接口
- 接口参数发生变更
- 响应结构发生修改

**不适用于：**
- 仅修改内部逻辑（接口签名不变）
- 数据库 Model 变更（不影响 API 层）

## 输出规范

| 项目         | 说明                                         |
| ------------ | -------------------------------------------- |
| 文档格式     | OpenAPI 3.1 (YAML)                           |
| 存放目录     | `docs/openapi/`                              |
| 文件命名     | `{operationId}.yaml`                         |
| 必需响应码   | 200 / 400 / 401 / 500                        |
| 可空字段表示 | `type: ['string', 'null']`（非 `nullable`）  |

## 目录结构

```
docs/
└── openapi/
    ├── wechatLogin.yaml
    ├── getUserProfile.yaml
    ├── bindPhone.yaml
    └── updateUserProfile.yaml
```

## 文档结构

每份文档遵循以下结构顺序：

```yaml
openapi: '3.1.0'        # 版本号加引号
info:                    # 接口标题与描述
security:                # 根级别鉴权配置（需要鉴权的接口）
paths:                   # 路径与操作定义
components:              # 数据模型与安全方案
  schemas:
  securitySchemes:
```

### 关键规则

| 规则                     | 说明                                                 |
| ------------------------ | ---------------------------------------------------- |
| 无 `servers` 段          | 不硬编码服务器地址                                   |
| 无 `tags` 段             | 不使用标签分组                                       |
| Security 在根级别        | 需要鉴权的接口默认启用；不需要的用 `security: []` 覆盖 |
| 路径参数在 path 级别     | 通过 `$ref` 引用 components/schemas                  |
| 无 `examples` 块         | 保持文档简洁                                         |
| 无 `allOf` 合并          | 响应直接用 `$ref` 引用                               |
| 可空类型用数组           | `type: [T, 'null']` 代替已废弃的 `nullable: true`    |
| 字符串版本号加引号       | `openapi: '3.1.0'`、`version: '1.0.0'`               |

## 支持的接口类型

### GET 接口（带路径参数）

```yaml
paths:
  /api/v1/user/{userId}/profile:
    parameters:
      - name: userId
        in: path
        required: true
        schema:
          $ref: '#/components/schemas/UserId'
    get:
      summary: 获取用户信息
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserProfile'
```

### 鉴权方式

所有需要鉴权的接口使用 Header 直接传递 `token` 参数：

```yaml
security:
  - TokenAuth: []

components:
  securitySchemes:
    TokenAuth:
      type: apiKey
      in: header
      name: token
      description: 在请求头中直接传递 token 参数进行鉴权
```

### POST 接口

```yaml
paths:
  /api/v1/user/bind-phone:
    post:
      summary: 绑定手机号
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
```

### 无需鉴权接口

```yaml
paths:
  /api/v1/auth/wechat-login:
    post:
      security: []        # 覆盖根级别 security
      summary: 微信登录
```

## 多语言类型映射

Skill 内置 Go、Java、PHP、Python 四种语言的类型映射表，覆盖常用数据类型：

### Go

| Go 类型     | OpenAPI 3.1 类型                                  |
| ----------- | ------------------------------------------------- |
| `string`    | `type: string`                                    |
| `int/int64` | `type: integer, format: int64`                    |
| `uint32`    | `type: integer, format: int32, minimum: 0`        |
| `float64`   | `type: number`                                    |
| `bool`      | `type: boolean`                                   |
| `time.Time` | `type: string, format: date-time`                 |
| `*T`        | `type: [T, 'null']`                               |
| `[]T`       | `type: array, items: { type: T }`                 |

### Java

| Java 类型            | OpenAPI 3.1 类型                                   |
| -------------------- | -------------------------------------------------- |
| `String`             | `type: string`                                     |
| `int/Integer`        | `type: integer, format: int32`                     |
| `long/Long`          | `type: integer, format: int64`                     |
| `double/Double`      | `type: number, format: double`                     |
| `float/Float`        | `type: number, format: float`                      |
| `BigDecimal`         | `type: number`                                     |
| `boolean/Boolean`    | `type: boolean`                                    |
| `LocalDate`          | `type: string, format: date`                       |
| `LocalDateTime`      | `type: string, format: date-time`                  |
| `UUID`               | `type: string, format: uuid`                       |
| `List<T>`            | `type: array, items: { type: T }`                  |
| `Map<String, T>`     | `type: object, additionalProperties: { type: T }`  |
| `Optional<T>`        | `type: [T, 'null']`                                |
| `@Nullable T`        | `type: [T, 'null']`                                |
| `Enum`               | `type: string, enum: [...]`                        |

### PHP

| PHP 类型               | OpenAPI 3.1 类型                                   |
| ---------------------- | -------------------------------------------------- |
| `string`               | `type: string`                                     |
| `int`                  | `type: integer, format: int64`                     |
| `float`                | `type: number, format: double`                     |
| `bool`                 | `type: boolean`                                    |
| `array`                | `type: array, items: { type: T }`                  |
| `DateTime/Carbon`      | `type: string, format: date-time`                  |
| `?T`（可空类型）        | `type: [T, 'null']`                                |
| `Collection`           | `type: array, items: { type: T }`                  |
| `enum`（PHP 8.1+）     | `type: string, enum: [...]`                        |

### Python

| Python 类型              | OpenAPI 3.1 类型                                   |
| ------------------------ | -------------------------------------------------- |
| `str`                    | `type: string`                                     |
| `int`                    | `type: integer, format: int64`                     |
| `float`                  | `type: number, format: double`                     |
| `bool`                   | `type: boolean`                                    |
| `bytes`                  | `type: string, format: byte`                       |
| `decimal.Decimal`        | `type: number`                                     |
| `datetime.date`          | `type: string, format: date`                       |
| `datetime.datetime`      | `type: string, format: date-time`                  |
| `uuid.UUID`              | `type: string, format: uuid`                       |
| `list[T]`                | `type: array, items: { type: T }`                  |
| `dict[str, T]`           | `type: object, additionalProperties: { type: T }`  |
| `Optional[T] / T \| None` | `type: [T, 'null']`                              |
| `Literal["a", "b"]`      | `type: string, enum: ["a", "b"]`                   |
| `Enum`                   | `type: string/integer, enum: [...]`                |

## 常见错误

| 错误                         | 正确做法                                    |
| ---------------------------- | ------------------------------------------- |
| 使用 `openapi: 3.0.3`        | 必须使用 `openapi: '3.1.0'`                 |
| 使用 `nullable: true`        | 使用 `type: [T, 'null']`                    |
| 包含 `servers` 段            | OpenAPI 3.1 文档不包含 servers              |
| 包含 `tags` 段               | OpenAPI 3.1 文档不包含 tags                 |
| Security 放在 operation 级别 | 需要鉴权的接口使用根级别 security            |
| 路径参数放在 operation 内    | 路径参数必须放在 path 级别                   |
| 使用 `allOf` 合并响应        | 直接用 `$ref` 引用                          |
| 添加 `examples` 块           | 不添加 examples，保持简洁                    |

## 触发方式

当 Claude Code 检测到以下场景时自动触发此 Skill：

1. 新增 Controller / Service 接口且编译通过后
2. 接口参数或响应结构发生变更时
3. 用户明确要求生成 OpenAPI 文档时
