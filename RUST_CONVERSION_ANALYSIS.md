# Claude Code Rust 移植分析文档

## 📋 项目概述

**源项目**: Claude Code (TypeScript/React)  
**目标**: 移植为 Rust，用于 Android APK 应用底层核心  
**架构模式**: Rust核心库 + JNI绑定 + Kotlin Android应用  
**版本**: 2.1.88  
**源代码规模**: 4,756 个文件，1,906 个核心源码文件  
**特性开关**: 50+ 个特性标志

---

## 🎯 核心架构设计

### 架构概览

```
┌─────────────────────────────────────────────────┐
│            Android App (Kotlin)                 │
│  ┌─────────────────────────────────────────┐   │
│  │    UI Layer (Jetpack Compose/XML)       │   │
│  │    - Chat界面                            │   │
│  │    - 工具执行结果展示                     │   │
│  │    - 会话管理界面                        │   │
│  └──────────────────┬──────────────────────┘   │
│                     │ JNI                       │
│  ┌──────────────────▼──────────────────────┐   │
│  │      claude_core (libclaude_core.so)     │   │
│  │  ┌──────────────────────────────────┐   │   │
│  │  │  JNI Bindings (jni/)             │   │   │
│  │  │  - SessionManager                │   │   │
│  │  │  - ToolExecutor                  │   │   │
│  │  │  - ApiClient                     │   │   │
│  │  │  - PromptBuilder                 │   │   │
│  │  └──────────────────────────────────┘   │   │
│  │  ┌──────────────────────────────────┐   │   │
│  │  │  Core Engine (core/)             │   │   │
│  │  │  - 会话管理                        │   │   │
│  │  │  - 工具执行引擎                    │   │   │
│  │  │  - Anthropic API客户端            │   │   │
│  │  │  - 提示词系统                      │   │   │
│  │  │  - 查询引擎                        │   │   │
│  │  └──────────────────────────────────┘   │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 关键设计原则

1. **纯库设计**: Rust部分编译为`.so`动态库，不包含CLI入口
2. **JNI优先**: 所有对外接口通过JNI暴露给Kotlin
3. **Android原生UI**: 不使用终端UI，由Kotlin/Jetpack Compose处理
4. **本地运行**: 所有逻辑在Android设备本地执行，仅API调用访问网络
5. **无远程依赖**: 不连接电脑或其他远程服务（除Anthropic API）

---

## 🛠️ 一、完整功能清单

### 1.1 需要移植的核心模块

| 模块 | 原始位置 | Rust实现 | JNI接口 | Android适配 |
|------|---------|---------|---------|------------|
| **会话管理** | `src/Task.ts`, `src/session/` | `core/session.rs` | `jni/session.rs` | ✅ SQLite存储 |
| **API客户端** | `src/services/api/` | `core/api.rs` | `jni/api.rs` | ✅ reqwest |
| **工具引擎** | `src/tools/` | `core/tools/` | `jni/tools.rs` | ⚠️ 需要适配 |
| **提示词系统** | `src/constants/prompts.ts` | `core/prompts.rs` | `jni/prompts.rs` | ✅ 纯逻辑 |
| **查询引擎** | `src/QueryEngine.ts` | `core/query.rs` | `jni/query.rs` | ✅ 核心逻辑 |
| **配置管理** | `src/utils/config.ts` | `core/config.rs` | `jni/config.rs` | ✅ JSON |
| **代理系统** | `src/tools/AgentTool/` | `core/agent.rs` | `jni/agent.rs` | ✅ 异步 |

### 1.2 需要重新设计的模块

| 模块 | 原始实现 | Android方案 | 说明 |
|------|---------|-------------|------|
| **CLI参数解析** | Commander.js | ❌ 移除 | 由Kotlin处理 |
| **终端UI** | Ink/React | ❌ 移除 | 用Jetpack Compose |
| **命令系统** | 斜杠命令 | ⚠️ 重新设计 | 作为Kotlin可调用的API |
| **BashTool** | Shell执行 | ⚠️ 重新设计 | Android受限shell |
| **Git操作** | simple-git | ⚠️ 受限 | Android Git库限制 |

### 1.3 不需要移植的模块

| 模块 | 原因 |
|------|------|
| **PowerShellTool** | Windows特定，Android不适用 |
| **REPLTool** | Android无终端环境 |
| **TerminalCaptureTool** | Android无终端 |
| **WebBrowserTool** | Android用WebView替代 |
| **桌面集成** (IDE, Chrome) | Android不适用 |
| **远程控制** (bridge) | 不连接电脑 |

---

## 📦 二、Rust库架构设计

### 2.1 项目结构

```
claude-code-rust/
├── Cargo.toml                  # 库配置（crate-type = ["cdylib"]）
├── src/
│   ├── lib.rs                  # 库入口，导出公共API
│   │
│   ├── jni/                    # JNI绑定层
│   │   ├── mod.rs              # JNI模块入口
│   │   ├── session.rs          # 会话管理JNI接口
│   │   │   - Java_com_claude_core_SessionManager_createSession
│   │   │   - Java_com_claude_core_SessionManager_sendMessage
│   │   │   - Java_com_claude_core_SessionManager_getHistory
│   │   │   - Java_com_claude_core_SessionManager_deleteSession
│   │   ├── tools.rs            # 工具执行JNI接口
│   │   │   - Java_com_claude_core_ToolExecutor_executeTool
│   │   │   - Java_com_claude_core_ToolExecutor_listTools
│   │   │   - Java_com_claude_core_ToolExecutor_getToolSchema
│   │   ├── api.rs              # API客户端JNI接口
│   │   │   - Java_com_claude_core_ApiClient_initialize
│   │   │   - Java_com_claude_core_ApiClient_setApiKey
│   │   │   - Java_com_claude_core_ApiClient_getModels
│   │   └── callback.rs         # 异步回调处理
│   │       - Rust→Kotlin回调机制
│   │       - 流式响应推送
│   │
│   ├── core/                   # 核心引擎（纯Rust逻辑）
│   │   ├── mod.rs
│   │   ├── session.rs          # 会话管理
│   │   │   - Session结构体
│   │   │   - 会话生命周期管理
│   │   │   - 对话历史存储
│   │   ├── query.rs            # 查询引擎
│   │   │   - 查询处理流程
│   │   │   - 流式响应处理
│   │   │   - 工具调用循环
│   │   ├── tool_manager.rs     # 工具管理器
│   │   │   - 工具注册表
│   │   │   - 工具执行框架
│   │   │   - 权限检查
│   │   └── context.rs          # 上下文管理
│   │       - 上下文窗口管理
│   │       - Token计数
│   │
│   ├── api/                    # API客户端
│   │   ├── mod.rs
│   │   ├── client.rs           # HTTP客户端
│   │   │   - reqwest异步客户端
│   │   │   - 连接池管理
│   │   ├── auth.rs             # 认证管理
│   │   │   - API Key管理
│   │   │   - OAuth支持
│   │   └── streaming.rs        # 流式处理
│   │       - SSE解析
│   │       - 事件流处理
│   │
│   ├── tools/                  # 工具实现（Android适配版）
│   │   ├── mod.rs
│   │   ├── trait.rs            # Tool trait定义
│   │   ├── file_read.rs        # 文件读取（Android存储）
│   │   ├── file_write.rs       # 文件写入（Android存储）
│   │   ├── web_fetch.rs        # HTTP请求
│   │   ├── web_search.rs       # Web搜索
│   │   ├── todo_write.rs       # 任务管理
│   │   ├── agent.rs            # 代理工具
│   │   └── android/            # Android特定工具
│   │       ├── shell.rs        # 受限shell（Android）
│   │       └── clipboard.rs    # 剪贴板访问
│   │
│   ├── prompts/                # 提示词系统
│   │   ├── mod.rs
│   │   ├── builder.rs          # 提示词构建器
│   │   ├── sections.rs         # 提示词部分
│   │   └── dynamic.rs          # 动态提示词
│   │
│   ├── config/                 # 配置系统
│   │   ├── mod.rs
│   │   ├── settings.rs         # 设置管理
│   │   └── storage.rs          # 配置存储（SQLite）
│   │
│   └── utils/                  # 工具函数
│       ├── mod.rs
│       ├── crypto.rs           # 加密工具
│       └── json.rs             # JSON处理
│
└── android/                    # Android构建配置
    ├── build.gradle            # Gradle构建脚本
    └── jni/                    # JNI头文件生成
        └── Android.mk
