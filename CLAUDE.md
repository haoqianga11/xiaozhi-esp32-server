# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

小智 ESP32 服务器是一个基于人机共生智能理论的多模态 AI 语音交互系统，为 ESP32 硬件设备提供后端服务。系统支持语音识别、自然语言处理、语音合成、视觉感知、声纹识别等功能。

## 项目架构

### 核心组件架构

项目采用微服务架构，包含以下主要组件：

1. **Python 核心服务** (`main/xiaozhi-server/`)
   - WebSocket 服务器：处理与 ESP32 设备的实时通信
   - HTTP 服务器：提供 REST API 接口
   - 插件化提供商系统：支持多种 AI 服务提供商

2. **Java 管理后台** (`main/manager-api/`)
   - Spring Boot 应用
   - 提供设备管理、用户管理、配置管理等功能
   - 使用 MySQL 数据库存储

3. **Web 管理界面** (`main/manager-web/`)
   - Vue 2.x 应用
   - Element UI 组件库
   - 提供后台管理界面

4. **移动端管理** (`main/manager-mobile/`)
   - uni-app 应用
   - 支持多端部署（H5、小程序、App）

### 核心模块说明

#### 提供商系统 (`core/providers/`)

- **ASR（语音识别）**: 支持阿里云、百度、豆包、FunASR 等
- **LLM（大语言模型）**: 支持 OpenAI、智谱、豆包、Gemini 等
- **TTS（语音合成）**: 支持阿里云、腾讯云、Edge TTS、FishSpeech 等
- **VAD（语音活动检测）**: Silero VAD
- **VLLM（视觉大模型）**: 支持多模态视觉理解
- **Intent（意图识别）**: 函数调用和 LLM 意图识别
- **Memory（记忆系统）**: 本地短期记忆和 mem0ai 接口
- **Tools（工具调用）**: 支持 IoT、MCP、插件等工具调用

#### 工具系统 (`core/tools/`)

- **device_iot**: IoT 设备控制
- **device_mcp**: MCP 客户端协议
- **mcp_endpoint**: MCP 接入点协议
- **server_mcp**: 服务端 MCP 协议
- **server_plugins**: 服务器插件系统

## 开发命令

### Python 服务开发

```bash
# 进入 Python 服务目录
cd main/xiaozhi-server

# 安装依赖
pip install -r requirements.txt

# 启动服务
python app.py

# 性能测试
python performance_tester.py

# 运行测试工具
# 在浏览器中打开 test/test_page.html
```

### Java 后台开发

```bash
# 进入 Java API 目录
cd main/manager-api

# 使用 Maven 编译和运行
mvn clean compile
mvn spring-boot:run

# 跳过测试打包
mvn clean package -DskipTests
```

### Web 界面开发

```bash
# 进入 Web 目录
cd main/manager-web

# 安装依赖
npm install

# 开发模式
npm run serve

# 构建生产版本
npm run build

# 代码分析
npm run analyze
```

### 移动端开发

```bash
# 进入移动端目录
cd main/manager-mobile

# 使用 pnpm 安装依赖
pnpm install

# 开发模式
pnpm dev

# 构建生产版本
pnpm build

# 类型检查
pnpm type-check

# 代码检查
pnpm lint
pnpm lint:fix
```

## 配置管理

### 配置文件优先级

1. `data/.config.yaml` (最高优先级，用户自定义配置)
2. `config.yaml` (默认配置)
3. 智控台配置 (如果使用智控台)

### 关键配置项

```yaml
server:
  ip: 0.0.0.0
  port: 8000
  http_port: 8003
  websocket: ws://你的ip或者域名:端口号/xiaozhi/v1/
  vision_explain: http://你的ip或者域名:端口号/mcp/vision/explain
```

## 部署方式

### Docker 部署

```bash
# 最简化安装（仅核心服务）
docker-compose -f docker-compose.yml up -d

# 全模块安装（包含管理后台）
docker-compose -f docker-compose_all.yml up -d
```

### 源码部署

1. 按照 Python 服务、Java 后台、Web 界面的顺序分别启动
2. 确保数据库连接正常
3. 配置文件正确设置

## 插件开发

### 功能插件位置

`plugins_func/functions/` 目录包含现有功能插件：
- `change_role.py`: 角色切换
- `get_weather.py`: 天气查询
- `play_music.py`: 音乐播放
- `hass_*.py`: HomeAssistant 集成

### 插件开发规范

1. 继承基础插件类
2. 实现必要的接口方法
3. 在 `loadplugins.py` 中注册新插件

## 测试工具

### 音频交互测试

- 位置：`main/xiaozhi-server/test/test_page.html`
- 功能：测试音频播放和接收功能
- 使用：浏览器直接打开

### 性能测试

- 位置：`main/xiaozhi-server/performance_tester.py`
- 功能：测试 ASR、LLM、VLLM、TTS 响应速度
- 使用：`python performance_tester.py`

## 开发注意事项

1. **配置安全**: 敏感信息（API 密钥等）应放在 `data/.config.yaml` 中
2. **代码规范**: 遵循各语言的代码规范，使用提供的 lint 工具
3. **测试**: 新功能需要编写相应的测试用例
4. **文档**: 重要功能需要更新相关文档
5. **依赖管理**: 使用项目指定的包管理工具（pnpm、mvn、pip）

## 常见问题

1. **端口冲突**: 检查 8000、8003 等端口是否被占用
2. **依赖问题**: 确保使用正确的 Python 版本和依赖版本
3. **数据库连接**: 确保 MySQL 服务正常运行且连接配置正确
4. **API 密钥**: 各 AI 服务的 API 密钥需要正确配置

## 忽略的目录
不用关注此目录：xiaozhi-server\