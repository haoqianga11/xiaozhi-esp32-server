# Manager-API 项目学习计划

## 📊 项目复杂度评估

**项目规模**：
- Java 文件：230 个
- 代码行数：约 14,193 行  
- 业务模块：7 个主要模块
- 技术栈：Spring Boot 3.4.3 + MyBatis-Plus + Shiro + Redis + WebSocket

**复杂度级别**：⭐⭐⭐⭐ (中等偏高)
- ✅ 架构清晰：标准分层架构（Controller-Service-Dao）
- ✅ 模块划分明确：业务模块相对独立
- ⚠️ 中等业务复杂度：设备管理、AI Agent、权限管理
- ⚠️ 依赖较多：多项技术集成

---

## 🎯 五阶段学习路线

### 🚀 第一阶段：环境搭建与项目运行（1-2天）
**目标**：让项目跑起来，了解整体结构

#### 学习内容
1. **项目结构预览**
   - [ ] 查看 `pom.xml` 了解技术栈和依赖
   - [ ] 查看 `application.yml` 了解配置项  
   - [ ] 了解目录结构和模块划分

2. **项目启动**
   暂时不执行

3. **最简单接口体验**
   - [ ] 查看系统参数模块（`sys/SysParamsController`）
   - [ ] 理解最基础的 CRUD 操作

#### 输出成果
- 对项目整体结构有基本认知

---

### 🛠️ 第二阶段：基础架构理解（2-3天）  
**目标**：理解 Spring Boot 项目的通用组件

#### 学习内容
1. **通用基础设施**（`common` 包）
   - [ ] 全局异常处理：`RenExceptionHandler`
   - [ ] 统一返回格式：`Result`、`PageData`
   - [ ] 工具类：`JsonUtils`、`DateUtils`、`ConvertUtils`
   - [ ] 分页处理、参数校验

2. **数据库交互基础**
   - [ ] MyBatis-Plus 配置：`MybatisPlusConfig`
   - [ ] 基础 Entity：`BaseEntity`
   - [ ] 基础 Service：`BaseService`、`CrudService`
   - [ ] 简单的 Dao 和 Mapper XML（如：`SysParamsDao.xml`）

3. **Redis 缓存**
   - [ ] Redis 配置：`RedisConfig`
   - [ ] Redis 工具类：`RedisUtils`
   - [ ] Lua 脚本应用

#### 输出成果
- 理解项目的技术架构
- 掌握通用组件的使用方法
- 能读懂基础的数据库操作

---

### 🔐 第三阶段：认证授权机制（3-4天）
**目标**：理解用户如何登录，API 如何鉴权

#### 学习内容  
1. **登录流程**
   - [ ] `LoginController` - 登录接口实现
   - [ ] `CaptchaService` - 验证码生成和校验
   - [ ] `TokenGenerator` - Token 生成规则
   - [ ] `SysUserTokenService` - Token 管理

2. **权限控制**
   - [ ] `Oauth2Filter` - Token 校验过滤器
   - [ ] `Oauth2Realm` - Shiro 认证授权
   - [ ] `ShiroConfig` - Shiro 安全配置
   - [ ] `SecurityUser` - 当前用户信息获取

3. **服务端密钥校验**
   - [ ] `ServerSecretFilter` - 服务端间通信校验
   - [ ] `ServerSecretToken` - 密钥 Token

#### 输出成果
- 理解完整的用户登录认证流程
- 掌握 API 接口的权限控制机制
- 能够调试和排查鉴权问题

---

### 📱 第四阶段：核心业务模块（4-5天）
**目标**：理解具体业务功能的实现

#### 学习顺序（从简单到复杂）

1. **系统管理模块**（`sys` - 最简单）
   - [ ] `AdminController` - 管理员用户管理
   - [ ] `SysDictTypeController` & `SysDictDataController` - 数据字典
   - [ ] `SysParamsController` - 系统参数配置
   - [ ] 理解典型的增删改查操作

2. **音色管理模块**（`timbre` - 简单）  
   - [ ] `TimbreController` - 音色数据管理
   - [ ] 简单的业务实体操作

3. **设备管理模块**（`device` - 中等复杂）
   - [ ] `DeviceController` - 设备注册、绑定、查询
   - [ ] `OTAController` - OTA 升级功能
   - [ ] 设备上报数据处理流程

4. **模型配置模块**（`model` - 中等复杂）
   - [ ] `ModelProviderController` - 模型供应商管理
   - [ ] `ModelController` - LLM/语音模型配置
   - [ ] 模型参数和配置管理