```

### 2.2 JNI接口设计

#### Kotlin端接口定义

```kotlin
// SessionManager.kt
package com.claude.core

class SessionManager {
    companion object {
        init {
            System.loadLibrary("claude_core")
        }
    }
    
    // JNI方法声明
    private external fun nativeCreateSession(config: String): Long
    private external fun nativeSendMessage(sessionId: Long, message: String, callback: SessionCallback)
    private external fun nativeGetHistory(sessionId: Long): String
    private external fun nativeDeleteSession(sessionId: Long)
    
    // 公共API
    fun createSession(config: SessionConfig): Long {
        return nativeCreateSession(Json.encodeToString(config))
    }
    
    fun sendMessage(sessionId: Long, message: String, callback: SessionCallback) {
        nativeSendMessage(sessionId, message, callback)
    }
}

// 回调接口
interface SessionCallback {
    fun onToken(token: String)
    fun onToolCall(toolName: String, input: String)
    fun onToolResult(toolName: String, result: String)
    fun onComplete(response: String)
    fun onError(error: String)
}
```

#### Rust端JNI实现

```rust
// src/jni/session.rs
use jni::JNIEnv;
use jni::objects::{JClass, JString};
use jni::sys::{jlong, jobject};

#[no_mangle]
pub extern "C" fn Java_com_claude_core_SessionManager_nativeCreateSession(
    env: JNIEnv,
    _class: JClass,
    config: JString,
) -> jlong {
    let config_str: String = env.get_string(config).unwrap().into();
    let session_config: SessionConfig = serde_json::from_str(&config_str).unwrap();
    
    let session = Session::new(session_config);
    Box::into_raw(Box::new(session)) as jlong
}

#[no_mangle]
pub extern "C" fn Java_com_claude_core_SessionManager_nativeSendMessage(
    env: JNIEnv,
    _class: JClass,
    session_id: jlong,
    message: JString,
    callback: jobject,
) {
    let session = unsafe { &mut *(session_id as *mut Session) };
    let msg: String = env.get_string(message).unwrap().into();
    
    // 异步执行，通过回调返回结果
    tokio::spawn(async move {
        match session.send_message(&msg).await {
            Ok(stream) => {
                // 处理流式响应
                for await token in stream {
                    // 通过JNI回调Kotlin
                    call_kotlin_callback(env, callback, "onToken", token);
                }
            }
            Err(e) => {
                call_kotlin_callback(env, callback, "onError", e.to_string());
            }
        }
    });
}
```

### 2.3 核心依赖配置

```toml
[package]
name = "claude-core"
version = "0.1.0"
edition = "2021"

[lib]
name = "claude_core"
crate-type = ["cdylib"]  # 编译为动态库供JNI使用

[dependencies]
# JNI绑定
jni = "0.21"

# 异步运行时
tokio = { version = "1.0", features = ["full"] }

# HTTP客户端（支持Android）
reqwest = { version = "0.11", features = ["json", "stream", "rustls-tls"] }

# JSON处理
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# 数据库（Android支持）
rusqlite = { version = "0.31", features = ["bundled"] }

# 错误处理
anyhow = "1.0"
thiserror = "1.0"

# 异步流
futures = "0.3"
async-stream = "0.3"

# 日志
tracing = "0.1"

# 正则表达式
regex = "1.9"

# 时间处理
chrono = { version = "0.4", features = ["serde"] }