#### 输出成果
- 掌握标准业务模块的实现模式
- 理解设备管理的核心流程
- 了解模型配置的管理方式

---

### 🤖 第五阶段：高级功能与集成（5-6天）
**目标**：掌握复杂的业务逻辑和第三方集成

#### 学习内容
1. **AI Agent 管理**（`agent` - 最复杂）
   - [ ] `AgentController` - Agent 基础信息管理
   - [ ] `AgentTemplateController` - Agent 模板系统
   - [ ] `AgentChatHistoryController` - 聊天记录管理
   - [ ] `AgentVoicePrintController` - 语音指纹识别
   - [ ] `AgentMcpAccessPointController` - MCP 协议集成
   - [ ] 插件映射和扩展机制

2. **第三方服务集成**
   - [ ] 短信服务（`sms` 模块）- 阿里云 SMS
   - [ ] WebSocket 实时通信 - `WebSocketClientManager`
   - [ ] Redis 在复杂业务中的应用

3. **高级功能**
   - [ ] 服务端行为触发 - `ServerSideManageController`
   - [ ] 异步处理 - `AsyncConfig`
   - [ ] 数据过滤和权限控制 - `DataFilter`

#### 输出成果
- 掌握复杂业务逻辑的设计和实现
- 理解第三方服务的集成方式
- 具备独立开发和维护的能力

---

## 🛠️ 学习方法指南

### 每阶段通用方法
1. **先整体后局部**：先看 Controller 了解接口，再看 Service 了解业务逻辑
2. **跟踪完整请求**：HTTP 请求 → Controller → Service → Dao → 数据库
3. **理解数据流转**：DTO(请求) → Entity(数据库) → VO(响应) 
4. **实际测试接口**：用 Swagger 文档或 Postman 测试

### 推荐工具
- **接口测试**：http://localhost:8002/xiaozhi/doc.html (Knife4j)
- **数据库查看**：MySQL Workbench 或其他数据库工具
- **代码阅读**：IDE 的"查找引用"功能跟踪调用链
- **API 调试**：Postman 或 curl 命令

### 代码阅读技巧
```text
典型的调用链路：
Controller (接收请求) 
    ↓
Service (业务逻辑)
    ↓  
Dao (数据访问)
    ↓
Mapper XML (SQL执行)
    ↓
Database (数据存储)
```

### 关键文件清单
```text
# 配置文件
├── application.yml          # 主配置
├── application-dev.yml      # 开发环境配置
└── pom.xml                 # 项目依赖

# 核心启动
└── AdminApplication.java    # 项目入口

# 基础架构
├── common/                 # 通用组件
├── security/config/        # 安全配置
└── security/oauth2/        # 认证授权

# 业务模块
├── sys/                   # 系统管理
├── device/                # 设备管理  
├── agent/                 # AI Agent
├── model/                 # 模型配置
├── sms/                   # 短信服务
└── timbre/                # 音色管理
```

---

## 📋 学习进度追踪

### 第一阶段 ⏳
- [ ] 项目环境搭建完成
- [ ] 成功启动并访问接口文档
- [ ] 理解项目整体结构

### 第二阶段 ⏳  
- [ ] 掌握通用基础组件
- [ ] 理解数据库交互机制
- [ ] 熟悉 Redis 缓存应用

### 第三阶段 ⏳
- [ ] 理解登录认证流程
- [ ] 掌握权限控制机制
- [ ] 能调试鉴权相关问题

### 第四阶段 ⏳
- [ ] 掌握系统管理模块
- [ ] 理解设备管理流程
- [ ] 了解模型配置管理

### 第五阶段 ⏳
- [ ] 掌握 AI Agent 复杂逻辑
- [ ] 理解第三方服务集成
- [ ] 具备独立开发能力

---

## 📝 学习笔记区域

### 重要概念记录
- [ ] Shiro 权限框架的核心概念
- [ ] MyBatis-Plus 的高级特性
- [ ] Redis 在项目中的应用场景
- [ ] MCP (Model Context Protocol) 协议

### 问题记录
- [ ] 遇到的技术难点
- [ ] 需要深入研究的部分
- [ ] 改进建议和想法

### 扩展学习
- [ ] Spring Boot 3.x 新特性
- [ ] AI Agent 相关技术
- [ ] WebSocket 实时通信
- [ ] 微服务架构演进

---

*最后更新时间：2025-08-28*
*预计总学习时间：15-20 天*
*难度级别：中等偏高 ⭐⭐⭐⭐*