# 加密
rustls = "0.21"
jsonwebtoken = "9.0"

# 文件处理
walkdir = "2.4"

# Android日志
android_logger = "0.13"
log = "0.4"
```

---

## 🔧 三、模块详细分析

### 3.1 会话管理模块

**原始实现**: `src/Task.ts`, `src/session/`

**Rust实现**: `core/session.rs`

```rust
// 核心结构
pub struct Session {
    id: String,
    config: SessionConfig,
    history: Vec<Message>,
    context: Context,
    tool_manager: ToolManager,
}

impl Session {
    pub fn new(config: SessionConfig) -> Self { ... }
    
    pub async fn send_message(&mut self, message: &str) -> Result<MessageStream> {
        // 1. 添加用户消息到历史
        // 2. 构建提示词
        // 3. 调用API
        // 4. 处理工具调用
        // 5. 返回流式响应
    }
    
    pub fn get_history(&self) -> &[Message] { ... }
    
    pub fn save_to_db(&self, db: &Connection) -> Result<()> { ... }
    
    pub fn load_from_db(db: &Connection, id: &str) -> Result<Self> { ... }
}
```

**JNI接口**:
- `createSession(config: String): Long`
- `sendMessage(sessionId: Long, message: String, callback: SessionCallback)`
- `getHistory(sessionId: Long): String`
- `deleteSession(sessionId: Long)`

**Android适配**:
- ✅ 使用SQLite存储会话历史
- ✅ 支持会话恢复
- ✅ 异步操作通过回调返回

### 3.2 API客户端模块

**原始实现**: `src/services/api/`

**Rust实现**: `api/client.rs`

```rust
pub struct ApiClient {
    client: reqwest::Client,
    api_key: String,
    base_url: String,
}

impl ApiClient {
    pub fn new(api_key: String) -> Self {
        Self {
            client: reqwest::Client::builder()
                .timeout(Duration::from_secs(30))
                .build()
                .unwrap(),
            api_key,
            base_url: "https://api.anthropic.com".to_string(),
        }
    }
    
    pub async fn send_message(
        &self,
        messages: &[Message],
        tools: &[Tool],
        model: &str,
    ) -> Result<MessageStream> {
        // 构建请求
        // 发送流式请求
        // 返回SSE事件流
    }
}
```

**JNI接口**:
- `initialize(apiKey: String): Boolean`
- `setApiKey(apiKey: String)`
- `getModels(): String`

**Android适配**:
- ✅ 使用reqwest（支持Android）
- ✅ 流式响应处理
- ✅ 错误重试机制

### 3.3 工具引擎模块

**原始实现**: `src/tools/`

**Rust实现**: `core/tool_manager.rs` + `tools/`

```rust
// 工具trait
pub trait Tool {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn schema(&self) -> ToolSchema;
    async fn execute(&self, input: ToolInput) -> Result<ToolOutput>;
}

// 工具管理器
pub struct ToolManager {
    tools: HashMap<String, Box<dyn Tool>>,
}

impl ToolManager {
    pub fn new() -> Self {
        let mut manager = Self { tools: HashMap::new() };
        
        // 注册Android适用的工具
        manager.register(Box::new(FileReadTool::new()));
        manager.register(Box::new(FileWriteTool::new()));
        manager.register(Box::new(WebFetchTool::new()));
        manager.register(Box::new(WebSearchTool::new()));
        manager.register(Box::new(TodoWriteTool::new()));
        manager.register(Box::new(AgentTool::new()));
        
        manager
    }
    
    pub async fn execute(&self, name: &str, input: ToolInput) -> Result<ToolOutput> {
        let tool = self.tools.get(name)
            .ok_or_else(|| anyhow::anyhow!("Tool not found: {}", name))?;
        tool.execute(input).await
    }
}
```

**JNI接口**:
- `executeTool(name: String, input: String, callback: ToolCallback): String`
- `listTools(): String`
- `getToolSchema(name: String): String`

**Android适配**:
- ⚠️ BashTool需要重新设计为受限shell
- ✅ 文件操作使用Android存储API
- ✅ Web工具正常使用

### 3.4 提示词系统模块

**原始实现**: `src/constants/prompts.ts`

**Rust实现**: `prompts/builder.rs`

```rust
pub struct PromptBuilder {
    sections: Vec<PromptSection>,
}

impl PromptBuilder {
    pub fn new() -> Self { ... }
    
    pub fn build_system_prompt(
        &self,
        tools: &[Tool],
        context: &Context,
    ) -> String {
        let mut prompt = String::new();
        
        prompt.push_str(&self.intro_section());
        prompt.push_str(&self.system_section());
        prompt.push_str(&self.doing_tasks_section());
        prompt.push_str(&self.actions_section());
        prompt.push_str(&self.tools_section(tools));
        prompt.push_str(&self.tone_section());
        
        // 动态部分
        prompt.push_str(&self.env_info_section(context));
        prompt.push_str(&self.memory_section(context));
        
        prompt
    }
}
```

**JNI接口**:
- `buildSystemPrompt(tools: String, context: String): String`

**Android适配**:
- ✅ 纯逻辑，无需特殊适配
- ✅ 环境信息适配Android

### 3.5 查询引擎模块

**原始实现**: `src/QueryEngine.ts`

**Rust实现**: `core/query.rs`

```rust
pub struct QueryEngine {
    api_client: ApiClient,
    tool_manager: ToolManager,
    prompt_builder: PromptBuilder,
}

impl QueryEngine {
    pub async fn execute_query(
        &self,
        session: &mut Session,
        message: &str,
    ) -> Result<MessageStream> {
        // 1. 构建消息列表
        let messages = session.build_messages(message);
        
        // 2. 构建系统提示词
        let system_prompt = self.prompt_builder.build_system_prompt(
            &self.tool_manager.list_tools(),
            &session.context(),
        );
        
        // 3. 调用API
        let stream = self.api_client.send_message(
            &messages,
            &self.tool_manager.list_tools(),
            &session.model(),
        ).await?;
        
        // 4. 处理工具调用
        Ok(self.process_stream(stream, session).await)
    }
}
```

**JNI接口**:
- `executeQuery(sessionId: Long, message: String, callback: QueryCallback)`

**Android适配**:
- ✅ 核心逻辑，无需特殊适配
- ✅ 流式响应通过JNI回调

---

## 📱 四、Android集成方案

### 4.1 Kotlin端架构

```
app/src/main/java/com/claude/app/
├── MainActivity.kt              # 主Activity
├── ClaudeApplication.kt         # Application类
│
├── core/
│   ├── ClaudeCore.kt           # Rust核心库封装
│   ├── SessionManager.kt       # 会话管理
│   ├── ToolExecutor.kt         # 工具执行
│   └── ApiClient.kt            # API客户端
│
├── ui/
│   ├── chat/
│   │   ├── ChatScreen.kt       # 聊天界面
│   │   ├── ChatViewModel.kt    # 聊天ViewModel
│   │   └── MessageList.kt      # 消息列表
│   ├── tools/
│   │   ├── ToolResultView.kt   # 工具结果展示
│   │   └── ToolPermission.kt   # 工具权限对话框
│   └── settings/
│       └── SettingsScreen.kt   # 设置界面
│
└── data/
    ├── database/
    │   └── AppDatabase.kt      # Room数据库
    └── repository/
        └── SessionRepository.kt # 会话仓库
```

### 4.2 JNI调用示例

```kotlin
// ClaudeCore.kt
package com.claude.app.core

import com.claude.core.SessionManager
import com.claude.core.ApiClient
import com.claude.core.ToolExecutor

class ClaudeCore(private val context: Context) {
    private val sessionManager = SessionManager()
    private val apiClient = ApiClient()
    private val toolExecutor = ToolExecutor()
    
    suspend fun initialize(apiKey: String): Boolean {
        return withContext(Dispatchers.IO) {
            apiClient.initialize(apiKey)
        }
    }
    
    suspend fun sendMessage(
        sessionId: Long,
        message: String,
        onToken: (String) -> Unit,
        onToolCall: (String, String) -> Unit,
        onComplete: (String) -> Unit,
        onError: (String) -> Unit
    ) {
        val callback = object : SessionCallback {
            override fun onToken(token: String) = onToken(token)
            override fun onToolCall(toolName: String, input: String) = onToolCall(toolName, input)
            override fun onToolResult(toolName: String, result: String) { }
            override fun onComplete(response: String) = onComplete(response)
            override fun onError(error: String) = onError(error)
        }
        
        sessionManager.sendMessage(sessionId, message, callback)
    }
}
```

### 4.3 Gradle配置

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        ndk {
            abiFilters += listOf("arm64-v8a", "armeabi-v7a")
        }
    }
    
    externalNativeBuild {
        cmake {
            path = file("../../claude-code-rust/android/CMakeLists.txt")
        }
    }
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0")
    implementation("androidx.compose.ui:ui:1.5.0")
    implementation("androidx.room:room-runtime:2.6.0")
}
```

---

## ⚠️ 五、关键挑战和解决方案

### 5.1 Rust-Android集成

| 挑战 | 解决方案 |
|------|---------|
| **交叉编译** | 使用`cross`工具或NDK工具链 |
| **JNI类型转换** | 使用serde序列化复杂类型 |
| **异步回调** | tokio + JNI全局引用 |
| **内存管理** | Rust所有权 + JNI局部引用 |
| **日志集成** | android_logger + log crate |

### 5.2 Android平台限制

| 挑战 | 解决方案 |
|------|---------|
| **文件系统访问** | Storage Access Framework |
| **进程执行** | 受限shell或移除BashTool |
| **网络权限** | Android网络安全配置 |
| **存储权限** | 运行时权限请求 |
| **后台执行** | 前台服务 + 通知 |
| **电池优化** | WorkManager调度 |

### 5.3 性能优化

| 优化点 | 策略 |
|--------|------|
| **内存使用** | 对象池、流式处理 |
| **网络优化** | 连接池、请求合并 |
| **数据库** | SQLite WAL模式 |
| **异步处理** | tokio多线程运行时 |
| **UI响应** | 协程 + 主线程调度 |

---

## 🧪 六、测试策略

### 6.1 Rust端测试

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_session_creation() {
        let config = SessionConfig::default();
        let session = Session::new(config);
        assert!(!session.id().is_empty());
    }
    
    #[tokio::test]
    async fn test_api_client() {
        let client = ApiClient::new("test-key".to_string());
        // 测试API调用
    }
}
```

### 6.2 JNI集成测试

```kotlin
@RunWith(AndroidJUnit4::class)
class SessionManagerTest {
    @Test
    fun testCreateSession() {
        val manager = SessionManager()
        val config = SessionConfig(model = "claude-sonnet-4-20250514")
        val sessionId = manager.createSession(config)
        assertTrue(sessionId > 0)
    }
}
```

### 6.3 Android端测试

- 单元测试: JUnit + Mockito
- UI测试: Espresso / Compose Testing
- 集成测试: AndroidJUnit4
- 性能测试: Android Benchmark

---

## 📚 七、参考资源

### Rust Android开发
- [jni crate文档](https://docs.rs/jni)
- [Android NDK Rust](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-06-rust-on-android.html)
- [cargo-ndk](https://github.com/bbqsrc/cargo-ndk)

### 相关项目
- [tiktoken-rs](https://github.com/DevBlockhouse/tiktoken-rs) - Rust token计数
- [reqwest](https://docs.rs/reqwest) - HTTP客户端（支持Android）
- [rusqlite](https://docs.rs/rusqlite) - SQLite绑定（支持Android）

---

**文档版本**: 2.0  
**最后更新**: 2026-04-01  
**架构变更**: 从CLI工具改为Android库 + JNI绑定