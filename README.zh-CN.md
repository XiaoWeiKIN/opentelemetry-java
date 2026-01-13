# OpenTelemetry Java 技术文档

> **注意**: 这是 OpenTelemetry Java 项目的中文技术文档，详细说明了项目的架构设计和工程设计。
> **For English documentation**, see [README.md](README.md)

## 📑 目录

- [概述](#概述)
- [🚀 快速开始](#-快速开始)

---

### 第一部分：项目概览

- [1. 概述和快速开始](#1-概述和快速开始)
- [2. 架构总览](#2-架构总览)
- [3. 目录结构和模块索引](#3-目录结构和模块索引)

### 第二部分：核心架构

- [4. API 层设计](#4-api-层设计)
- [5. SDK 层设计](#5-sdk-层设计)
- [6. Context 传播机制](#6-context-传播机制)
- [7. Traces（分布式跟踪）](#7-traces分布式跟踪)
- [8. Metrics（指标收集）](#8-metrics指标收集)
- [9. Logs（日志记录）](#9-logs日志记录)
- [10. 导出器架构](#10-导出器架构)

### 第三部分：工程设计

- [11. 构建系统（buildSrc）](#11-构建系统buildsrc)
- [12. 代码质量保证](#12-代码质量保证)
- [13. Android 兼容性](#13-android-兼容性)
- [14. API 兼容性保证](#14-api-兼容性保证)
- [15. BOM 版本管理](#15-bom-版本管理)
- [16. 自动配置机制](#16-自动配置机制)
- [17. 扩展和插件机制](#17-扩展和插件机制)
- [18. 性能优化](#18-性能优化)

### 第四部分：核心组件

- [19. Context 模块详解](#19-context-模块详解)
- [20. Semconv（语义约定）](#20-semconv语义约定)
- [21. 导出器详解](#21-导出器详解)
- [22. 扩展模块](#22-扩展模块)
- [23. 集成和兼容层](#23-集成和兼容层)

### 第五部分：使用指南

- [24. 快速开始](#24-快速开始)
- [25. 常见使用场景](#25-常见使用场景)
- [26. 高级配置](#26-高级配置)
- [27. 故障排查](#27-故障排查)

### 第六部分：开发指南

- [28. 贡献指南](#28-贡献指南)
- [29. 测试指南](#29-测试指南)
- [30. 发布流程](#30-发布流程)

### 第七部分：附录

- [31. 附录](#31-附录)

---

## 概述

**OpenTelemetry Java** 是 OpenTelemetry 项目的 Java 实现，提供了完整的可观测性框架，用于收集、处理和导出应用程序的遥测数据（Traces、Metrics、Logs）。

**项目统计**:
- **版本**: 1.58.0-SNAPSHOT
- **模块数量**: 47 个
- **架构模式**: API/SDK 分离
- **支持的 Java 版本**: Java 8+（编译使用 Java 21）
- **Android 支持**: API 23+ (Android 6.0+)
- **GitHub**: [open-telemetry/opentelemetry-java](https://github.com/open-telemetry/opentelemetry-java)

**核心特性**:
- ✅ **API/SDK 分离架构**: 稳定的 API 层 + 可插拔的 SDK 实现
- ✅ **三大信号支持**: Traces（分布式跟踪）、Metrics（指标收集）、Logs（日志记录）
- ✅ **Context 传播**: 跨线程、跨进程的上下文传播机制
- ✅ **SPI 扩展架构**: 可插拔的导出器、采样器、资源检测器
- ✅ **自动配置**: 基于环境变量的零代码配置
- ✅ **Builder 模式**: 流式 API 构建 SDK 组件
- ✅ **多导出器支持**: OTLP、Zipkin、Prometheus、Jaeger 等
- ✅ **兼容层**: OpenTracing、OpenCensus、Micrometer 兼容
- ✅ **Android 兼容**: 支持 Android API 23+，包含 Desugar 库支持
- ✅ **BOM 版本管理**: 统一管理 47 个模块的版本

---

## 🚀 快速开始

### 快速参考卡片

| 使用场景 | 推荐模块 | 章节链接 |
|---------|---------|---------|
| 快速开始（推荐） | `opentelemetry-bom` + `opentelemetry-api` + `opentelemetry-sdk` | [24. 快速开始](#24-快速开始) |
| 分布式跟踪 | `opentelemetry-api` + `opentelemetry-sdk-trace` | [7. Traces](#7-traces分布式跟踪) |
| 指标收集 | `opentelemetry-api` + `opentelemetry-sdk-metrics` | [8. Metrics](#8-metrics指标收集) |
| 日志记录 | `opentelemetry-api-logs` + `opentelemetry-sdk-logs` | [9. Logs](#9-logs日志记录) |
| 导出到 OTLP | `opentelemetry-exporter-otlp` | [21. 导出器详解](#21-导出器详解) |
| 自动配置 | `opentelemetry-sdk-extension-autoconfigure` | [16. 自动配置机制](#16-自动配置机制) |
| Spring Boot 集成 | `opentelemetry-sdk` + Spring Boot 自动配置 | [25. 常见使用场景](#25-常见使用场景) |
| Android 应用 | `opentelemetry-sdk` + Animal Sniffer | [13. Android 兼容性](#13-android-兼容性) |

### 常用模块索引

**API 模块** (4个):
- `api:all` - 聚合所有稳定 API
- `api:incubator` - 孵化中的实验性 API
- `api:events` - 事件 API
- `api:logs` - 日志 API

**SDK 核心** (8个):
- `sdk:all` - 聚合所有 SDK
- `sdk:common` - 通用 SDK 组件
- `sdk:trace` - 分布式跟踪 SDK
- `sdk:metrics` - 指标收集 SDK
- `sdk:logs` - 日志 SDK
- `sdk:testing` - SDK 测试工具
- `sdk-extensions:autoconfigure` - 自动配置
- `sdk-extensions:autoconfigure-spi` - 自动配置 SPI

**导出器** (7个):
- `exporters:otlp:all` - OTLP 协议导出器
- `exporters:zipkin` - Zipkin 导出器
- `exporters:prometheus` - Prometheus 导出器
- `exporters:logging` - 日志导出器
- 更多见 [3. 目录结构和模块索引](#3-目录结构和模块索引)

---

# 第一部分：项目概览

## 1. 概述和快速开始

### 1.1 项目简介

**OpenTelemetry** 是 CNCF (Cloud Native Computing Foundation) 的孵化项目，旨在为云原生软件提供统一的可观测性标准。OpenTelemetry Java 是其 Java 语言实现，提供了完整的 API 和 SDK，用于收集、处理和导出应用程序的遥测数据。

**解决的问题**:
- **分布式系统可观测性**: 跨服务追踪请求链路，快速定位性能瓶颈
- **标准化**: 统一的 API 避免供应商锁定
- **零代码侵入（可选）**: 通过 Java Agent 自动注入，无需修改代码
- **多后端支持**: 同时导出到 Jaeger、Prometheus、Zipkin 等多个后端

**核心概念**:
```
┌──────────────────────────────────────────────────────┐
│              User Application                        │
│         (使用 OpenTelemetry API)                      │
└────────────────────┬─────────────────────────────────┘
                     │ 依赖
┌────────────────────▼─────────────────────────────────┐
│           OpenTelemetry API (稳定)                   │
│   ┌──────────┬──────────┬──────────┬──────────┐    │
│   │ Tracer   │ Meter    │ Logger   │ Context  │    │
│   │ API      │ API      │ API      │ API      │    │
│   └──────────┴──────────┴──────────┴──────────┘    │
└────────────────────┬─────────────────────────────────┘
                     │ 实现
┌────────────────────▼─────────────────────────────────┐
│         OpenTelemetry SDK (可插拔)                   │
│   ┌──────────┬──────────┬──────────┬──────────┐    │
│   │ Tracer   │ Meter    │ Logger   │ Context  │    │
│   │ Provider │ Provider │ Provider │ Storage  │    │
│   └─────┬────┴────┬─────┴─────┬────┴────┬─────┘    │
│         │         │           │         │          │
│     ┌───▼───┐ ┌───▼───┐  ┌───▼───┐ ┌───▼───┐     │
│     │Sampler│ │Aggre- │  │Log    │ │Thread │     │
│     │       │ │gator  │  │Record │ │Local  │     │
│     └───┬───┘ └───┬───┘  │Proc   │ └───────┘     │
│         │         │       └───┬───┘               │
│     ┌───▼─────────▼───────────▼───┐               │
│     │   Span/Metric/Log Processor │               │
│     └───┬───────────────────────┬──┘               │
│         │                       │                  │
└─────────┼───────────────────────┼──────────────────┘
          │ 导出                  │ 导出
          ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  OTLP Exporter   │    │ Other Exporters  │
│  (gRPC/HTTP)     │    │ (Zipkin/Prom)    │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         ▼                       ▼
┌────────────────────────────────────────────┐
│   Backend (Jaeger/Prometheus/etc.)         │
└────────────────────────────────────────────┘
```

### 1.2 核心特性详解

#### 1.2.1 API/SDK 分离架构

**设计理念**:
- **API 层**: 提供稳定的接口，保证向后兼容（语义化版本控制）
- **SDK 层**: 可插拔的实现，支持自定义和扩展

**优势**:
```java
// 用户代码只依赖稳定的 API
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;

public class MyService {
    private final Tracer tracer;

    public MyService(OpenTelemetry openTelemetry) {
        // API 接口稳定，升级 SDK 版本不影响用户代码
        this.tracer = openTelemetry.getTracer("my-service", "1.0.0");
    }
}
```

**分离的价值**:
- ✅ **稳定性**: API 升级遵循语义化版本，破坏性变更最小化
- ✅ **灵活性**: 可以替换 SDK 实现而不修改业务代码
- ✅ **测试性**: 使用 No-op 实现进行单元测试
- ✅ **渐进式采用**: 先引入 API，后续再配置 SDK

#### 1.2.2 三大信号（Signals）

OpenTelemetry 支持三种遥测信号：

**1. Traces（分布式跟踪）**:
```
用户请求 → 服务 A → 服务 B → 数据库
   ↓         ↓         ↓         ↓
 Span 1   Span 2    Span 3    Span 4
   └────────┴─────────┴─────────┘
         Trace (完整的请求链路)
```

**2. Metrics（指标收集）**:
```java
// Counter: 单调递增计数器
requestCounter.add(1);

// Histogram: 分布式统计
responseTimeHistogram.record(123);

// Gauge: 当前值观测
memoryGauge.set(1024);
```

**3. Logs（日志记录）**:
```java
// 结构化日志，自动关联当前 Span
logger.info("Processing request",
    Map.of("userId", "12345", "action", "purchase"));
```

**三大信号的协同**:
```
Trace (traceId=abc123)
├── Span 1 (spanId=001)
│   ├── Metrics: request.count=1, request.duration=50ms
│   └── Logs: [INFO] Request started, userId=12345
├── Span 2 (spanId=002)
│   ├── Metrics: db.query.duration=20ms
│   └── Logs: [DEBUG] Query executed, rows=10
└── Span 3 (spanId=003)
    └── Logs: [INFO] Request completed
```

#### 1.2.3 Context 传播

**同进程传播（ThreadLocal）**:
```java
Span span = tracer.spanBuilder("operation").startSpan();
try (Scope scope = span.makeCurrent()) {
    // 当前线程的所有操作都在此 Span 下
    callAnotherMethod();  // 自动继承 Span
} finally {
    span.end();
}
```

**跨进程传播（HTTP Headers）**:
```
客户端                           服务端
┌─────────┐                     ┌─────────┐
│ Span A  │  HTTP Request       │ Span B  │
│ traceId │  ─────────────────> │ traceId │
│ spanId  │  Headers:           │ spanId  │
│         │  traceparent: ...   │ parent  │
└─────────┘                     └─────────┘
```

#### 1.2.4 SPI 扩展架构

通过 Java Service Provider Interface (SPI) 实现可插拔：

```
SDK 扩展点
├── Sampler SPI
│   ├── AlwaysOn
│   ├── AlwaysOff
│   ├── TraceIdRatioBased
│   └── ParentBased
├── Exporter SPI
│   ├── OTLP (gRPC/HTTP)
│   ├── Zipkin
│   ├── Prometheus
│   └── Logging
├── Resource Detector SPI
│   ├── OS
│   ├── Process
│   ├── Container
│   └── Host
└── Context Storage SPI
    ├── ThreadLocal (默认)
    ├── StormContext
    └── ContextDB (自定义)
```

### 1.3 快速开始

#### 1.3.1 添加依赖

**使用 BOM 管理版本（推荐）**:

```kotlin
// build.gradle.kts
dependencies {
    // 1. 导入 BOM，统一管理 OpenTelemetry 版本
    implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))

    // 2. 添加 API 和 SDK（不需要指定版本）
    implementation("io.opentelemetry:opentelemetry-api")
    implementation("io.opentelemetry:opentelemetry-sdk")

    // 3. 添加导出器
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")

    // 4. （可选）自动配置
    implementation("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure")
}
```

**Maven 配置**:
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.opentelemetry</groupId>
      <artifactId>opentelemetry-bom</artifactId>
      <version>1.35.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
  </dependency>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
  </dependency>
</dependencies>
```

#### 1.3.2 初始化 SDK

**手动配置（完全控制）**:

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;

public class Application {
    public static void main(String[] args) {
        // 1. 配置导出器（OTLP gRPC）
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://localhost:4317")
            .build();

        // 2. 创建 TracerProvider
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
            .build();

        // 3. 初始化 OpenTelemetry 并注册为全局实例
        OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();

        // 4. 应用程序逻辑
        runApplication(openTelemetry);

        // 5. 关闭时清理资源
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            tracerProvider.close();
        }));
    }
}
```

**自动配置（零代码）**:

```java
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;

public class Application {
    public static void main(String[] args) {
        // 自动从环境变量配置
        OpenTelemetry openTelemetry = AutoConfiguredOpenTelemetrySdk.initialize()
            .getOpenTelemetrySdk();

        runApplication(openTelemetry);
    }
}
```

**环境变量配置**:
```bash
# 服务名称
export OTEL_SERVICE_NAME=my-service

# OTLP 导出器端点
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 采样策略（10% 采样）
export OTEL_TRACES_SAMPLER=traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1

# 资源属性
export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.version=1.0.0
```

#### 1.3.3 创建第一个 Trace

```java
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.StatusCode;
import io.opentelemetry.context.Scope;

public class MyService {
    private final Tracer tracer;

    public MyService(OpenTelemetry openTelemetry) {
        // 获取 Tracer（需指定 instrumentation scope）
        this.tracer = openTelemetry.getTracer("my-service", "1.0.0");
    }

    public void processRequest(String userId) {
        // 1. 创建 Span
        Span span = tracer.spanBuilder("processRequest")
            .setSpanKind(SpanKind.INTERNAL)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // 2. 添加属性
            span.setAttribute("user.id", userId);
            span.setAttribute("http.method", "POST");

            // 3. 添加事件
            span.addEvent("Processing started");

            // 4. 业务逻辑
            doWork();

            // 5. 标记成功
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            // 6. 记录异常
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "Request failed");
            throw e;
        } finally {
            // 7. 结束 Span（自动记录结束时间）
            span.end();
        }
    }

    private void doWork() {
        // 创建子 Span（自动继承父 Span）
        Span childSpan = tracer.spanBuilder("doWork")
            .setParent(Context.current())
            .startSpan();

        try (Scope scope = childSpan.makeCurrent()) {
            // 子 Span 的业务逻辑
            Thread.sleep(100);
        } catch (InterruptedException e) {
            childSpan.recordException(e);
        } finally {
            childSpan.end();
        }
    }
}
```

**生成的 Trace 结构**:
```
Trace (traceId=abc123def456)
└── Span: processRequest (spanId=001, duration=150ms)
    ├── Attribute: user.id=12345
    ├── Attribute: http.method=POST
    ├── Event: Processing started
    └── Child Span: doWork (spanId=002, duration=100ms)
```

#### 1.3.4 创建第一个 Metric

```java
import io.opentelemetry.api.metrics.Meter;
import io.opentelemetry.api.metrics.LongCounter;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;

public class MetricsExample {
    private final LongCounter requestCounter;
    private final DoubleHistogram responseTimeHistogram;

    public MetricsExample(OpenTelemetry openTelemetry) {
        Meter meter = openTelemetry.getMeter("my-service", "1.0.0");

        // 1. Counter（计数器）- 单调递增
        this.requestCounter = meter.counterBuilder("requests")
            .setDescription("Total number of requests")
            .setUnit("1")
            .build();

        // 2. Histogram（直方图）- 分布统计
        this.responseTimeHistogram = meter.histogramBuilder("response.time")
            .setDescription("Response time distribution")
            .setUnit("ms")
            .build();
    }

    public void handleRequest(String endpoint, String method) {
        long startTime = System.currentTimeMillis();

        try {
            // 业务逻辑
            processRequest();

            // 记录成功请求
            requestCounter.add(1,
                Attributes.of(
                    AttributeKey.stringKey("endpoint"), endpoint,
                    AttributeKey.stringKey("method"), method,
                    AttributeKey.stringKey("status"), "success"
                ));
        } catch (Exception e) {
            // 记录失败请求
            requestCounter.add(1,
                Attributes.of(
                    AttributeKey.stringKey("endpoint"), endpoint,
                    AttributeKey.stringKey("method"), method,
                    AttributeKey.stringKey("status"), "error"
                ));
        } finally {
            // 记录响应时间
            long duration = System.currentTimeMillis() - startTime;
            responseTimeHistogram.record(duration,
                Attributes.of(
                    AttributeKey.stringKey("endpoint"), endpoint
                ));
        }
    }
}
```

**生成的 Metrics**:
```
# Counter
requests{endpoint="/api/users",method="GET",status="success"} 100
requests{endpoint="/api/users",method="POST",status="error"} 5

# Histogram
response.time_bucket{endpoint="/api/users",le="10"} 50
response.time_bucket{endpoint="/api/users",le="50"} 80
response.time_bucket{endpoint="/api/users",le="100"} 95
response.time_bucket{endpoint="/api/users",le="+Inf"} 100
response.time_sum{endpoint="/api/users"} 3500
response.time_count{endpoint="/api/users"} 100
```

#### 1.3.5 Context 传播示例

**跨线程传播**:

```java
import io.opentelemetry.context.Context;
import java.util.concurrent.CompletableFuture;

public class AsyncExample {
    private final Tracer tracer;

    public void asyncOperation() {
        Span span = tracer.spanBuilder("asyncOp").startSpan();

        try (Scope scope = span.makeCurrent()) {
            // 1. 捕获当前 Context
            Context context = Context.current();

            // 2. 异步操作中传播 Context
            CompletableFuture.runAsync(() -> {
                try (Scope asyncScope = context.makeCurrent()) {
                    // Context 已传播，可以访问父 Span
                    Span currentSpan = Span.current();
                    currentSpan.addEvent("Async operation started");

                    // 创建子 Span
                    Span childSpan = tracer.spanBuilder("asyncWork")
                        .startSpan();
                    try {
                        // 异步业务逻辑
                    } finally {
                        childSpan.end();
                    }
                }
            });
        } finally {
            span.end();
        }
    }
}
```

**跨进程传播（HTTP）**:

```java
import io.opentelemetry.context.propagation.TextMapGetter;
import io.opentelemetry.context.propagation.TextMapSetter;

// 客户端：注入 Context 到 HTTP Headers
public void sendHttpRequest() {
    HttpURLConnection connection = ...;

    // 注入当前 Context
    openTelemetry.getPropagators().getTextMapPropagator()
        .inject(Context.current(), connection, new TextMapSetter<HttpURLConnection>() {
            @Override
            public void set(HttpURLConnection carrier, String key, String value) {
                carrier.setRequestProperty(key, value);
            }
        });

    connection.connect();
}

// 服务端：从 HTTP Headers 提取 Context
public void handleHttpRequest(HttpServletRequest request) {
    // 提取 Context
    Context extractedContext = openTelemetry.getPropagators().getTextMapPropagator()
        .extract(Context.current(), request, new TextMapGetter<HttpServletRequest>() {
            @Override
            public Iterable<String> keys(HttpServletRequest carrier) {
                return Collections.list(carrier.getHeaderNames());
            }

            @Override
            public String get(HttpServletRequest carrier, String key) {
                return carrier.getHeader(key);
            }
        });

    // 使用提取的 Context
    try (Scope scope = extractedContext.makeCurrent()) {
        // 处理请求
    }
}
```

### 1.4 安装和配置

#### 1.4.1 系统要求

- **编译**: Java 21+ (项目使用 Java 21 编译)
- **运行时**: Java 8+ (生成的字节码兼容 Java 8)
- **Android**: API 23+ (Android 6.0+)
- **Gradle**: 8.0+
- **Maven**: 3.6+

#### 1.4.2 BOM 版本管理

OpenTelemetry Java 提供两个 BOM（Bill of Materials）：

**1. Stable BOM** (`opentelemetry-bom`):
```kotlin
dependencies {
    implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))

    // 稳定 API 和 SDK
    implementation("io.opentelemetry:opentelemetry-api")
    implementation("io.opentelemetry:opentelemetry-sdk")
}
```

**2. Alpha BOM** (`opentelemetry-bom-alpha`):
```kotlin
dependencies {
    // Alpha BOM 自动继承 Stable BOM
    implementation(platform("io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))

    // 稳定 API
    implementation("io.opentelemetry:opentelemetry-api")

    // Alpha API（实验性）
    implementation("io.opentelemetry:opentelemetry-api-incubator")
}
```

详见 [15. BOM 版本管理](#15-bom-版本管理)。

#### 1.4.3 配置方式对比

| 配置方式 | 适用场景 | 优势 | 劣势 |
|---------|---------|------|------|
| **手动配置** | 需要完全控制 | 灵活、可定制 | 代码侵入、配置繁琐 |
| **自动配置** | 快速集成 | 零代码、易维护 | 灵活性较低 |
| **Java Agent** | 无代码修改 | 完全无侵入 | 调试困难 |

---

## 2. 架构总览

### 2.1 API/SDK 分离架构

```
┌──────────────────────────────────────────────────────────┐
│                   User Application                       │
│         (仅依赖稳定的 API 接口)                           │
└────────────────────┬─────────────────────────────────────┘
                     │ 依赖关系
┌────────────────────▼─────────────────────────────────────┐
│              OpenTelemetry API (稳定)                    │
│  ┌──────────┬──────────┬──────────┬──────────┐         │
│  │ Tracer   │ Meter    │ Logger   │ Context  │         │
│  │ API      │ API      │ API      │ API      │         │
│  └──────────┴──────────┴──────────┴──────────┘         │
│                                                          │
│  - 稳定的公共接口                                         │
│  - 保证向后兼容                                           │
│  - 遵循语义化版本                                         │
└────────────────────┬─────────────────────────────────────┘
                     │ 实现关系
┌────────────────────▼─────────────────────────────────────┐
│            OpenTelemetry SDK (可插拔)                    │
│  ┌──────────┬──────────┬──────────┬──────────┐         │
│  │ Tracer   │ Meter    │ Logger   │ Context  │         │
│  │ Provider │ Provider │ Provider │ Storage  │         │
│  └─────┬────┴────┬─────┴─────┬────┴────┬─────┘         │
│        │         │           │         │               │
│    ┌───▼───┐ ┌───▼───┐  ┌───▼───┐ ┌───▼───┐          │
│    │Sampler│ │Aggre- │  │Log    │ │Thread │          │
│    │       │ │gator  │  │Record │ │Local  │          │
│    └───┬───┘ └───┬───┘  │Proc   │ └───────┘          │
│        │         │       └───┬───┘                    │
│    ┌───▼─────────▼───────────▼───┐                    │
│    │   Span/Metric/Log Processor │                    │
│    └───┬───────────────────────┬──┘                    │
│        │                       │                       │
└────────┼───────────────────────┼───────────────────────┘
         │ 导出                  │ 导出
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  OTLP Exporter   │    │ Other Exporters  │
│  (gRPC/HTTP)     │    │ (Zipkin/Prom)    │
└────────┬─────────┘    └────────┬─────────┘
         │                       │
         ▼                       ▼
┌──────────────────────────────────────────────┐
│      Backend (Jaeger/Prometheus等)           │
└──────────────────────────────────────────────┘
```

**架构特点**:
- ✅ **稳定的 API 层**: 用户代码只依赖 API，升级 SDK 不影响业务逻辑
- ✅ **可插拔的 SDK**: 可以替换不同的 SDK 实现（官方、第三方、No-op）
- ✅ **SPI 扩展点**: 导出器、采样器、资源检测器等都可自定义
- ✅ **语义化版本**: API 遵循严格的向后兼容性保证

### 2.2 三大信号架构

```
                 OpenTelemetry SDK
           ┌───────────────────────────┐
           │                           │
     ┌─────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
     │  Traces    │  │   Metrics   │  │    Logs     │
     │  (跟踪)    │  │  (指标)     │  │  (日志)     │
     └─────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                 │
     ┌─────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
     │   Span     │  │  Metric     │  │   Log       │
     │ Processor  │  │  Reader     │  │ Processor   │
     └─────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                 │
     ┌─────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
     │   Span     │  │  Metric     │  │   Log       │
     │ Exporter   │  │  Exporter   │  │ Exporter    │
     └─────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                │                 │
           └────────┬───────┴─────────────────┘
                    │
            ┌───────▼────────┐
            │ OTLP Protocol  │
            └───────┬────────┘
                    │
            ┌───────▼────────┐
            │ Collector/     │
            │ Backend        │
            └────────────────┘
```

**1. Traces (分布式跟踪)**:
- **Tracer**: 创建 Span 的工厂
- **Span**: 代表一个操作单元，包含开始/结束时间、属性、事件、状态
- **SpanProcessor**: 处理 Span（如批处理、过滤）
- **SpanExporter**: 导出 Span 到后端

**2. Metrics (指标收集)**:
- **Meter**: 创建 Metric 的工厂
- **Metric Instruments**: Counter、Histogram、Gauge 等
- **MetricReader**: 定期读取 Metric 数据
- **MetricExporter**: 导出 Metric 到后端

**3. Logs (日志记录)**:
- **Logger**: 创建 Log Record 的工厂
- **LogRecord**: 结构化日志记录
- **LogRecordProcessor**: 处理日志记录
- **LogRecordExporter**: 导出日志到后端

### 2.3 Context 传播机制

```
Thread 1                    Thread 2
┌───────────┐              ┌───────────┐
│ Context   │              │ Context   │
│ ┌───────┐ │  propagate   │ ┌───────┐ │
│ │ Span  │ ├──────────────>│ │ Span  │ │
│ │Baggage│ │              │ │Baggage│ │
│ └───────┘ │              │ └───────┘ │
└─────┬─────┘              └───────────┘
      │
      │ ThreadLocal
      ▼
┌─────────────────┐
│ ContextStorage  │
│ (ThreadLocal)   │
└─────────────────┘

Process 1                  Process 2
┌─────────────┐           ┌─────────────┐
│ HTTP Request│           │ HTTP Request│
│ Headers:    │           │ Headers:    │
│ traceparent │───────────>│ traceparent │
│ tracestate  │ HTTP       │ tracestate  │
└─────────────┘           └─────────────┘
```

**Context 传播的三个层次**:
1. **进程内传播**: 使用 ThreadLocal 存储当前 Context
2. **跨线程传播**: 手动捕获 Context 并在新线程中恢复
3. **跨进程传播**: 通过 HTTP Headers（W3C Trace Context）或其他协议传播

详见 [6. Context 传播机制](#6-context-传播机制)。

### 2.4 SPI 扩展架构

```
┌──────────────────────────────────────────┐
│        SDK Extension Points              │
├──────────────────────────────────────────┤
│                                          │
│  ┌────────────┐  ┌────────────┐        │
│  │  Sampler   │  │  Exporter  │        │
│  │    SPI     │  │    SPI     │        │
│  └─────┬──────┘  └─────┬──────┘        │
│        │                │               │
│  ┌─────▼──────┐  ┌─────▼──────┐        │
│  │ParentBased │  │OTLP        │        │
│  │AlwaysOn    │  │Zipkin      │        │
│  │TraceId     │  │Prometheus  │        │
│  │Ratio       │  │Logging     │        │
│  └────────────┘  └────────────┘        │
│                                          │
│  ┌────────────┐  ┌────────────┐        │
│  │ Resource   │  │ Context    │        │
│  │ Detector   │  │ Storage    │        │
│  │   SPI      │  │   SPI      │        │
│  └─────┬──────┘  └─────┬──────┘        │
│        │                │               │
│  ┌─────▼──────┐  ┌─────▼──────┐        │
│  │OS          │  │ThreadLocal │        │
│  │Process     │  │StormContext│        │
│  │Container   │  │ContextDB   │        │
│  │Host        │  │            │        │
│  └────────────┘  └────────────┘        │
│                                          │
└──────────────────────────────────────────┘
```

**SPI 扩展点**:
- **Sampler SPI**: 自定义采样策略
- **Exporter SPI**: 自定义导出器
- **Resource Detector SPI**: 自定义资源检测
- **Context Storage SPI**: 自定义 Context 存储

详见 [17. 扩展和插件机制](#17-扩展和插件机制)。

### 2.5 模块依赖关系

```
┌──────────────────────────────────────────┐
│          User Application                │
└─────────────┬────────────────────────────┘
              │ uses
┌─────────────▼────────────────────────────┐
│              api:all                     │
│  (Tracer, Meter, Logger, Context APIs)   │
└─────────────┬────────────────────────────┘
              │ implemented by
┌─────────────▼────────────────────────────┐
│             sdk:all                      │
│  (TracerProvider, MeterProvider, etc.)   │
└┬────────────┬──────────────┬─────────────┘
 │            │              │
 │ depends on │ depends on   │ depends on
 ▼            ▼              ▼
┌───────┐  ┌────────┐  ┌────────┐
│sdk:   │  │sdk:    │  │sdk:    │
│trace  │  │metrics │  │logs    │
└───┬───┘  └───┬────┘  └───┬────┘
    │          │           │
    │ depends on           │
    ▼          ▼           ▼
┌──────────────────────────────┐
│       sdk:common             │
└──────────────┬───────────────┘
               │ depends on
┌──────────────▼───────────────┐
│         context              │
└──────────────────────────────┘
```

**依赖层次**:
1. **context**: 最底层，无依赖
2. **sdk:common**: 依赖 context
3. **sdk:trace/metrics/logs**: 依赖 sdk:common
4. **sdk:all**: 聚合所有 SDK 模块
5. **api:all**: API 层，独立于 SDK

详见 [3. 目录结构和模块索引](#3-目录结构和模块索引)。

### 2.6 自动配置流程

```
Application Startup
       │
       ▼
┌────────────────────┐
│Read Environment    │
│Variables           │
│- OTEL_SERVICE_NAME │
│- OTEL_EXPORTER_*   │
│- OTEL_TRACES_*     │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│Load Config File    │
│- application.yml   │
│- otel.properties   │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│Discover SPI        │
│Implementations     │
│- Exporters         │
│- Samplers          │
│- ResourceDetectors │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│Build SDK           │
│Components          │
│- TracerProvider    │
│- MeterProvider     │
│- LoggerProvider    │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│Register Global     │
│OpenTelemetry       │
└────────────────────┘
```

**自动配置的优势**:
- ✅ **零代码配置**: 通过环境变量配置所有组件
- ✅ **SPI 自动发现**: 自动加载 classpath 中的扩展
- ✅ **配置优先级**: 环境变量 > 配置文件 > 默认值
- ✅ **易于测试**: 不同环境使用不同配置

详见 [16. 自动配置机制](#16-自动配置机制)。

---

## 3. 目录结构和模块索引

### 3.1 完整目录结构

```
opentelemetry-java/
├── api/                                # API 模块（4个）
│   ├── all/                            # 聚合所有稳定 API
│   ├── incubator/                      # 孵化中的实验性 API
│   ├── events/                         # 事件 API
│   └── logs/                           # 日志 API
├── sdk/                                # SDK 核心（8个）
│   ├── all/                            # 聚合所有 SDK
│   ├── common/                         # 通用 SDK 组件
│   ├── trace/                          # 分布式跟踪 SDK
│   ├── metrics/                        # 指标收集 SDK
│   ├── logs/                           # 日志 SDK
│   ├── testing/                        # SDK 测试工具
│   └── extensions/
│       ├── autoconfigure/              # 自动配置
│       └── autoconfigure-spi/          # 自动配置 SPI
├── sdk-extensions/                     # SDK 扩展（6个）
│   ├── incubator/                      # 孵化扩展
│   ├── jaeger-remote-sampler/          # Jaeger 远程采样
│   ├── aws/                            # AWS 扩展
│   ├── resources/                      # 资源检测器
│   ├── zpages/                         # zPages 调试页面
│   └── noop-api/                       # No-op 实现
├── exporters/                          # 导出器（7个）
│   ├── otlp/                           # OTLP 协议导出器
│   │   ├── all/                        # 聚合 OTLP 导出器
│   │   ├── common/                     # OTLP 通用组件
│   │   └── testing/                    # OTLP 测试工具
│   ├── zipkin/                         # Zipkin 导出器
│   ├── prometheus/                     # Prometheus 导出器
│   ├── logging/                        # 日志导出器
│   ├── logging-otlp/                   # OTLP 日志导出
│   ├── common/                         # 导出器通用组件
│   └── sender/                         # 发送器实现
│       ├── okhttp/                     # OkHttp 发送器
│       └── jdk/                        # JDK HttpClient 发送器
├── extensions/                         # 扩展（6个）
│   ├── trace-propagators/              # 跟踪传播器
│   ├── kotlin/                         # Kotlin 扩展
│   └── ...
├── integration-tests/                  # 集成测试（8个）
│   ├── otlp/                           # OTLP 集成测试
│   ├── tracecontext/                   # W3C Trace Context 测试
│   └── ...
├── context/                            # Context 传播
├── semconv/                            # 语义约定
├── opentracing-shim/                   # OpenTracing 兼容层
├── opencensus-shim/                    # OpenCensus 兼容层
├── micrometer1-shim/                   # Micrometer 1.x 兼容层
├── buildSrc/                           # 构建系统
│   ├── src/main/kotlin/
│   │   ├── otel.java-conventions.gradle.kts
│   │   ├── otel.publish-conventions.gradle.kts
│   │   ├── otel.errorprone-conventions.gradle.kts
│   │   ├── otel.spotless-conventions.gradle.kts
│   │   ├── otel.jacoco-conventions.gradle.kts
│   │   ├── otel.japicmp-conventions.gradle.kts
│   │   ├── otel.animalsniffer-conventions.gradle.kts
│   │   ├── otel.bom-conventions.gradle.kts
│   │   ├── otel.protobuf-conventions.gradle.kts
│   │   └── otel.jmh-conventions.gradle.kts
│   ├── README.md                       # buildSrc 详细文档
│   └── docs/
│       ├── bom-guide.md                # BOM 指南
│       └── animalsniffer-guide.md      # AnimalSniffer 指南
├── custom-checks/                      # ErrorProne 自定义检查
│   ├── src/main/java/
│   │   ├── OtelInternalJavadoc.java
│   │   └── OtelPrivateConstructorForUtilityClass.java
│   └── README.md                       # custom-checks 详细文档
├── bom/                                # 稳定版 BOM
├── bom-alpha/                          # Alpha BOM
├── dependencyManagement/               # 依赖管理
├── animal-sniffer-signature/           # Android API 签名
├── perf-harness/                       # 性能测试
├── benchmarks/                         # JMH 基准测试
├── testing-internal/                   # 内部测试工具
├── build.gradle.kts                    # 根构建脚本
├── settings.gradle.kts                 # Gradle 设置
├── README.md                           # 英文项目 README
└── README.zh-CN.md                     # 中文技术文档（本文档）
```

### 3.2 模块分类和说明

#### 3.2.1 API 模块（4个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `api:all` | `io.opentelemetry:opentelemetry-api` | 聚合所有稳定 API | 推荐使用，包含 Trace/Meter/Context API |
| `api:incubator` | `io.opentelemetry:opentelemetry-api-incubator` | 实验性 API | 早期采用者，API 可能变更 |
| `api:events` | `io.opentelemetry:opentelemetry-api-events` | 事件 API | 结构化事件记录 |
| `api:logs` | `io.opentelemetry:opentelemetry-api-logs` | 日志 API | 日志桥接器实现 |

#### 3.2.2 SDK 核心（8个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `sdk:all` | `io.opentelemetry:opentelemetry-sdk` | 聚合所有 SDK | 推荐使用，包含所有信号的 SDK |
| `sdk:common` | `io.opentelemetry:opentelemetry-sdk-common` | 通用 SDK 组件 | 内部依赖，用户通常不直接使用 |
| `sdk:trace` | `io.opentelemetry:opentelemetry-sdk-trace` | 分布式跟踪 SDK | 仅需 Trace 功能时使用 |
| `sdk:metrics` | `io.opentelemetry:opentelemetry-sdk-metrics` | 指标收集 SDK | 仅需 Metrics 功能时使用 |
| `sdk:logs` | `io.opentelemetry:opentelemetry-sdk-logs` | 日志 SDK | 仅需 Logs 功能时使用 |
| `sdk:testing` | `io.opentelemetry:opentelemetry-sdk-testing` | SDK 测试工具 | 单元测试中验证遥测数据 |
| `sdk-extensions:autoconfigure` | `io.opentelemetry:opentelemetry-sdk-extension-autoconfigure` | 自动配置 | 零代码配置 SDK |
| `sdk-extensions:autoconfigure-spi` | `io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi` | 自动配置 SPI | 扩展自动配置 |

#### 3.2.3 导出器（7个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `exporters:otlp:all` | `io.opentelemetry:opentelemetry-exporter-otlp` | OTLP 协议导出器 | 推荐使用，导出到 Collector |
| `exporters:zipkin` | `io.opentelemetry:opentelemetry-exporter-zipkin` | Zipkin 导出器 | 导出到 Zipkin 后端 |
| `exporters:prometheus` | `io.opentelemetry:opentelemetry-exporter-prometheus` | Prometheus 导出器 | 暴露 Prometheus metrics 端点 |
| `exporters:logging` | `io.opentelemetry:opentelemetry-exporter-logging` | 日志导出器 | 调试、开发环境 |
| `exporters:logging-otlp` | `io.opentelemetry:opentelemetry-exporter-logging-otlp` | OTLP 日志导出 | 调试 OTLP 格式 |
| `exporters:common` | `io.opentelemetry:opentelemetry-exporter-common` | 导出器通用组件 | 内部依赖 |
| `exporters:sender:*` | `io.opentelemetry:opentelemetry-exporter-sender-*` | 发送器实现 | 内部依赖（OkHttp/JDK） |

#### 3.2.4 扩展（6个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `sdk-extensions:incubator` | `io.opentelemetry:opentelemetry-sdk-extension-incubator` | 孵化扩展 | 实验性 SDK 功能 |
| `sdk-extensions:jaeger-remote-sampler` | `io.opentelemetry:opentelemetry-sdk-extension-jaeger-remote-sampler` | Jaeger 远程采样 | 动态采样配置 |
| `sdk-extensions:aws` | `io.opentelemetry:opentelemetry-sdk-extension-aws` | AWS 扩展 | AWS X-Ray 集成 |
| `sdk-extensions:resources` | `io.opentelemetry:opentelemetry-sdk-extension-resources` | 资源检测器 | 自动检测运行环境 |
| `extensions:trace-propagators` | `io.opentelemetry:opentelemetry-extension-trace-propagators` | 跟踪传播器 | B3、Jaeger 传播协议 |
| `extensions:kotlin` | `io.opentelemetry:opentelemetry-extension-kotlin` | Kotlin 扩展 | Kotlin 协程支持 |

#### 3.2.5 集成/兼容层（8个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `opentracing-shim` | `io.opentelemetry:opentelemetry-opentracing-shim` | OpenTracing 兼容层 | 从 OpenTracing 迁移 |
| `opencensus-shim` | `io.opentelemetry:opentelemetry-opencensus-shim` | OpenCensus 兼容层 | 从 OpenCensus 迁移 |
| `micrometer1-shim` | `io.opentelemetry:opentelemetry-micrometer1-shim` | Micrometer 1.x 兼容 | Micrometer Metrics 桥接 |
| `integration-tests:*` | - | 集成测试 | 验证协议兼容性 |

#### 3.2.6 构建和质量（6个）

| 模块 | 说明 | 详细文档 |
|------|------|---------|
| `buildSrc` | 构建系统（10个约定插件） | [buildSrc/README.md](buildSrc/README.md) |
| `custom-checks` | ErrorProne 自定义检查 | [custom-checks/README.md](custom-checks/README.md) |
| `bom` | 稳定版 BOM | [buildSrc/docs/bom-guide.md](buildSrc/docs/bom-guide.md) |
| `bom-alpha` | Alpha BOM | [buildSrc/docs/bom-guide.md](buildSrc/docs/bom-guide.md) |
| `dependencyManagement` | 依赖管理 | [buildSrc/README.md](buildSrc/README.md) |
| `animal-sniffer-signature` | Android API 签名 | [buildSrc/docs/animalsniffer-guide.md](buildSrc/docs/animalsniffer-guide.md) |

#### 3.2.7 基础设施（8个）

| 模块 | 坐标 | 说明 | 使用场景 |
|------|------|------|---------|
| `context` | `io.opentelemetry:opentelemetry-context` | Context 传播 | 内部依赖 |
| `semconv` | `io.opentelemetry:opentelemetry-semconv` | 语义约定 | 标准属性名称 |
| `perf-harness` | - | 性能测试 | 性能基准 |
| `benchmarks` | - | JMH 基准测试 | 微基准测试 |
| `testing-internal` | - | 内部测试工具 | 测试框架 |

### 3.3 模块间依赖关系矩阵

```
       │ context │ sdk:common │ sdk:trace │ sdk:metrics │ sdk:logs │ api:all │ sdk:all
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
context│         │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
sdk:   │    ✓    │            │           │             │          │         │
common │         │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
sdk:   │    ✓    │     ✓      │           │             │          │         │
trace  │         │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
sdk:   │    ✓    │     ✓      │           │             │          │         │
metrics│         │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
sdk:   │    ✓    │     ✓      │           │             │          │         │
logs   │         │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
api:all│    ✓    │            │           │             │          │         │
───────┼─────────┼────────────┼───────────┼─────────────┼──────────┼─────────┼─────────
sdk:all│    ✓    │     ✓      │     ✓     │      ✓      │    ✓     │    ✓    │
```

**说明**:
- `✓` 表示该模块依赖列标题的模块
- `context` 是最底层模块，无依赖
- `sdk:common` 依赖 `context`
- `sdk:trace/metrics/logs` 依赖 `sdk:common` 和 `context`
- `sdk:all` 聚合所有 SDK 模块
- `api:all` 独立于 SDK，仅依赖 `context`

---

**相关章节**:
- → 下一节: [4. API 层设计](#4-api-层设计)
- ↑ 返回目录: [目录](#📑-目录)

---

**最后更新**: 2026-01-09
**文档版本**: 1.0.0
**项目版本**: 1.58.0-SNAPSHOT

**维护者**: OpenTelemetry Java 项目组
**问题反馈**: [GitHub Issues](https://github.com/open-telemetry/opentelemetry-java/issues)
**贡献指南**: [CONTRIBUTING.md](https://github.com/open-telemetry/opentelemetry-java/blob/main/CONTRIBUTING.md)

# 第二部分：核心架构

## 4. API 层设计

### 4.1 API 模块组织

OpenTelemetry Java API 层由 4 个核心模块组成，提供稳定的公共接口：

```
api/
├── all/                 # 聚合模块（推荐使用）
│   └── 包含所有稳定 API
├── incubator/          # 孵化模块（实验性 API）
│   └── 未来可能进入稳定 API 的特性
├── events/             # 事件 API
│   └── 结构化事件记录
└── logs/               # 日志 API
    └── 日志桥接器接口
```

**模块依赖关系**:
```
api:all
├── api:trace (内部模块)
├── api:metrics (内部模块)
├── api:context (内部模块)
└── api:baggage (内部模块)

api:incubator
└── api:all

api:events
└── api:all

api:logs
└── api:all
```

### 4.2 接口设计原则

#### 4.2.1 最小化原则（Minimal API Surface）

**目标**: 只暴露必需的接口，减少用户学习成本和 API 维护负担。

**示例 - Tracer API**:
```java
public interface Tracer {
    // 仅提供核心方法
    SpanBuilder spanBuilder(String spanName);
}

public interface SpanBuilder {
    // Builder 模式提供灵活配置
    SpanBuilder setParent(Context context);
    SpanBuilder setSpanKind(SpanKind spanKind);
    SpanBuilder setAttribute(String key, String value);
    Span startSpan();
}
```

**对比过度设计的反例**:
```java
// ❌ 不推荐：过多方法导致 API 臃肿
public interface Tracer {
    Span startSpan(String name);
    Span startClientSpan(String name);
    Span startServerSpan(String name);
    Span startInternalSpan(String name);
    // ... 20+ 方法
}
```

#### 4.2.2 稳定性保证

**语义化版本控制**:
- **Major 版本**（1.x → 2.x）: 允许破坏性变更
- **Minor 版本**（1.0.0 → 1.1.0）: 新增功能，保持向后兼容
- **Patch 版本**（1.0.0 → 1.0.1）: Bug 修复，完全兼容

**API 稳定性标记**:
```java
// 稳定 API（无注解）
public interface Tracer {
    SpanBuilder spanBuilder(String spanName);
}

// 实验性 API（明确标记）
@Experimental
public interface EventEmitter {
    void emit(String eventName, Attributes attributes);
}
```

#### 4.2.3 可扩展性设计

**通过 SPI 实现扩展**:
```
API 层（接口）
    ↓ 定义
SDK 层（默认实现）
    ↓ 可替换
用户自定义实现
```

**示例 - Context 存储扩展**:
```java
// API 定义存储接口
public interface ContextStorage {
    Context current();
    Scope attach(Context context);
}

// SDK 提供默认实现
public class ThreadLocalContextStorage implements ContextStorage {
    // ThreadLocal 实现
}

// 用户可自定义实现（如 Kotlin Coroutine）
public class CoroutineContextStorage implements ContextStorage {
    // 协程实现
}
```

### 4.3 向后兼容策略

#### 4.3.1 废弃流程

**第 1 步：标记为 @Deprecated**
```java
/**
 * @deprecated Use {@link #newMethod()} instead. This method will be removed in 2.0.0.
 */
@Deprecated
public void oldMethod() {
    // 旧实现
}
```

**第 2 步：提供迁移指南**
```markdown
## 迁移指南：oldMethod → newMethod

**影响版本**: 1.35.0+
**移除版本**: 2.0.0

**旧代码**:
```java
tracer.oldMethod();
```

**新代码**:
```java
tracer.newMethod();
```
```

**第 3 步：至少保留 1 个 Major 版本周期**
```
1.35.0: 标记 @Deprecated
1.36.0 - 1.x: 继续支持
2.0.0: 移除
```

#### 4.3.2 兼容性检查（JApiCmp）

自动检测破坏性变更：
```bash
./gradlew japicmp

# 报告示例
Class Foo: Method bar() has been removed
Class Foo: Method baz() parameter type changed from String to int
```

### 4.4 API vs SPI 的区别

| 特性 | API | SPI |
|------|-----|-----|
| **定义** | 供用户调用的接口 | 供实现者扩展的接口 |
| **稳定性** | 高（语义化版本） | 较低（可能频繁变更） |
| **使用方** | 应用开发者 | SDK 开发者、扩展开发者 |
| **示例** | `Tracer`, `Span`, `Meter` | `SpanExporter`, `Sampler`, `ContextStorage` |
| **包位置** | `io.opentelemetry.api.*` | `io.opentelemetry.sdk.*` |

**API 示例**（用户代码）:
```java
// 用户只调用 API
Tracer tracer = openTelemetry.getTracer("my-app");
Span span = tracer.spanBuilder("operation").startSpan();
```

**SPI 示例**（扩展开发）:
```java
// 实现自定义导出器
public class MyExporter implements SpanExporter {
    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        // 自定义导出逻辑
    }
}
```

### 4.5 Builder 模式应用

#### 4.5.1 设计目标

- ✅ **类型安全**: 编译时检查参数
- ✅ **流式 API**: 链式调用提升可读性
- ✅ **可选参数**: 避免构造函数过载
- ✅ **默认值**: 提供合理的默认配置

#### 4.5.2 Span Builder 示例

```java
Span span = tracer.spanBuilder("processRequest")
    .setParent(Context.current())               // 可选：设置父 Span
    .setSpanKind(SpanKind.INTERNAL)            // 可选：设置 Span 类型
    .setAttribute("user.id", "12345")          // 可选：添加属性
    .setAttribute("http.method", "POST")
    .setStartTimestamp(startTime)              // 可选：自定义开始时间
    .startSpan();                              // 必需：创建 Span
```

**Builder 接口设计**:
```java
public interface SpanBuilder {
    // 返回 this 支持链式调用
    SpanBuilder setParent(Context context);
    SpanBuilder setSpanKind(SpanKind spanKind);
    SpanBuilder setAttribute(String key, String value);
    SpanBuilder setAttribute(String key, long value);
    SpanBuilder setStartTimestamp(long startTimestamp, TimeUnit unit);
    
    // 终止方法
    Span startSpan();
}
```

#### 4.5.3 Meter Builder 示例

```java
LongCounter counter = meter.counterBuilder("requests")
    .setDescription("Total number of requests")  // 可选：描述
    .setUnit("1")                                // 可选：单位
    .build();                                    // 创建 Counter
```

### 4.6 核心 API 接口详解

#### 4.6.1 Tracer API

**获取 Tracer**:
```java
Tracer tracer = openTelemetry.getTracer(
    "instrumentation-library-name",  // 必需：库名称
    "1.0.0"                          // 可选：库版本
);
```

**创建 Span**:
```java
Span span = tracer.spanBuilder("operation")
    .startSpan();

try (Scope scope = span.makeCurrent()) {
    // Span 自动成为当前上下文
    
    // 添加属性
    span.setAttribute("key", "value");
    
    // 添加事件
    span.addEvent("Processing started");
    
    // 记录异常
    try {
        doWork();
    } catch (Exception e) {
        span.recordException(e);
        throw e;
    }
} finally {
    span.end();
}
```

#### 4.6.2 Meter API

**获取 Meter**:
```java
Meter meter = openTelemetry.getMeter(
    "instrumentation-library-name",
    "1.0.0"
);
```

**Counter（计数器）**:
```java
LongCounter counter = meter.counterBuilder("requests")
    .setDescription("Total requests")
    .setUnit("1")
    .build();

// 递增
counter.add(1, Attributes.of(
    AttributeKey.stringKey("endpoint"), "/api/users"
));
```

**Histogram（直方图）**:
```java
DoubleHistogram histogram = meter.histogramBuilder("response.time")
    .setDescription("Response time distribution")
    .setUnit("ms")
    .build();

// 记录值
histogram.record(123.45, Attributes.of(
    AttributeKey.stringKey("endpoint"), "/api/users"
));
```

**Gauge（仪表盘）**:
```java
// ObservableGauge（异步观测）
meter.gaugeBuilder("memory.usage")
    .setDescription("Current memory usage")
    .setUnit("bytes")
    .buildWithCallback(measurement -> {
        long memoryUsage = Runtime.getRuntime().totalMemory();
        measurement.record(memoryUsage);
    });
```

#### 4.6.3 Logger API

**获取 Logger**:
```java
Logger logger = openTelemetry.getLoggerProvider()
    .get("instrumentation-library-name");
```

**发出日志**:
```java
logger.logRecordBuilder()
    .setSeverity(Severity.INFO)
    .setBody("User logged in")
    .setAttribute("user.id", "12345")
    .setAttribute("action", "login")
    .emit();
```

---

## 5. SDK 层设计

### 5.1 SDK 模块组织

SDK 层由 8 个核心模块组成，提供 API 的默认实现：

```
sdk/
├── all/                     # 聚合模块（推荐使用）
│   └── 包含所有 SDK
├── common/                  # 通用组件
│   ├── Clock
│   ├── Resource
│   └── CompletableResultCode
├── trace/                   # 分布式跟踪 SDK
│   ├── SdkTracerProvider
│   ├── SdkSpan
│   ├── SpanProcessor
│   └── SpanExporter
├── metrics/                 # 指标收集 SDK
│   ├── SdkMeterProvider
│   ├── MetricReader
│   └── MetricExporter
├── logs/                    # 日志 SDK
│   ├── SdkLoggerProvider
│   ├── LogRecordProcessor
│   └── LogRecordExporter
├── testing/                 # 测试工具
│   ├── InMemorySpanExporter
│   └── TestClock
└── extensions/
    ├── autoconfigure/       # 自动配置
    └── autoconfigure-spi/   # 自动配置 SPI
```

### 5.2 可插拔架构

#### 5.2.1 Exporter 扩展点

```
SpanExporter (SPI)
├── OtlpGrpcSpanExporter        # gRPC 传输
├── OtlpHttpSpanExporter        # HTTP 传输
├── ZipkinSpanExporter          # Zipkin 格式
├── LoggingSpanExporter         # 日志输出（调试）
└── CustomSpanExporter          # 用户自定义
```

**自定义 Exporter 示例**:
```java
public class CustomSpanExporter implements SpanExporter {
    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        spans.forEach(span -> {
            // 自定义导出逻辑
            System.out.println("Exporting span: " + span.getName());
        });
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode flush() {
        // 刷新缓冲区
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        // 清理资源
        return CompletableResultCode.ofSuccess();
    }
}
```

**注册 Exporter**:
```java
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(new CustomSpanExporter()).build())
    .build();
```

#### 5.2.2 Sampler 扩展点

```
Sampler (SPI)
├── AlwaysOnSampler             # 100% 采样
├── AlwaysOffSampler            # 0% 采样
├── TraceIdRatioBasedSampler    # 基于 TraceId 的比例采样
├── ParentBasedSampler          # 继承父 Span 的采样决策
└── CustomSampler               # 用户自定义
```

**自定义 Sampler 示例**:
```java
public class CustomSampler implements Sampler {
    @Override
    public SamplingResult shouldSample(
        Context parentContext,
        String traceId,
        String name,
        SpanKind spanKind,
        Attributes attributes,
        List<LinkData> parentLinks
    ) {
        // 自定义采样逻辑
        if (attributes.get(AttributeKey.stringKey("priority")) != null) {
            return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
        }
        return SamplingResult.create(SamplingDecision.DROP);
    }

    @Override
    public String getDescription() {
        return "CustomSampler";
    }
}
```

**注册 Sampler**:
```java
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setSampler(new CustomSampler())
    .build();
```

#### 5.2.3 Resource Detector 扩展点

```
ResourceDetector (SPI)
├── OsResourceDetector          # 操作系统信息
├── ProcessResourceDetector     # 进程信息
├── ContainerResourceDetector   # 容器信息（Docker/K8s）
├── HostResourceDetector        # 主机信息
└── CustomResourceDetector      # 用户自定义
```

**自定义 Resource Detector 示例**:
```java
public class CustomResourceDetector implements ResourceDetector {
    @Override
    public Resource detect() {
        return Resource.create(
            Attributes.of(
                AttributeKey.stringKey("custom.key"), "custom.value",
                AttributeKey.stringKey("deployment.environment"), "production"
            )
        );
    }
}
```

**注册 Resource Detector**:
```java
Resource resource = Resource.getDefault()
    .merge(new CustomResourceDetector().detect());

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setResource(resource)
    .build();
```

### 5.3 配置和生命周期管理

#### 5.3.1 SDK 初始化流程

```
1. 创建 Resource（资源信息）
   ↓
2. 创建 Exporter（导出器）
   ↓
3. 创建 Processor（处理器）
   ↓
4. 创建 TracerProvider/MeterProvider/LoggerProvider
   ↓
5. 创建 OpenTelemetry 实例
   ↓
6. 注册为全局实例（可选）
```

**完整初始化示例**:
```java
// 1. 创建 Resource
Resource resource = Resource.getDefault()
    .merge(Resource.create(
        Attributes.of(
            AttributeKey.stringKey("service.name"), "my-service",
            AttributeKey.stringKey("service.version"), "1.0.0"
        )
    ));

// 2. 创建 Exporter
SpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();

// 3. 创建 Processor
SpanProcessor spanProcessor = BatchSpanProcessor.builder(spanExporter)
    .setScheduleDelay(Duration.ofSeconds(5))
    .setMaxQueueSize(2048)
    .setMaxExportBatchSize(512)
    .build();

// 4. 创建 TracerProvider
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setResource(resource)
    .addSpanProcessor(spanProcessor)
    .setSampler(Sampler.traceIdRatioBased(0.1))  // 10% 采样
    .build();

// 5. 创建 OpenTelemetry
OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    .setPropagators(ContextPropagators.create(
        TextMapPropagator.composite(
            W3CTraceContextPropagator.getInstance(),
            W3CBaggagePropagator.getInstance()
        )
    ))
    .buildAndRegisterGlobal();  // 注册为全局实例

// 6. 关闭时清理
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    tracerProvider.close();
}));
```

#### 5.3.2 生命周期管理

**关闭顺序**:
```
Application Shutdown
    ↓
1. 停止接受新 Span
    ↓
2. Processor.shutdown()
    ├── 刷新缓冲区
    └── 等待导出完成
    ↓
3. Exporter.shutdown()
    ├── 关闭连接
    └── 释放资源
    ↓
4. TracerProvider.close()
    └── 清理内部状态
```

**优雅关闭示例**:
```java
// 方式 1：手动关闭
tracerProvider.close();

// 方式 2：使用 Shutdown Hook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("Shutting down OpenTelemetry...");
    tracerProvider.close();
    System.out.println("OpenTelemetry shutdown complete.");
}));

// 方式 3：使用 CompletableResultCode 异步关闭
CompletableResultCode result = tracerProvider.shutdown();
result.join(10, TimeUnit.SECONDS);  // 等待最多 10 秒
if (result.isSuccess()) {
    System.out.println("Shutdown successful");
} else {
    System.err.println("Shutdown failed or timed out");
}
```

### 5.4 资源管理和线程安全

#### 5.4.1 资源管理

**Span 生命周期**:
```java
Span span = tracer.spanBuilder("operation").startSpan();
try (Scope scope = span.makeCurrent()) {
    // Scope 实现 AutoCloseable
    // try-with-resources 自动恢复 Context
    doWork();
} finally {
    span.end();  // 必须调用 end()
}
```

**避免资源泄漏**:
```java
// ✅ 正确：使用 try-finally
Span span = tracer.spanBuilder("operation").startSpan();
try {
    doWork();
} finally {
    span.end();
}

// ❌ 错误：忘记 end()
Span span = tracer.spanBuilder("operation").startSpan();
doWork();
// Span 永远不会结束！
```

#### 5.4.2 线程安全

**OpenTelemetry 实例线程安全**:
```java
// 单例模式，全局共享
private static final OpenTelemetry OPENTELEMETRY = 
    OpenTelemetrySdk.builder()...build();

// 多线程安全访问
public void method1() {
    Tracer tracer = OPENTELEMETRY.getTracer("service1");
}

public void method2() {
    Tracer tracer = OPENTELEMETRY.getTracer("service2");
}
```

**Span 不是线程安全的**:
```java
// ❌ 错误：跨线程共享 Span
Span span = tracer.spanBuilder("operation").startSpan();
CompletableFuture.runAsync(() -> {
    span.setAttribute("key", "value");  // 线程不安全！
});

// ✅ 正确：传播 Context，每个线程创建子 Span
Span span = tracer.spanBuilder("operation").startSpan();
Context context = Context.current().with(span);

CompletableFuture.runAsync(() -> {
    try (Scope scope = context.makeCurrent()) {
        Span childSpan = tracer.spanBuilder("async-work").startSpan();
        try {
            childSpan.setAttribute("key", "value");  // 安全
        } finally {
            childSpan.end();
        }
    }
});
```

### 5.5 性能优化策略

#### 5.5.1 对象池化

**Span 对象重用**:
```java
// SDK 内部使用对象池
private static final ObjectPool<SdkSpan> SPAN_POOL = 
    new ObjectPool<>(SdkSpan::new, 100);

// 创建 Span
Span span = SPAN_POOL.acquire();
span.initialize(...);

// 回收 Span
span.end();
SPAN_POOL.release(span);
```

#### 5.5.2 批处理

**BatchSpanProcessor 配置**:
```java
SpanProcessor processor = BatchSpanProcessor.builder(exporter)
    .setScheduleDelay(Duration.ofSeconds(5))      // 每 5 秒导出一次
    .setMaxQueueSize(2048)                        // 队列最大 2048 个 Span
    .setMaxExportBatchSize(512)                   // 每批最多 512 个 Span
    .setExporterTimeout(Duration.ofSeconds(30))   // 导出超时 30 秒
    .build();
```

**批处理流程**:
```
Span.end()
    ↓
加入队列
    ↓
队列满 或 5 秒超时
    ↓
批量导出（512 个/批）
    ↓
Exporter.export()
```

#### 5.5.3 采样优化

**避免不必要的 Span 创建**:
```java
// 方式 1：使用采样器
Sampler sampler = Sampler.traceIdRatioBased(0.1);  // 10% 采样

// 方式 2：手动判断
if (Span.current().getSpanContext().isSampled()) {
    // 只在采样时执行昂贵操作
    span.setAttribute("expensive.data", computeExpensiveData());
}
```

---

## 6. Context 传播机制

### 6.1 Context API 设计

**Context 接口**:
```java
public interface Context {
    // 获取当前 Context
    static Context current();
    
    // 创建新 Context（不可变）
    <V> Context with(ContextKey<V> key, V value);
    
    // 获取值
    <V> V get(ContextKey<V> key);
    
    // 使 Context 成为当前
    Scope makeCurrent();
}
```

**ContextKey 示例**:
```java
// 定义 ContextKey
private static final ContextKey<String> USER_ID_KEY = 
    ContextKey.named("user.id");

// 存储值
Context context = Context.current().with(USER_ID_KEY, "12345");

// 获取值
String userId = context.get(USER_ID_KEY);
```

### 6.2 ThreadLocal 实现

**默认实现（ThreadLocalContextStorage）**:
```java
public class ThreadLocalContextStorage implements ContextStorage {
    private static final ThreadLocal<Context> THREAD_LOCAL = 
        new ThreadLocal<>() {
            @Override
            protected Context initialValue() {
                return Context.root();
            }
        };

    @Override
    public Context current() {
        return THREAD_LOCAL.get();
    }

    @Override
    public Scope attach(Context context) {
        Context previous = current();
        THREAD_LOCAL.set(context);
        
        return () -> {
            THREAD_LOCAL.set(previous);  // 恢复之前的 Context
        };
    }
}
```

**使用示例**:
```java
Span span = tracer.spanBuilder("operation").startSpan();

// attach 使 Context 成为当前
Scope scope = span.makeCurrent();
try {
    // 当前线程的 Context 已包含 span
    Span current = Span.current();  // 返回 span
    doWork();
} finally {
    scope.close();  // 恢复之前的 Context
}
```

### 6.3 跨线程传播

#### 6.3.1 ExecutorService 传播

**问题**: ThreadLocal 不会自动传播到新线程。

```java
// ❌ 错误：Context 丢失
Span span = tracer.spanBuilder("parent").startSpan();
try (Scope scope = span.makeCurrent()) {
    executor.submit(() -> {
        Span current = Span.current();  // 返回 INVALID（非 parent）
    });
}
```

**解决方案**: 手动捕获和恢复 Context。

```java
// ✅ 正确：传播 Context
Span span = tracer.spanBuilder("parent").startSpan();
try (Scope scope = span.makeCurrent()) {
    Context context = Context.current();  // 捕获当前 Context
    
    executor.submit(() -> {
        try (Scope asyncScope = context.makeCurrent()) {  // 恢复 Context
            Span current = Span.current();  // 返回 parent
            
            // 创建子 Span
            Span childSpan = tracer.spanBuilder("child").startSpan();
            try {
                doWork();
            } finally {
                childSpan.end();
            }
        }
    });
} finally {
    span.end();
}
```

#### 6.3.2 CompletableFuture 传播

**使用包装器自动传播**:
```java
public class ContextPropagatingExecutor implements Executor {
    private final Executor delegate;

    public ContextPropagatingExecutor(Executor delegate) {
        this.delegate = delegate;
    }

    @Override
    public void execute(Runnable command) {
        Context context = Context.current();  // 捕获
        delegate.execute(() -> {
            try (Scope scope = context.makeCurrent()) {  // 恢复
                command.run();
            }
        });
    }
}

// 使用
Executor executor = new ContextPropagatingExecutor(
    Executors.newFixedThreadPool(10)
);

CompletableFuture.runAsync(() -> {
    // Context 自动传播
    Span current = Span.current();
}, executor);
```

### 6.4 跨进程传播

#### 6.4.1 W3C Trace Context 协议

**HTTP Headers**:
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             ││ └─────────────────┬──────────────────┘ └──────┬──────┘ │
             ││              Trace ID (16 bytes)         Span ID     Flags
             │└─ Version (00)                            (8 bytes)   (01=采样)
             └─ 格式标识
```

**客户端注入**:
```java
// 1. 配置 Propagator
TextMapPropagator propagator = W3CTraceContextPropagator.getInstance();

// 2. 创建 HTTP 请求
HttpURLConnection connection = (HttpURLConnection) url.openConnection();

// 3. 注入 Context
propagator.inject(Context.current(), connection, new TextMapSetter<HttpURLConnection>() {
    @Override
    public void set(HttpURLConnection carrier, String key, String value) {
        carrier.setRequestProperty(key, value);
    }
});

// 发送请求
connection.connect();
```

**服务端提取**:
```java
// 1. 配置 Propagator
TextMapPropagator propagator = W3CTraceContextPropagator.getInstance();

// 2. 从 HTTP 请求提取
Context extractedContext = propagator.extract(
    Context.current(),
    request,
    new TextMapGetter<HttpServletRequest>() {
        @Override
        public Iterable<String> keys(HttpServletRequest carrier) {
            return Collections.list(carrier.getHeaderNames());
        }

        @Override
        public String get(HttpServletRequest carrier, String key) {
            return carrier.getHeader(key);
        }
    }
);

// 3. 使用提取的 Context
try (Scope scope = extractedContext.makeCurrent()) {
    // 处理请求
    Span span = tracer.spanBuilder("handleRequest").startSpan();
    try {
        // Span 自动继承父 Span
    } finally {
        span.end();
    }
}
```

#### 6.4.2 多 Propagator 组合

```java
TextMapPropagator compositePropagator = TextMapPropagator.composite(
    W3CTraceContextPropagator.getInstance(),  // traceparent
    W3CBaggagePropagator.getInstance(),       // baggage
    JaegerPropagator.getInstance()            // uber-trace-id
);

OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
    .setPropagators(ContextPropagators.create(compositePropagator))
    .build();
```

---

## 7. Traces（分布式跟踪）

### 7.1 Tracer API 和 Span 模型

#### 7.1.1 Span 数据模型

```
Span
├── Trace ID (16 bytes)          # 唯一标识一次完整请求
├── Span ID (8 bytes)            # 唯一标识当前 Span
├── Parent Span ID (8 bytes)     # 父 Span ID（根 Span 为 null）
├── Name (String)                # Span 名称
├── Start Timestamp (long)       # 开始时间（纳秒）
├── End Timestamp (long)         # 结束时间（纳秒）
├── Span Kind (enum)             # Span 类型
│   ├── INTERNAL                 # 内部操作
│   ├── SERVER                   # 服务端请求
│   ├── CLIENT                   # 客户端请求
│   ├── PRODUCER                 # 消息生产者
│   └── CONSUMER                 # 消息消费者
├── Status (enum)                # 状态
│   ├── UNSET                    # 未设置
│   ├── OK                       # 成功
│   └── ERROR                    # 错误
├── Attributes (Map)             # 键值对属性
├── Events (List)                # 时间点事件
│   └── Event
│       ├── Name
│       ├── Timestamp
│       └── Attributes
├── Links (List)                 # 与其他 Span 的关联
│   └── Link
│       ├── Span Context
│       └── Attributes
└── Resource (Attributes)        # 资源信息（进程、服务等）
```

#### 7.1.2 Span 创建示例

```java
Span span = tracer.spanBuilder("processRequest")
    .setSpanKind(SpanKind.SERVER)
    .setParent(extractedContext)
    .setAttribute("http.method", "POST")
    .setAttribute("http.url", "/api/users")
    .setAttribute("http.target", "/api/users")
    .setAttribute("http.route", "/api/users")
    .setStartTimestamp(startTime, TimeUnit.NANOSECONDS)
    .startSpan();

try (Scope scope = span.makeCurrent()) {
    // 添加更多属性
    span.setAttribute("user.id", "12345");
    span.setAttribute("request.size", 1024);
    
    // 添加事件
    span.addEvent("Database query started");
    
    // 业务逻辑
    processRequest();
    
    // 添加事件（带属性）
    span.addEvent("Database query completed",
        Attributes.of(
            AttributeKey.longKey("rows"), 10L,
            AttributeKey.longKey("duration.ms"), 50L
        ));
    
    // 设置状态
    span.setStatus(StatusCode.OK);
} catch (Exception e) {
    // 记录异常
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, "Request failed: " + e.getMessage());
    throw e;
} finally {
    span.end();
}
```

### 7.2 Span 生命周期

```
1. spanBuilder()
    ↓
2. 配置 (setSpanKind, setAttribute, ...)
    ↓
3. startSpan()
    ├── 生成 Span ID
    ├── 记录开始时间
    └── 触发 Sampler
    ↓
4. makeCurrent() [可选]
    └── 设置为当前 Context
    ↓
5. 业务逻辑
    ├── setAttribute()
    ├── addEvent()
    └── recordException()
    ↓
6. setStatus()
    ↓
7. end()
    ├── 记录结束时间
    ├── 触发 SpanProcessor.onEnd()
    └── 加入导出队列
    ↓
8. SpanProcessor 处理
    └── BatchSpanProcessor 批量导出
    ↓
9. SpanExporter.export()
    └── 发送到后端
```

### 7.3 Span 关系

#### 7.3.1 Parent-Child 关系

```
请求流程:
Client → Service A → Service B → Database

Trace 结构:
Trace (abc123)
└── Span 1 (Client REQUEST)
    └── Span 2 (Service A handleRequest)
        ├── Span 3 (Service A → Service B RPC)
        │   └── Span 4 (Service B handleRequest)
        │       └── Span 5 (Database Query)
        └── Span 6 (Service A processResult)
```

**代码示例**:
```java
// Service A
Span parentSpan = tracer.spanBuilder("handleRequest").startSpan();
try (Scope scope = parentSpan.makeCurrent()) {
    // 创建子 Span（自动继承父 Context）
    Span childSpan = tracer.spanBuilder("callServiceB")
        .setSpanKind(SpanKind.CLIENT)
        .startSpan();
    
    try (Scope childScope = childSpan.makeCurrent()) {
        callServiceB();
    } finally {
        childSpan.end();
    }
} finally {
    parentSpan.end();
}
```

#### 7.3.2 Follows-From 关系

用于异步操作，表示因果关系但不是严格的父子关系。

```java
// 创建父 Span
Span parentSpan = tracer.spanBuilder("queueMessage").startSpan();
try {
    sendToQueue(message);
} finally {
    parentSpan.end();
}

// 异步消费者（使用 Link）
Span consumerSpan = tracer.spanBuilder("processMessage")
    .addLink(parentSpan.getSpanContext())  // 添加 Link
    .startSpan();
try {
    processMessage(message);
} finally {
    consumerSpan.end();
}
```

### 7.4 采样策略

#### 7.4.1 AlwaysOnSampler

```java
// 100% 采样（开发/调试环境）
Sampler sampler = Sampler.alwaysOn();
```

#### 7.4.2 AlwaysOffSampler

```java
// 0% 采样（完全禁用）
Sampler sampler = Sampler.alwaysOff();
```

#### 7.4.3 TraceIdRatioBasedSampler

```java
// 10% 采样（生产环境）
Sampler sampler = Sampler.traceIdRatioBased(0.1);
```

**工作原理**:
```
TraceId (16 bytes) → 转换为 long → 取模 → 判断是否 < 阈值
0af7651916cd43dd → 0x0af765... → % 10000 < 1000 → 采样
```

#### 7.4.4 ParentBasedSampler

```java
// 继承父 Span 的采样决策
Sampler sampler = Sampler.parentBased(
    Sampler.traceIdRatioBased(0.1)  // 根 Span 使用 10% 采样
);
```

**决策逻辑**:
```
有父 Span？
├── Yes → 继承父 Span 的采样决策
└── No → 使用 root sampler（traceIdRatioBased(0.1)）
```

### 7.5 Span Processor 和 Exporter

#### 7.5.1 SimpleSpanProcessor

```java
// 同步导出（每个 Span 结束立即导出）
SpanProcessor processor = SimpleSpanProcessor.create(
    OtlpGrpcSpanExporter.builder().build()
);
```

**使用场景**: 调试、低吞吐量场景

**缺点**: 阻塞业务线程，性能差

#### 7.5.2 BatchSpanProcessor

```java
// 异步批量导出（推荐）
SpanProcessor processor = BatchSpanProcessor.builder(exporter)
    .setScheduleDelay(Duration.ofSeconds(5))      // 每 5 秒导出
    .setMaxQueueSize(2048)                        // 队列最大 2048
    .setMaxExportBatchSize(512)                   // 每批 512 个
    .setExporterTimeout(Duration.ofSeconds(30))   // 超时 30 秒
    .build();
```

**工作流程**:
```
Span.end()
    ↓
加入队列（非阻塞）
    ↓
后台线程定期导出
    ├── 条件 1: 队列满 (2048)
    ├── 条件 2: 5 秒超时
    └── 条件 3: 应用关闭
    ↓
批量导出（512 个/批）
    ↓
Exporter.export()
```

#### 7.5.3 自定义 SpanProcessor

```java
public class FilteringSpanProcessor implements SpanProcessor {
    private final SpanProcessor delegate;

    public FilteringSpanProcessor(SpanProcessor delegate) {
        this.delegate = delegate;
    }

    @Override
    public void onStart(Context parentContext, ReadWriteSpan span) {
        delegate.onStart(parentContext, span);
    }

    @Override
    public boolean isStartRequired() {
        return delegate.isStartRequired();
    }

    @Override
    public void onEnd(ReadableSpan span) {
        // 过滤：只导出错误 Span
        if (span.getStatus().getStatusCode() == StatusCode.ERROR) {
            delegate.onEnd(span);
        }
    }

    @Override
    public boolean isEndRequired() {
        return true;
    }
}
```

---

## 8. Metrics（指标收集）

### 8.1 Metrics API

#### 8.1.1 Counter（计数器）

**特点**: 单调递增，只能增加不能减少。

```java
// 创建 Counter
LongCounter counter = meter.counterBuilder("http.requests")
    .setDescription("Total HTTP requests")
    .setUnit("1")
    .build();

// 递增
counter.add(1);
counter.add(5, Attributes.of(
    AttributeKey.stringKey("endpoint"), "/api/users",
    AttributeKey.stringKey("method"), "GET"
));
```

**使用场景**:
- 请求总数
- 错误总数
- 消息发送数
- 字节传输总量

#### 8.1.2 Histogram（直方图）

**特点**: 记录值的分布（如响应时间、请求大小）。

```java
// 创建 Histogram
DoubleHistogram histogram = meter.histogramBuilder("http.response.time")
    .setDescription("HTTP response time distribution")
    .setUnit("ms")
    .build();

// 记录值
histogram.record(123.45);
histogram.record(234.56, Attributes.of(
    AttributeKey.stringKey("endpoint"), "/api/users"
));
```

**生成的指标**（Prometheus 格式）:
```
http_response_time_bucket{endpoint="/api/users",le="10"} 50
http_response_time_bucket{endpoint="/api/users",le="50"} 80
http_response_time_bucket{endpoint="/api/users",le="100"} 95
http_response_time_bucket{endpoint="/api/users",le="250"} 100
http_response_time_bucket{endpoint="/api/users",le="+Inf"} 100
http_response_time_sum{endpoint="/api/users"} 12345.67
http_response_time_count{endpoint="/api/users"} 100
```

**使用场景**:
- 响应时间
- 请求大小
- 数据库查询时间
- GC 停顿时间

#### 8.1.3 Gauge（仪表盘）

**特点**: 观测瞬时值（可增可减）。

```java
// ObservableGauge（异步回调）
meter.gaugeBuilder("memory.used")
    .setDescription("Current memory usage")
    .setUnit("bytes")
    .buildWithCallback(measurement -> {
        long memoryUsed = Runtime.getRuntime().totalMemory() - 
                          Runtime.getRuntime().freeMemory();
        measurement.record(memoryUsed);
    });

// ObservableLongGauge
meter.gaugeBuilder("active.connections")
    .ofLongs()
    .setDescription("Current active connections")
    .setUnit("1")
    .buildWithCallback(measurement -> {
        long activeConnections = connectionPool.getActiveCount();
        measurement.record(activeConnections);
    });
```

**使用场景**:
- 内存使用量
- 活跃连接数
- 队列长度
- 温度、CPU 使用率

#### 8.1.4 UpDownCounter

**特点**: 可增可减的计数器。

```java
LongUpDownCounter upDownCounter = meter.upDownCounterBuilder("queue.size")
    .setDescription("Current queue size")
    .setUnit("1")
    .build();

// 增加
upDownCounter.add(10);

// 减少
upDownCounter.add(-5);
```

**使用场景**:
- 队列大小
- 活跃任务数
- 库存数量

### 8.2 聚合策略

#### 8.2.1 Sum Aggregation（求和）

适用于 Counter、UpDownCounter。

```
时间窗口 1: [1, 2, 3, 4, 5] → Sum = 15
时间窗口 2: [6, 7, 8] → Sum = 21
累计: 15 + 21 = 36
```

#### 8.2.2 Histogram Aggregation

适用于 Histogram。

```
记录值: [10, 20, 30, 40, 50, 100, 120, 150]

生成桶:
le="10":   1   (10)
le="50":   5   (10, 20, 30, 40, 50)
le="100":  6   (+ 100)
le="200":  8   (+ 120, 150)
le="+Inf": 8

sum: 520
count: 8
```

#### 8.2.3 LastValue Aggregation

适用于 Gauge。

```
观测值: [100, 105, 103, 110]
LastValue: 110 (最后一个值)
```

### 8.3 时间窗口管理

#### 8.3.1 Delta Temporality（增量）

```
时刻 T0: Counter = 10
时刻 T1: Counter = 15 → 导出 Δ5
时刻 T2: Counter = 23 → 导出 Δ8
```

**适用**: Prometheus（Pull 模式）

#### 8.3.2 Cumulative Temporality（累计）

```
时刻 T0: Counter = 10 → 导出 10
时刻 T1: Counter = 15 → 导出 15
时刻 T2: Counter = 23 → 导出 23
```

**适用**: OTLP（Push 模式）

### 8.4 MetricReader 和 MetricExporter

#### 8.4.1 PeriodicMetricReader（定期导出）

```java
MetricReader reader = PeriodicMetricReader.builder(
    OtlpGrpcMetricExporter.builder().build()
)
.setInterval(Duration.ofSeconds(60))  // 每 60 秒导出
.build();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(reader)
    .build();
```

#### 8.4.2 PrometheusHttpServer（拉取模式）

```java
// 启动 Prometheus HTTP 服务器
PrometheusHttpServer prometheusServer = PrometheusHttpServer.builder()
    .setHost("localhost")
    .setPort(9464)
    .build();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(prometheusServer)
    .build();

// Prometheus 抓取端点: http://localhost:9464/metrics
```

### 8.5 性能考虑

#### 8.5.1 避免高基数属性

```java
// ❌ 错误：user.id 导致高基数（每个用户一个时间序列）
counter.add(1, Attributes.of(
    AttributeKey.stringKey("user.id"), "12345"
));

// ✅ 正确：使用低基数属性
counter.add(1, Attributes.of(
    AttributeKey.stringKey("endpoint"), "/api/users"),
    AttributeKey.stringKey("method"), "GET")
));
```

#### 8.5.2 使用 View 减少基数

```java
SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerView(
        InstrumentSelector.builder()
            .setName("http.requests")
            .build(),
        View.builder()
            .setAttributeFilter(key -> 
                key.equals("endpoint") || key.equals("method")
            )
            .build()
    )
    .build();
```

---

**相关章节**:
- ← 上一节: [3. 目录结构和模块索引](#3-目录结构和模块索引)
- → 下一节: [9. Logs（日志记录）](#9-logs日志记录)
- ↑ 返回目录: [目录](#📑-目录)

## 9. Logs（日志记录）

### 9.1 Logs API 设计

#### 9.1.1 Logger API

**获取 Logger**:
```java
import io.opentelemetry.api.logs.Logger;
import io.opentelemetry.api.logs.LoggerProvider;

Logger logger = openTelemetry.getLoggerProvider()
    .get("instrumentation-library-name");
```

**LogRecord 数据模型**:
```
LogRecord
├── Timestamp (long)              # 日志时间戳
├── Observed Timestamp (long)     # 观测时间戳
├── Trace ID (16 bytes)           # 关联的 Trace ID
├── Span ID (8 bytes)             # 关联的 Span ID
├── Severity (enum)               # 严重级别
│   ├── TRACE
│   ├── DEBUG
│   ├── INFO
│   ├── WARN
│   └── ERROR
├── Severity Text (String)        # 严重级别文本
├── Body (String/JSON)            # 日志内容
├── Attributes (Map)              # 结构化属性
└── Resource (Attributes)         # 资源信息
```

#### 9.1.2 发出日志

**基本日志**:
```java
logger.logRecordBuilder()
    .setSeverity(Severity.INFO)
    .setBody("User logged in")
    .setAttribute("user.id", "12345")
    .setAttribute("action", "login")
    .emit();
```

**带异常的日志**:
```java
try {
    processRequest();
} catch (Exception e) {
    logger.logRecordBuilder()
        .setSeverity(Severity.ERROR)
        .setBody("Request processing failed")
        .setAttribute("error.message", e.getMessage())
        .setAttribute("error.type", e.getClass().getName())
        .emit();
}
```

### 9.2 与日志框架集成

#### 9.2.1 SLF4J 集成

**添加依赖**:
```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-api")
    implementation("io.opentelemetry:opentelemetry-sdk-logs")
    
    // SLF4J 桥接
    implementation("io.opentelemetry.instrumentation:opentelemetry-logback-appender-1.0:1.31.0-alpha")
}
```

**Logback 配置** (`logback.xml`):
```xml
<configuration>
    <appender name="OTEL" class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender">
        <captureExperimentalAttributes>true</captureExperimentalAttributes>
        <captureKeyValuePairAttributes>true</captureKeyValuePairAttributes>
    </appender>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="OTEL" />
        <appender-ref ref="CONSOLE" />
    </root>
</configuration>
```

**使用 SLF4J**:
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class MyService {
    private static final Logger logger = LoggerFactory.getLogger(MyService.class);

    public void processRequest() {
        Span span = tracer.spanBuilder("processRequest").startSpan();
        try (Scope scope = span.makeCurrent()) {
            // SLF4J 日志自动关联到当前 Span
            logger.info("Processing request for user {}", userId);
            
            // 日志会包含 traceId 和 spanId
        } finally {
            span.end();
        }
    }
}
```

#### 9.2.2 Log4j2 集成

**添加依赖**:
```kotlin
dependencies {
    implementation("io.opentelemetry.instrumentation:opentelemetry-log4j-appender-2.17:1.31.0-alpha")
}
```

**Log4j2 配置** (`log4j2.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration packages="io.opentelemetry.instrumentation.log4j.appender.v2_17">
    <Appenders>
        <OpenTelemetry name="OTEL">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
        </OpenTelemetry>
        <Console name="CONSOLE" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="OTEL"/>
            <AppenderRef ref="CONSOLE"/>
        </Root>
    </Loggers>
</Configuration>
```

### 9.3 日志和 Span 关联

#### 9.3.1 自动关联

**工作原理**:
```
当前 Span → Context.current() → Span.current()
    ↓
SLF4J/Log4j2 Appender 读取当前 Span
    ↓
提取 traceId 和 spanId
    ↓
添加到 LogRecord 的 Attributes
    ↓
导出到后端（带 traceId/spanId）
```

**示例**:
```java
Span span = tracer.spanBuilder("processRequest").startSpan();
try (Scope scope = span.makeCurrent()) {
    // 日志自动包含 traceId 和 spanId
    logger.info("Step 1: Validating request");
    
    validateRequest();
    
    logger.info("Step 2: Processing business logic");
    
    processBusinessLogic();
    
    logger.info("Step 3: Sending response");
    
    span.setStatus(StatusCode.OK);
} finally {
    span.end();
}
```

**生成的日志**:
```json
{
  "timestamp": "2024-01-09T10:30:00.000Z",
  "severity": "INFO",
  "body": "Step 1: Validating request",
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "spanId": "b7ad6b7169203331",
  "resource": {
    "service.name": "my-service"
  }
}
```

#### 9.3.2 手动关联

**在无 Span 场景中手动添加 traceId**:
```java
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.TraceId;
import io.opentelemetry.api.trace.SpanId;

public void logWithTraceContext(String traceId, String spanId, String message) {
    logger.logRecordBuilder()
        .setSeverity(Severity.INFO)
        .setBody(message)
        .setAttribute("trace.id", traceId)
        .setAttribute("span.id", spanId)
        .emit();
}
```

### 9.4 LogRecordProcessor 和 Exporter

#### 9.4.1 LogRecordProcessor

**SimpleLogRecordProcessor**（同步）:
```java
import io.opentelemetry.sdk.logs.SdkLoggerProvider;
import io.opentelemetry.sdk.logs.export.SimpleLogRecordProcessor;
import io.opentelemetry.exporter.otlp.logs.OtlpGrpcLogRecordExporter;

LogRecordExporter logExporter = OtlpGrpcLogRecordExporter.builder()
    .setEndpoint("http://localhost:4317")
    .build();

SdkLoggerProvider loggerProvider = SdkLoggerProvider.builder()
    .addLogRecordProcessor(SimpleLogRecordProcessor.create(logExporter))
    .build();
```

**BatchLogRecordProcessor**（异步批处理，推荐）:
```java
import io.opentelemetry.sdk.logs.export.BatchLogRecordProcessor;

LogRecordProcessor processor = BatchLogRecordProcessor.builder(logExporter)
    .setScheduleDelay(Duration.ofSeconds(5))
    .setMaxQueueSize(2048)
    .setMaxExportBatchSize(512)
    .build();

SdkLoggerProvider loggerProvider = SdkLoggerProvider.builder()
    .addLogRecordProcessor(processor)
    .build();
```

#### 9.4.2 LogRecordExporter

**OTLP Exporter**:
```java
// gRPC
LogRecordExporter grpcExporter = OtlpGrpcLogRecordExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();

// HTTP/JSON
LogRecordExporter httpExporter = OtlpHttpLogRecordExporter.builder()
    .setEndpoint("http://localhost:4318/v1/logs")
    .addHeader("Authorization", "Bearer token")
    .build();
```

**Logging Exporter**（调试）:
```java
import io.opentelemetry.exporter.logging.LoggingLogRecordExporter;

LogRecordExporter loggingExporter = LoggingLogRecordExporter.create();

SdkLoggerProvider loggerProvider = SdkLoggerProvider.builder()
    .addLogRecordProcessor(SimpleLogRecordProcessor.create(loggingExporter))
    .build();
```

### 9.5 日志最佳实践

#### 9.5.1 结构化日志

**使用 Attributes 而非字符串拼接**:
```java
// ❌ 不好：字符串拼接
logger.info("User " + userId + " logged in from " + ipAddress);

// ✓ 好：结构化属性
logger.logRecordBuilder()
    .setSeverity(Severity.INFO)
    .setBody("User logged in")
    .setAttribute("user.id", userId)
    .setAttribute("ip.address", ipAddress)
    .setAttribute("user.agent", userAgent)
    .emit();

// 或使用 SLF4J 的 MDC
import org.slf4j.MDC;
MDC.put("user.id", userId);
logger.info("User logged in");
MDC.clear();
```

#### 9.5.2 日志级别管理

**根据环境配置日志级别**:
```java
// 开发环境：DEBUG
// 测试环境：INFO
// 生产环境：WARN

if (environment.isDevelopment()) {
    logger.debug("Detailed processing information: {}", details);
} else {
    logger.info("Processing completed");
}
```

#### 9.5.3 敏感信息脱敏

**避免记录敏感信息**:
```java
// ❌ 错误：记录密码
logger.info("User {} logged in with password {}", username, password);

// ✓ 正确：脱敏或省略
logger.info("User {} logged in", username);

// ✓ 正确：脱敏敏感数据
String maskedCard = cardNumber.replaceAll("\\d(?=\\d{4})", "*");
logger.info("Payment processed for card {}", maskedCard);
```

---

## 10. 导出器架构

### 10.1 导出器接口设计

#### 10.1.1 SpanExporter 接口

**核心接口**:
```java
public interface SpanExporter {
    // 导出 Span 数据
    CompletableResultCode export(Collection<SpanData> spans);

    // 刷新缓冲区
    CompletableResultCode flush();

    // 关闭导出器
    CompletableResultCode shutdown();
}
```

**CompletableResultCode**:
```java
// 异步结果对象
CompletableResultCode result = exporter.export(spans);

// 等待完成
result.join(10, TimeUnit.SECONDS);

// 检查结果
if (result.isSuccess()) {
    System.out.println("Export successful");
} else {
    System.err.println("Export failed");
}
```

#### 10.1.2 MetricExporter 接口

```java
public interface MetricExporter {
    // 导出 Metric 数据
    CompletableResultCode export(Collection<MetricData> metrics);

    // 刷新缓冲区
    CompletableResultCode flush();

    // 关闭导出器
    CompletableResultCode shutdown();

    // 获取聚合时间性（Delta 或 Cumulative）
    AggregationTemporality getAggregationTemporality(InstrumentType instrumentType);
}
```

#### 10.1.3 LogRecordExporter 接口

```java
public interface LogRecordExporter {
    // 导出 LogRecord 数据
    CompletableResultCode export(Collection<LogRecordData> logs);

    // 关闭导出器
    CompletableResultCode shutdown();
}
```

### 10.2 OTLP 协议详解

#### 10.2.1 OTLP 协议概述

**OTLP** (OpenTelemetry Protocol) 是 OpenTelemetry 的原生协议，支持 gRPC 和 HTTP/JSON 两种传输方式。

**协议特点**:
- ✅ 高效的 Protobuf 二进制格式（gRPC）
- ✅ 支持 JSON 格式（HTTP）
- ✅ 支持批量传输
- ✅ 支持压缩（gzip）
- ✅ 支持元数据（headers）
- ✅ 支持重试和超时

**OTLP 架构**:
```
Client Application
    ↓ export
SpanProcessor → BatchSpanProcessor
    ↓ batch
OtlpGrpcSpanExporter / OtlpHttpSpanExporter
    ↓ serialize (Protobuf/JSON)
gRPC/HTTP Client
    ↓ network (4317/4318)
OTLP Receiver (OpenTelemetry Collector)
    ↓ process
Backend (Jaeger/Prometheus/etc.)
```

#### 10.2.2 OTLP gRPC Exporter

**配置 OTLP gRPC Exporter**:
```java
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.exporter.otlp.metrics.OtlpGrpcMetricExporter;
import io.opentelemetry.exporter.otlp.logs.OtlpGrpcLogRecordExporter;

// Traces
SpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")                  // gRPC 端点
    .setTimeout(Duration.ofSeconds(10))                    // 超时时间
    .setCompression("gzip")                                // 压缩
    .addHeader("api-key", "your-api-key")                  // 自定义 header
    .build();

// Metrics
MetricExporter metricExporter = OtlpGrpcMetricExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();

// Logs
LogRecordExporter logExporter = OtlpGrpcLogRecordExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();
```

**环境变量配置**:
```bash
# OTLP gRPC 端点
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 超时时间
export OTEL_EXPORTER_OTLP_TIMEOUT=10000

# 压缩
export OTEL_EXPORTER_OTLP_COMPRESSION=gzip

# Headers
export OTEL_EXPORTER_OTLP_HEADERS=api-key=your-api-key,other-header=value
```

#### 10.2.3 OTLP HTTP Exporter

**配置 OTLP HTTP Exporter**:
```java
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;
import io.opentelemetry.exporter.otlp.http.metrics.OtlpHttpMetricExporter;
import io.opentelemetry.exporter.otlp.http.logs.OtlpHttpLogRecordExporter;

// Traces
SpanExporter spanExporter = OtlpHttpSpanExporter.builder()
    .setEndpoint("http://localhost:4318/v1/traces")        // HTTP 端点
    .setTimeout(Duration.ofSeconds(10))
    .setCompression("gzip")
    .addHeader("Authorization", "Bearer token")
    .build();

// Metrics
MetricExporter metricExporter = OtlpHttpMetricExporter.builder()
    .setEndpoint("http://localhost:4318/v1/metrics")
    .build();

// Logs
LogRecordExporter logExporter = OtlpHttpLogRecordExporter.builder()
    .setEndpoint("http://localhost:4318/v1/logs")
    .build();
```

**环境变量配置**:
```bash
# OTLP HTTP 端点
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# 信号特定端点
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4318/v1/traces
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics
export OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs

# Protocol (http/protobuf, http/json)
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
```

#### 10.2.4 OTLP Protobuf 格式

**Traces Protobuf 定义**（简化）:
```protobuf
message TracesData {
  repeated ResourceSpans resource_spans = 1;
}

message ResourceSpans {
  Resource resource = 1;
  repeated ScopeSpans scope_spans = 2;
}

message Span {
  bytes trace_id = 1;
  bytes span_id = 2;
  bytes parent_span_id = 3;
  string name = 4;
  SpanKind kind = 5;
  fixed64 start_time_unix_nano = 6;
  fixed64 end_time_unix_nano = 7;
  repeated KeyValue attributes = 8;
  repeated Event events = 9;
  repeated Link links = 10;
  Status status = 11;
}
```

### 10.3 Zipkin 导出器

#### 10.3.1 Zipkin 协议

**Zipkin Span 格式**:
```json
{
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "id": "b7ad6b7169203331",
  "parentId": "a7ad6b7169203330",
  "name": "processRequest",
  "timestamp": 1609459200000000,
  "duration": 150000,
  "kind": "SERVER",
  "localEndpoint": {
    "serviceName": "my-service",
    "ipv4": "192.168.1.1",
    "port": 8080
  },
  "tags": {
    "http.method": "POST",
    "http.url": "/api/users",
    "user.id": "12345"
  },
  "annotations": [
    {
      "timestamp": 1609459200050000,
      "value": "Processing started"
    }
  ]
}
```

#### 10.3.2 配置 Zipkin Exporter

```java
import io.opentelemetry.exporter.zipkin.ZipkinSpanExporter;

SpanExporter zipkinExporter = ZipkinSpanExporter.builder()
    .setEndpoint("http://localhost:9411/api/v2/spans")     // Zipkin 端点
    .setTimeout(Duration.ofSeconds(10))
    .build();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(zipkinExporter).build())
    .setResource(Resource.create(
        Attributes.of(
            AttributeKey.stringKey("service.name"), "my-service"
        )
    ))
    .build();
```

**环境变量配置**:
```bash
export OTEL_TRACES_EXPORTER=zipkin
export OTEL_EXPORTER_ZIPKIN_ENDPOINT=http://localhost:9411/api/v2/spans
```

### 10.4 Prometheus 导出器

#### 10.4.1 Prometheus 拉取模式

**Prometheus 导出器特点**:
- ✅ Pull 模式（Prometheus 主动抓取）
- ✅ 暴露 HTTP 端点（默认 `/metrics`）
- ✅ Prometheus 文本格式
- ✅ 支持直方图（Histogram）和摘要（Summary）

**配置 Prometheus Exporter**:
```java
import io.opentelemetry.exporter.prometheus.PrometheusHttpServer;

PrometheusHttpServer prometheusServer = PrometheusHttpServer.builder()
    .setHost("localhost")
    .setPort(9464)
    .build();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(prometheusServer)
    .build();

// Prometheus 抓取端点: http://localhost:9464/metrics
```

#### 10.4.2 Prometheus 指标格式

**Counter 示例**:
```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{endpoint="/api/users",method="GET"} 100
http_requests_total{endpoint="/api/users",method="POST"} 50
```

**Histogram 示例**:
```
# HELP http_response_time_seconds HTTP response time distribution
# TYPE http_response_time_seconds histogram
http_response_time_seconds_bucket{endpoint="/api/users",le="0.01"} 50
http_response_time_seconds_bucket{endpoint="/api/users",le="0.05"} 80
http_response_time_seconds_bucket{endpoint="/api/users",le="0.1"} 95
http_response_time_seconds_bucket{endpoint="/api/users",le="+Inf"} 100
http_response_time_seconds_sum{endpoint="/api/users"} 3.5
http_response_time_seconds_count{endpoint="/api/users"} 100
```

**Gauge 示例**:
```
# HELP memory_used_bytes Current memory usage
# TYPE memory_used_bytes gauge
memory_used_bytes 1073741824
```

### 10.5 批处理和重试策略

#### 10.5.1 BatchSpanProcessor 配置

```java
SpanProcessor processor = BatchSpanProcessor.builder(exporter)
    .setScheduleDelay(Duration.ofSeconds(5))              // 批处理延迟
    .setMaxQueueSize(2048)                                // 队列最大大小
    .setMaxExportBatchSize(512)                           // 每批最大 Span 数
    .setExporterTimeout(Duration.ofSeconds(30))           // 导出超时
    .build();
```

**批处理流程**:
```
Span.end()
    ↓
加入队列（非阻塞）
    ↓
触发批处理（满足以下任一条件）:
    ├── 队列大小 >= maxExportBatchSize (512)
    ├── 距离上次导出 >= scheduleDelay (5 秒)
    └── 应用关闭（shutdown）
    ↓
从队列取出 maxExportBatchSize 个 Span
    ↓
调用 Exporter.export(spans)
    ↓
等待结果（最多 exporterTimeout 秒）
    ↓
成功: 清除已导出的 Span
失败: 丢弃（或记录错误）
```

#### 10.5.2 OTLP 重试策略

**内置重试机制**:
```java
OtlpGrpcSpanExporter exporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .setRetryPolicy(RetryPolicy.getDefault())             // 默认重试策略
    .build();
```

**默认重试策略**:
```
失败原因                  重试策略
─────────────────────────────────────────────
网络错误（连接失败）       → 重试（指数退避）
5xx 服务器错误             → 重试（指数退避）
4xx 客户端错误             → 不重试
超时                       → 重试（指数退避）
成功                       → 不重试
```

**指数退避算法**:
```
重试次数  延迟时间
─────────────────
1         1 秒
2         2 秒
3         4 秒
4         8 秒
5         16 秒
最大重试: 5 次
```

### 10.6 自定义导出器开发

#### 10.6.1 实现自定义 SpanExporter

```java
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;

public class CustomSpanExporter implements SpanExporter {

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        // 自定义导出逻辑
        try {
            for (SpanData span : spans) {
                // 转换为自定义格式
                String json = convertToJson(span);
                
                // 发送到自定义后端
                sendToBackend(json);
                
                // 记录日志
                System.out.println("Exported span: " + span.getName());
            }
            return CompletableResultCode.ofSuccess();
        } catch (Exception e) {
            System.err.println("Export failed: " + e.getMessage());
            return CompletableResultCode.ofFailure();
        }
    }

    @Override
    public CompletableResultCode flush() {
        // 刷新缓冲区（如果有）
        System.out.println("Flushing exporter");
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        // 关闭连接、释放资源
        System.out.println("Shutting down exporter");
        return CompletableResultCode.ofSuccess();
    }

    private String convertToJson(SpanData span) {
        // 转换为 JSON 格式
        return String.format(
            "{\"traceId\":\"%s\",\"spanId\":\"%s\",\"name\":\"%s\"}",
            span.getTraceId(),
            span.getSpanId(),
            span.getName()
        );
    }

    private void sendToBackend(String json) throws Exception {
        // 发送到自定义后端（HTTP、Kafka、数据库等）
        // 示例：HTTP POST
        // HttpClient client = HttpClient.newHttpClient();
        // HttpRequest request = HttpRequest.newBuilder()
        //     .uri(URI.create("https://my-backend.com/spans"))
        //     .header("Content-Type", "application/json")
        //     .POST(HttpRequest.BodyPublishers.ofString(json))
        //     .build();
        // client.send(request, HttpResponse.BodyHandlers.ofString());
    }
}
```

#### 10.6.2 注册自定义导出器

```java
SpanExporter customExporter = new CustomSpanExporter();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(customExporter).build())
    .build();

OpenTelemetry openTelemetry = OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    .buildAndRegisterGlobal();
```

#### 10.6.3 自定义导出器最佳实践

**1. 实现幂等性**:
```java
// ✓ 好：使用唯一 ID 避免重复
private final Set<String> exportedSpanIds = new ConcurrentHashMap<String, Boolean>().newKeySet();

@Override
public CompletableResultCode export(Collection<SpanData> spans) {
    for (SpanData span : spans) {
        String spanId = span.getSpanId();
        if (exportedSpanIds.contains(spanId)) {
            continue;  // 跳过已导出的 Span
        }
        sendToBackend(span);
        exportedSpanIds.add(spanId);
    }
    return CompletableResultCode.ofSuccess();
}
```

**2. 实现异步导出**:
```java
private final ExecutorService executor = Executors.newFixedThreadPool(4);

@Override
public CompletableResultCode export(Collection<SpanData> spans) {
    CompletableResultCode result = new CompletableResultCode();
    
    executor.submit(() -> {
        try {
            for (SpanData span : spans) {
                sendToBackend(span);
            }
            result.succeed();
        } catch (Exception e) {
            result.fail();
        }
    });
    
    return result;
}

@Override
public CompletableResultCode shutdown() {
    executor.shutdown();
    try {
        executor.awaitTermination(30, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        executor.shutdownNow();
    }
    return CompletableResultCode.ofSuccess();
}
```

**3. 实现连接池**:
```java
// 使用连接池提高性能
private final HttpClient httpClient = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .executor(Executors.newFixedThreadPool(10))
    .build();
```

**4. 实现错误处理和重试**:
```java
@Override
public CompletableResultCode export(Collection<SpanData> spans) {
    int maxRetries = 3;
    int retryCount = 0;
    
    while (retryCount < maxRetries) {
        try {
            sendToBackend(spans);
            return CompletableResultCode.ofSuccess();
        } catch (Exception e) {
            retryCount++;
            if (retryCount >= maxRetries) {
                System.err.println("Export failed after " + maxRetries + " retries");
                return CompletableResultCode.ofFailure();
            }
            
            // 指数退避
            long delay = (long) Math.pow(2, retryCount) * 1000;
            try {
                Thread.sleep(delay);
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                return CompletableResultCode.ofFailure();
            }
        }
    }
    
    return CompletableResultCode.ofFailure();
}
```

---

**相关章节**:
- ← 上一节: [8. Metrics（指标收集)](#8-metrics指标收集)
- → 下一节: [11. 构建系统（buildSrc）](#11-构建系统buildsrc)
- ↑ 返回目录: [目录](#📑-目录)

---

# 第四部分：核心组件

## 19. Context 模块详解

### 19.1 Context 模块概述

**Context 模块**是 OpenTelemetry Java 的基础设施层，提供了跨 API 边界和线程传播作用域值的机制。它是整个 SDK 的基石，支撑着 Span、Baggage 等上下文信息的传播。

**模块位置**: `context/`

**核心特性**:
- ✅ 不可变的 Context 对象
- ✅ ThreadLocal 存储机制
- ✅ 作用域（Scope）管理
- ✅ 跨线程传播支持
- ✅ 严格模式调试
- ✅ 无依赖（零依赖模块）

**架构图**:
```
User Code
    ↓ 使用
Context API (不可变)
    ├── Context.current()
    ├── Context.with(key, value)
    └── Context.makeCurrent()
    ↓ 存储
ContextStorage (SPI)
    ├── ThreadLocalContextStorage (默认)
    ├── StrictContextStorage (调试)
    └── 自定义实现
    ↓ 底层
ThreadLocal<Context>
```

### 19.2 Context API 详解

#### 19.2.1 核心接口

**Context 接口**:
```java
public interface Context {
    // 获取当前 Context
    static Context current();

    // 获取根 Context
    static Context root();

    // 创建新 Context（添加键值对）
    <V> Context with(ContextKey<V> key, V value);

    // 获取值
    <V> V get(ContextKey<V> key);

    // 使 Context 成为当前（返回 Scope）
    @MustBeClosed
    Scope makeCurrent();

    // 包装 Runnable（自动传播 Context）
    Runnable wrap(Runnable runnable);

    // 包装 Callable
    <T> Callable<T> wrap(Callable<T> callable);

    // 包装 Executor
    Executor wrap(Executor executor);

    // 包装 ExecutorService
    ExecutorService wrap(ExecutorService executorService);
}
```

**Scope 接口**:
```java
public interface Scope extends AutoCloseable {
    @Override
    void close();
}
```

#### 19.2.2 ContextKey 使用

**创建 ContextKey**:
```java
import io.opentelemetry.context.Context;
import io.opentelemetry.context.ContextKey;

public class MyService {
    // 创建 ContextKey（通常为静态常量）
    private static final ContextKey<String> USER_ID_KEY =
        ContextKey.named("user.id");

    private static final ContextKey<String> REQUEST_ID_KEY =
        ContextKey.named("request.id");

    public void processRequest(String userId, String requestId) {
        // 存储值到 Context
        Context context = Context.current()
            .with(USER_ID_KEY, userId)
            .with(REQUEST_ID_KEY, requestId);

        // 使 Context 成为当前
        try (Scope scope = context.makeCurrent()) {
            // 在此作用域内，所有代码都可以访问这些值
            doWork();
        }
    }

    private void doWork() {
        // 获取当前 Context 的值
        String userId = USER_ID_KEY.get();
        String requestId = REQUEST_ID_KEY.get();

        System.out.println("Processing request " + requestId +
                          " for user " + userId);
    }
}
```

#### 19.2.3 Context 不可变性

**Context 是不可变的**，每次调用 `with()` 都会创建新的 Context 对象：

```java
Context parent = Context.current();
Context child = parent.with(USER_ID_KEY, "12345");

// parent 和 child 是不同的对象
assert parent != child;

// parent 不包含 USER_ID_KEY
assert parent.get(USER_ID_KEY) == null;

// child 包含 USER_ID_KEY
assert "12345".equals(child.get(USER_ID_KEY));
```

**优势**:
- ✅ 线程安全（无需同步）
- ✅ 可以安全地跨线程传递
- ✅ 避免意外修改

### 19.3 ContextStorage 实现

#### 19.3.1 ThreadLocalContextStorage（默认）

**实现原理**:
```java
public final class ThreadLocalContextStorage implements ContextStorage {
    private static final ThreadLocal<Context> THREAD_LOCAL_CONTEXT =
        new ThreadLocal<Context>() {
            @Override
            protected Context initialValue() {
                return ArrayBasedContext.root();
            }
        };

    @Override
    public Context current() {
        return THREAD_LOCAL_CONTEXT.get();
    }

    @Override
    public Scope attach(Context toAttach) {
        Context current = current();
        THREAD_LOCAL_CONTEXT.set(toAttach);

        return () -> {
            THREAD_LOCAL_CONTEXT.set(current);  // 恢复之前的 Context
        };
    }
}
```

**特点**:
- ✅ 基于 ThreadLocal，每个线程独立的 Context
- ✅ 高性能（无锁）
- ✅ 自动隔离不同线程的 Context

**使用示例**:
```java
// 在主线程中
Context context = Context.current().with(USER_ID_KEY, "12345");

try (Scope scope = context.makeCurrent()) {
    // 主线程可以访问
    assert "12345".equals(USER_ID_KEY.get());

    // 新线程无法访问（ThreadLocal 隔离）
    new Thread(() -> {
        assert USER_ID_KEY.get() == null;  // 新线程的 Context 是独立的
    }).start();
}
```

#### 19.3.2 StrictContextStorage（调试模式）

**启用严格模式**:
```bash
java -Dio.opentelemetry.context.enableStrictContext=true YourApp
```

**严格模式检查**:
1. **Scope 必须在创建它的线程中关闭**
2. **Scope 不能被垃圾回收前未关闭**
3. **检测 Kotlin 协程中的错误使用**

**示例 - 检测跨线程关闭错误**:
```java
Context context = Context.current().with(USER_ID_KEY, "12345");
Scope scope = context.makeCurrent();

// ❌ 错误：在不同线程中关闭
new Thread(() -> {
    scope.close();  // 抛出异常：Scope closed on different thread!
}).start();
```

**示例 - 检测未关闭的 Scope**:
```java
// ❌ 错误：忘记关闭 Scope
void badMethod() {
    Context context = Context.current().with(USER_ID_KEY, "12345");
    Scope scope = context.makeCurrent();

    // 忘记 scope.close()
    // 严格模式会在 GC 时打印警告
}
```

#### 19.3.3 自定义 ContextStorage

**实现自定义存储**:
```java
import io.opentelemetry.context.Context;
import io.opentelemetry.context.ContextStorage;
import io.opentelemetry.context.Scope;

public class CustomContextStorage implements ContextStorage {
    // 自定义存储机制（如 InheritableThreadLocal、协程上下文等）
    private static final InheritableThreadLocal<Context> CONTEXT =
        new InheritableThreadLocal<Context>() {
            @Override
            protected Context initialValue() {
                return Context.root();
            }
        };

    @Override
    public Context current() {
        return CONTEXT.get();
    }

    @Override
    public Scope attach(Context toAttach) {
        Context previous = current();
        CONTEXT.set(toAttach);

        return () -> CONTEXT.set(previous);
    }
}
```

**注册自定义存储**:

通过 SPI 机制，创建文件 `META-INF/services/io.opentelemetry.context.ContextStorageProvider`:
```
com.example.CustomContextStorageProvider
```

**Provider 实现**:
```java
public class CustomContextStorageProvider implements ContextStorageProvider {
    @Override
    public ContextStorage get() {
        return new CustomContextStorage();
    }
}
```

### 19.4 跨线程传播

#### 19.4.1 Context.wrap() 方法

**自动传播 Context**:
```java
Context context = Context.current().with(USER_ID_KEY, "12345");

// 包装 Runnable
Runnable task = context.wrap(() -> {
    // 自动在正确的 Context 中执行
    assert "12345".equals(USER_ID_KEY.get());
    System.out.println("Task executed with user " + USER_ID_KEY.get());
});

// 在新线程中执行
new Thread(task).start();
```

**包装 Callable**:
```java
Callable<String> task = context.wrap(() -> {
    return "Result for user " + USER_ID_KEY.get();
});

String result = executorService.submit(task).get();
```

**包装 Executor**:
```java
Executor wrappedExecutor = context.wrap(executor);

// 所有提交的任务都会在正确的 Context 中执行
wrappedExecutor.execute(() -> {
    assert "12345".equals(USER_ID_KEY.get());
});
```

#### 19.4.2 手动传播

**手动捕获和恢复**:
```java
// 在父线程中捕获
Context context = Context.current();

// 在子线程中恢复
new Thread(() -> {
    try (Scope scope = context.makeCurrent()) {
        // 现在可以访问父线程的 Context
        doWork();
    }
}).start();
```

#### 19.4.3 CompletableFuture 传播

**使用 Context.wrap()**:
```java
Context context = Context.current().with(USER_ID_KEY, "12345");

CompletableFuture.supplyAsync(
    context.wrap(() -> {
        return "User: " + USER_ID_KEY.get();
    })
).thenAccept(result -> {
    System.out.println(result);
});
```

**使用自定义 Executor**:
```java
Executor contextAwareExecutor = Context.taskWrapping(
    Executors.newFixedThreadPool(10)
);

CompletableFuture.supplyAsync(() -> {
    // Context 自动传播
    return USER_ID_KEY.get();
}, contextAwareExecutor);
```

### 19.5 Context 与 Span 的集成

#### 19.5.1 Span 存储在 Context 中

**Span 使用 Context 传播**:
```java
// Span 被存储在 Context 中
Span span = tracer.spanBuilder("operation").startSpan();

// makeCurrent() 将 Span 放入 Context
try (Scope scope = span.makeCurrent()) {
    // 获取当前 Span
    Span currentSpan = Span.current();
    assert currentSpan == span;

    // 创建子 Span（自动继承父 Context）
    Span childSpan = tracer.spanBuilder("child").startSpan();
    try (Scope childScope = childSpan.makeCurrent()) {
        // childSpan 的父 Span 是 span
    } finally {
        childSpan.end();
    }
} finally {
    span.end();
}
```

#### 19.5.2 Context 传播链路

```
Thread 1
└── Context (root)
    └── Context (userId=12345)
        └── Context (+ Span A)
            └── Context (+ Span B)
                └── 传播到 Thread 2
                    └── Context (userId=12345, Span B)
                        └── Context (+ Span C)
```

### 19.6 最佳实践

#### 19.6.1 始终使用 try-with-resources

```java
// ✅ 正确：自动关闭 Scope
try (Scope scope = context.makeCurrent()) {
    doWork();
}

// ❌ 错误：忘记关闭
Scope scope = context.makeCurrent();
doWork();
// Scope 泄漏！
```

#### 19.6.2 避免在 Context 中存储大对象

```java
// ❌ 不好：存储大对象
Context context = Context.current().with(LARGE_DATA_KEY, hugeObject);

// ✅ 好：只存储引用或 ID
Context context = Context.current().with(DATA_ID_KEY, "data-123");
```

#### 19.6.3 使用明确的 ContextKey 命名

```java
// ✅ 好：清晰的命名
private static final ContextKey<String> USER_ID_KEY =
    ContextKey.named("user.id");

// ❌ 不好：模糊的命名
private static final ContextKey<String> KEY =
    ContextKey.named("k");
```

#### 19.6.4 在单元测试中启用严格模式

```java
// JUnit 5
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class MyTest {
    @BeforeAll
    void setUp() {
        System.setProperty("io.opentelemetry.context.enableStrictContext", "true");
    }
}
```

### 19.7 性能考虑

#### 19.7.1 Context 创建成本

**Context 是轻量级的**:
- 使用数组存储（ArrayBasedContext）
- 写时复制（Copy-on-Write）
- 共享不可变数据

**基准测试结果**:
```
Context.current():              ~5 ns
Context.with():                 ~20 ns
Context.makeCurrent():          ~10 ns
Scope.close():                  ~10 ns
```

#### 19.7.2 优化建议

**1. 减少 Context 层级**:
```java
// ❌ 不好：多层嵌套
Context c1 = Context.current().with(KEY1, val1);
Context c2 = c1.with(KEY2, val2);
Context c3 = c2.with(KEY3, val3);

// ✅ 好：批量设置（如果可能）
Context context = Context.current()
    .with(KEY1, val1)
    .with(KEY2, val2)
    .with(KEY3, val3);
```

**2. 重用 Context**:
```java
// 如果 Context 不变，重用它
private final Context baseContext = Context.root()
    .with(SERVICE_NAME_KEY, "my-service");

public void handleRequest() {
    try (Scope scope = baseContext.makeCurrent()) {
        // ...
    }
}
```

### 19.8 调试和故障排查

#### 19.8.1 启用调试日志

```bash
java -Djava.util.logging.config.file=logging.properties \
     -Dio.opentelemetry.context.enableStrictContext=true \
     YourApp
```

**logging.properties**:
```properties
io.opentelemetry.context.level=FINE
```

#### 19.8.2 常见问题

**问题 1: Context 丢失**

```java
// 原因：忘记 makeCurrent()
Context context = Context.current().with(USER_ID_KEY, "12345");

// ❌ 错误：没有使 Context 成为当前
doWork();  // USER_ID_KEY.get() 返回 null

// ✅ 正确
try (Scope scope = context.makeCurrent()) {
    doWork();  // USER_ID_KEY.get() 返回 "12345"
}
```

**问题 2: 跨线程 Context 丢失**

```java
// ❌ 错误：ThreadLocal 不会自动传播
new Thread(() -> {
    assert USER_ID_KEY.get() == null;  // Context 丢失
}).start();

// ✅ 正确：手动传播
Context context = Context.current();
new Thread(() -> {
    try (Scope scope = context.makeCurrent()) {
        assert USER_ID_KEY.get() != null;  // Context 正确传播
    }
}).start();
```

**问题 3: Scope 泄漏**

```java
// ❌ 错误：异常导致 Scope 未关闭
Scope scope = context.makeCurrent();
doWork();  // 如果抛异常，scope.close() 不会执行
scope.close();

// ✅ 正确：使用 try-finally
Scope scope = context.makeCurrent();
try {
    doWork();
} finally {
    scope.close();
}

// ✅ 更好：使用 try-with-resources
try (Scope scope = context.makeCurrent()) {
    doWork();
}
```

---

## 20. Semconv（语义约定）

### 20.1 语义约定概述

**Semantic Conventions（语义约定）** 定义了 OpenTelemetry 中标准的属性名称、度量名称和枚举值，确保不同语言实现和不同供应商之间的互操作性。

**模块位置**: `api/incubator` (语义约定现在是 API 的一部分)

**核心目标**:
- ✅ 统一命名规范
- ✅ 跨语言一致性
- ✅ 后端兼容性
- ✅ 可搜索性和可分析性

### 20.2 语义约定分类

#### 20.2.1 Resource 语义约定

**服务相关**:
```java
import io.opentelemetry.semconv.ResourceAttributes;

Resource resource = Resource.create(
    Attributes.of(
        // 服务名称（必需）
        ResourceAttributes.SERVICE_NAME, "my-service",

        // 服务版本
        ResourceAttributes.SERVICE_VERSION, "1.0.0",

        // 服务命名空间
        ResourceAttributes.SERVICE_NAMESPACE, "production",

        // 服务实例 ID
        ResourceAttributes.SERVICE_INSTANCE_ID, "instance-123"
    )
);
```

**部署环境**:
```java
Attributes.of(
    // 部署环境
    ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production",

    // 云提供商
    ResourceAttributes.CLOUD_PROVIDER, "aws",

    // 云区域
    ResourceAttributes.CLOUD_REGION, "us-east-1",

    // 云账户 ID
    ResourceAttributes.CLOUD_ACCOUNT_ID, "123456789012"
)
```

**主机信息**:
```java
Attributes.of(
    // 主机名
    ResourceAttributes.HOST_NAME, "server-01",

    // 主机类型
    ResourceAttributes.HOST_TYPE, "m5.large",

    // 操作系统
    ResourceAttributes.OS_TYPE, "linux",

    // OS 版本
    ResourceAttributes.OS_VERSION, "Ubuntu 20.04"
)
```

**容器信息**:
```java
Attributes.of(
    // 容器名称
    ResourceAttributes.CONTAINER_NAME, "my-app",

    // 容器 ID
    ResourceAttributes.CONTAINER_ID, "abc123",

    // 容器镜像名称
    ResourceAttributes.CONTAINER_IMAGE_NAME, "my-app",

    // 容器镜像标签
    ResourceAttributes.CONTAINER_IMAGE_TAG, "v1.0.0"
)
```

**Kubernetes 信息**:
```java
Attributes.of(
    // K8s 集群名称
    ResourceAttributes.K8S_CLUSTER_NAME, "prod-cluster",

    // K8s 命名空间
    ResourceAttributes.K8S_NAMESPACE_NAME, "default",

    // K8s Pod 名称
    ResourceAttributes.K8S_POD_NAME, "my-app-pod-123",

    // K8s Deployment 名称
    ResourceAttributes.K8S_DEPLOYMENT_NAME, "my-app"
)
```

#### 20.2.2 Trace 语义约定

**HTTP 客户端**:
```java
import io.opentelemetry.semconv.SemanticAttributes;

Span span = tracer.spanBuilder("GET /users")
    .setSpanKind(SpanKind.CLIENT)
    .startSpan();

span.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
span.setAttribute(SemanticAttributes.HTTP_URL, "https://api.example.com/users");
span.setAttribute(SemanticAttributes.HTTP_TARGET, "/users");
span.setAttribute(SemanticAttributes.HTTP_SCHEME, "https");
span.setAttribute(SemanticAttributes.HTTP_HOST, "api.example.com");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);
span.setAttribute(SemanticAttributes.HTTP_REQUEST_CONTENT_LENGTH, 1024L);
span.setAttribute(SemanticAttributes.HTTP_RESPONSE_CONTENT_LENGTH, 2048L);
```

**HTTP 服务端**:
```java
Span span = tracer.spanBuilder("GET /api/users")
    .setSpanKind(SpanKind.SERVER)
    .startSpan();

span.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
span.setAttribute(SemanticAttributes.HTTP_TARGET, "/api/users");
span.setAttribute(SemanticAttributes.HTTP_ROUTE, "/api/users/:id");
span.setAttribute(SemanticAttributes.HTTP_SCHEME, "https");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);
span.setAttribute(SemanticAttributes.HTTP_CLIENT_IP, "192.168.1.1");
span.setAttribute(SemanticAttributes.HTTP_USER_AGENT, "Mozilla/5.0...");
```

**数据库操作**:
```java
Span span = tracer.spanBuilder("SELECT users")
    .setSpanKind(SpanKind.CLIENT)
    .startSpan();

span.setAttribute(SemanticAttributes.DB_SYSTEM, "postgresql");
span.setAttribute(SemanticAttributes.DB_NAME, "mydb");
span.setAttribute(SemanticAttributes.DB_USER, "dbuser");
span.setAttribute(SemanticAttributes.DB_CONNECTION_STRING, "postgresql://localhost:5432/mydb");
span.setAttribute(SemanticAttributes.DB_STATEMENT, "SELECT * FROM users WHERE id = ?");
span.setAttribute(SemanticAttributes.DB_OPERATION, "SELECT");
span.setAttribute(SemanticAttributes.DB_SQL_TABLE, "users");
```

**RPC 调用**:
```java
Span span = tracer.spanBuilder("grpc.UserService/GetUser")
    .setSpanKind(SpanKind.CLIENT)
    .startSpan();

span.setAttribute(SemanticAttributes.RPC_SYSTEM, "grpc");
span.setAttribute(SemanticAttributes.RPC_SERVICE, "UserService");
span.setAttribute(SemanticAttributes.RPC_METHOD, "GetUser");
span.setAttribute(SemanticAttributes.NET_PEER_NAME, "api.example.com");
span.setAttribute(SemanticAttributes.NET_PEER_PORT, 50051);
```

**消息队列**:
```java
// 生产者
Span span = tracer.spanBuilder("send user.created")
    .setSpanKind(SpanKind.PRODUCER)
    .startSpan();

span.setAttribute(SemanticAttributes.MESSAGING_SYSTEM, "kafka");
span.setAttribute(SemanticAttributes.MESSAGING_DESTINATION, "user.created");
span.setAttribute(SemanticAttributes.MESSAGING_DESTINATION_KIND, "topic");
span.setAttribute(SemanticAttributes.MESSAGING_MESSAGE_ID, "msg-123");
span.setAttribute(SemanticAttributes.MESSAGING_KAFKA_PARTITION, 0);

// 消费者
Span span = tracer.spanBuilder("process user.created")
    .setSpanKind(SpanKind.CONSUMER)
    .startSpan();

span.setAttribute(SemanticAttributes.MESSAGING_SYSTEM, "kafka");
span.setAttribute(SemanticAttributes.MESSAGING_DESTINATION, "user.created");
span.setAttribute(SemanticAttributes.MESSAGING_OPERATION, "process");
span.setAttribute(SemanticAttributes.MESSAGING_CONSUMER_ID, "consumer-group-1");
```

#### 20.2.3 Metric 语义约定

**HTTP 服务器指标**:
```java
// 请求持续时间
DoubleHistogram requestDuration = meter.histogramBuilder("http.server.duration")
    .setUnit("ms")
    .setDescription("HTTP server request duration")
    .build();

requestDuration.record(123.45,
    Attributes.of(
        SemanticAttributes.HTTP_METHOD, "GET",
        SemanticAttributes.HTTP_ROUTE, "/api/users",
        SemanticAttributes.HTTP_STATUS_CODE, 200
    )
);

// 活跃请求数
LongUpDownCounter activeRequests = meter.upDownCounterBuilder("http.server.active_requests")
    .setUnit("1")
    .setDescription("Number of active HTTP requests")
    .build();
```

**系统指标**:
```java
// CPU 使用率
meter.gaugeBuilder("system.cpu.utilization")
    .setUnit("1")
    .setDescription("CPU utilization")
    .buildWithCallback(measurement -> {
        measurement.record(getCpuUsage(),
            Attributes.of(
                SemanticAttributes.CPU_STATE, "user"
            )
        );
    });

// 内存使用
meter.gaugeBuilder("system.memory.usage")
    .ofLongs()
    .setUnit("bytes")
    .setDescription("Memory usage")
    .buildWithCallback(measurement -> {
        measurement.record(getMemoryUsage(),
            Attributes.of(
                SemanticAttributes.MEMORY_STATE, "used"
            )
        );
    });
```

### 20.3 命名规范

#### 20.3.1 属性命名规则

**规则**:
1. 使用小写字母和点分隔符
2. 使用命名空间（如 `http.`, `db.`, `net.`）
3. 避免缩写（除非是通用的，如 `http`, `db`）
4. 使用单数形式（除非语义上是复数）

**示例**:
```java
// ✅ 好
SemanticAttributes.HTTP_METHOD
SemanticAttributes.DB_NAME
SemanticAttributes.NET_PEER_NAME

// ❌ 不好
"httpMethod"  // 驼峰命名
"database"    // 没有命名空间
"net_peer"    // 下划线
```

#### 20.3.2 度量命名规则

**规则**:
1. 使用点分隔符
2. 包含单位（如 `.duration`, `.size`）
3. 使用复数形式（如 `.requests`, `.connections`）

**示例**:
```java
// ✅ 好
"http.server.duration"
"system.cpu.utilization"
"process.runtime.jvm.memory.used"

// ❌ 不好
"httpServerDuration"  // 驼峰命名
"cpu"                 // 不够具体
"requests"            // 缺少命名空间
```

### 20.4 使用建议

#### 20.4.1 优先使用标准属性

```java
// ✅ 好：使用标准属性
span.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);

// ❌ 不好：自定义属性名称
span.setAttribute("method", "GET");
span.setAttribute("status", 200);
```

#### 20.4.2 自定义属性使用命名空间

```java
// ✅ 好：使用自己的命名空间
AttributeKey<String> CUSTOM_USER_TIER =
    AttributeKey.stringKey("myapp.user.tier");

span.setAttribute(CUSTOM_USER_TIER, "premium");

// ❌ 不好：污染全局命名空间
span.setAttribute("tier", "premium");
```

#### 20.4.3 保持属性一致性

```java
// ✅ 好：所有 HTTP 请求使用相同的属性
void recordHttpRequest(Span span, HttpRequest request) {
    span.setAttribute(SemanticAttributes.HTTP_METHOD, request.getMethod());
    span.setAttribute(SemanticAttributes.HTTP_URL, request.getUrl());
    span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, request.getStatusCode());
}

// ❌ 不好：不同地方使用不同属性
span.setAttribute("method", "GET");  // 某处
span.setAttribute(SemanticAttributes.HTTP_METHOD, "POST");  // 另一处
```

### 20.5 版本管理

**语义约定版本**:
- 语义约定独立版本控制
- 向后兼容的变更：Minor 版本更新
- 破坏性变更：Major 版本更新

**检查当前版本**:
```java
// 语义约定版本通常在 ResourceAttributes 或 SemanticAttributes 类的注释中
```

### 20.6 参考资源

**官方文档**:
- OpenTelemetry 语义约定规范: https://opentelemetry.io/docs/specs/semconv/
- HTTP 语义约定: https://opentelemetry.io/docs/specs/semconv/http/
- 数据库语义约定: https://opentelemetry.io/docs/specs/semconv/database/
- RPC 语义约定: https://opentelemetry.io/docs/specs/semconv/rpc/

---

## 21. 导出器详解

### 21.1 导出器概述

**导出器（Exporter）** 负责将遥测数据发送到后端系统。OpenTelemetry Java 提供了多种导出器实现，支持不同的协议和后端。

**模块位置**: `exporters/`

**导出器分类**:
```
exporters/
├── otlp/                   # OTLP 协议（推荐）
│   ├── all/                # 聚合所有 OTLP 导出器
│   ├── common/             # OTLP 通用组件
│   └── profiles/           # OTLP Profiles 支持
├── zipkin/                 # Zipkin 格式
├── prometheus/             # Prometheus 格式
├── logging/                # 日志输出（调试）
└── logging-otlp/           # OTLP 格式日志
```

### 21.2 OTLP 导出器详解

#### 21.2.1 OTLP 协议概述

**OTLP (OpenTelemetry Protocol)** 是 OpenTelemetry 的原生协议，具有以下特点：
- ✅ 支持 gRPC 和 HTTP 传输
- ✅ Protobuf 和 JSON 编码
- ✅ 批量传输和压缩
- ✅ 支持所有信号（Traces、Metrics、Logs）
- ✅ 高性能和低开销

**协议端点**:
```
gRPC:  http://localhost:4317
HTTP:  http://localhost:4318
  ├── /v1/traces
  ├── /v1/metrics
  └── /v1/logs
```

#### 21.2.2 OTLP gRPC 导出器

**添加依赖**:
```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}
```

**配置 Traces 导出**:
```java
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.trace.export.SpanExporter;

SpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .setCompression("gzip")
    .addHeader("api-key", "your-api-key")
    .build();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
    .build();
```

**配置 Metrics 导出**:
```java
import io.opentelemetry.exporter.otlp.metrics.OtlpGrpcMetricExporter;
import io.opentelemetry.sdk.metrics.export.MetricExporter;

MetricExporter metricExporter = OtlpGrpcMetricExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(
        PeriodicMetricReader.builder(metricExporter)
            .setInterval(Duration.ofSeconds(60))
            .build()
    )
    .build();
```

**配置 Logs 导出**:
```java
import io.opentelemetry.exporter.otlp.logs.OtlpGrpcLogRecordExporter;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;

LogRecordExporter logExporter = OtlpGrpcLogRecordExporter.builder()
    .setEndpoint("http://localhost:4317")
    .setTimeout(Duration.ofSeconds(10))
    .build();

SdkLoggerProvider loggerProvider = SdkLoggerProvider.builder()
    .addLogRecordProcessor(
        BatchLogRecordProcessor.builder(logExporter).build()
    )
    .build();
```

#### 21.2.3 OTLP HTTP 导出器

**配置 HTTP 导出**:
```java
import io.opentelemetry.exporter.otlp.http.trace.OtlpHttpSpanExporter;

SpanExporter spanExporter = OtlpHttpSpanExporter.builder()
    .setEndpoint("http://localhost:4318/v1/traces")
    .setTimeout(Duration.ofSeconds(10))
    .setCompression("gzip")
    .addHeader("Authorization", "Bearer token")
    .build();
```

**HTTP vs gRPC 对比**:

| 特性 | gRPC | HTTP |
|------|------|------|
| 性能 | 更高（二进制协议） | 较低（文本协议） |
| 压缩 | 内置支持 | 需手动配置 |
| 流式传输 | 支持 | 不支持 |
| 防火墙友好 | 较差（非标准端口） | 更好（HTTP/HTTPS） |
| 调试 | 较困难 | 更容易（可读格式） |

**选择建议**:
- **生产环境**: 优先使用 gRPC（高性能）
- **防火墙限制**: 使用 HTTP
- **调试**: 使用 HTTP + JSON 格式

#### 21.2.4 OTLP 环境变量配置

**通用配置**:
```bash
# OTLP 端点（gRPC 或 HTTP）
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 协议（grpc, http/protobuf, http/json）
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc

# 超时时间（毫秒）
export OTEL_EXPORTER_OTLP_TIMEOUT=10000

# 压缩（gzip, none）
export OTEL_EXPORTER_OTLP_COMPRESSION=gzip

# Headers
export OTEL_EXPORTER_OTLP_HEADERS=api-key=your-key,other-header=value
```

**信号特定配置**:
```bash
# Traces 特定端点
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4318/v1/traces
export OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_TRACES_HEADERS=trace-api-key=key

# Metrics 特定端点
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics
export OTEL_EXPORTER_OTLP_METRICS_PROTOCOL=http/protobuf

# Logs 特定端点
export OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs
export OTEL_EXPORTER_OTLP_LOGS_PROTOCOL=http/protobuf
```

### 21.3 Zipkin 导出器

#### 21.3.1 Zipkin 集成

**添加依赖**:
```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-exporter-zipkin")
}
```

**配置 Zipkin 导出器**:
```java
import io.opentelemetry.exporter.zipkin.ZipkinSpanExporter;

SpanExporter zipkinExporter = ZipkinSpanExporter.builder()
    .setEndpoint("http://localhost:9411/api/v2/spans")
    .setTimeout(Duration.ofSeconds(10))
    .build();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(zipkinExporter).build())
    .setResource(Resource.create(
        Attributes.of(
            ResourceAttributes.SERVICE_NAME, "my-service"
        )
    ))
    .build();
```

**环境变量配置**:
```bash
export OTEL_TRACES_EXPORTER=zipkin
export OTEL_EXPORTER_ZIPKIN_ENDPOINT=http://localhost:9411/api/v2/spans
```

#### 21.3.2 Zipkin 数据格式

**Zipkin Span JSON 格式**:
```json
{
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "id": "b7ad6b7169203331",
  "parentId": "a7ad6b7169203330",
  "name": "GET /users",
  "timestamp": 1704715200000000,
  "duration": 150000,
  "kind": "SERVER",
  "localEndpoint": {
    "serviceName": "my-service",
    "ipv4": "192.168.1.1",
    "port": 8080
  },
  "tags": {
    "http.method": "GET",
    "http.url": "/users",
    "http.status_code": "200"
  },
  "annotations": [
    {
      "timestamp": 1704715200050000,
      "value": "Processing started"
    }
  ]
}
```

### 21.4 Prometheus 导出器

#### 21.4.1 Prometheus 集成

**添加依赖**:
```kotlin
dependencies {
    implementation("io.opentelemetry:opentelemetry-exporter-prometheus")
}
```

**配置 Prometheus HTTP Server**:
```java
import io.opentelemetry.exporter.prometheus.PrometheusHttpServer;

PrometheusHttpServer prometheusServer = PrometheusHttpServer.builder()
    .setHost("localhost")
    .setPort(9464)
    .build();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(prometheusServer)
    .build();

// Prometheus 抓取端点: http://localhost:9464/metrics
```

**创建指标**:
```java
Meter meter = meterProvider.get("my-service");

LongCounter requestCounter = meter.counterBuilder("http_requests_total")
    .setDescription("Total HTTP requests")
    .setUnit("1")
    .build();

requestCounter.add(1,
    Attributes.of(
        AttributeKey.stringKey("method"), "GET",
        AttributeKey.stringKey("endpoint"), "/api/users",
        AttributeKey.stringKey("status"), "200"
    )
);
```

#### 21.4.2 Prometheus 指标格式

**访问 http://localhost:9464/metrics**:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 100.0

# HELP http_response_time_seconds HTTP response time
# TYPE http_response_time_seconds histogram
http_response_time_seconds_bucket{endpoint="/api/users",le="0.01"} 50.0
http_response_time_seconds_bucket{endpoint="/api/users",le="0.05"} 80.0
http_response_time_seconds_bucket{endpoint="/api/users",le="0.1"} 95.0
http_response_time_seconds_bucket{endpoint="/api/users",le="+Inf"} 100.0
http_response_time_seconds_sum{endpoint="/api/users"} 3.5
http_response_time_seconds_count{endpoint="/api/users"} 100.0

# HELP system_memory_usage_bytes Memory usage
# TYPE system_memory_usage_bytes gauge
system_memory_usage_bytes{state="used"} 1.073741824E9
```

### 21.5 Logging 导出器（调试）

#### 21.5.1 配置日志导出器

**Traces**:
```java
import io.opentelemetry.exporter.logging.LoggingSpanExporter;

SpanExporter loggingExporter = LoggingSpanExporter.create();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(SimpleSpanProcessor.create(loggingExporter))
    .build();
```

**Metrics**:
```java
import io.opentelemetry.exporter.logging.LoggingMetricExporter;

MetricExporter loggingExporter = LoggingMetricExporter.create();

SdkMeterProvider meterProvider = SdkMeterProvider.builder()
    .registerMetricReader(
        PeriodicMetricReader.builder(loggingExporter)
            .setInterval(Duration.ofSeconds(60))
            .build()
    )
    .build();
```

**Logs**:
```java
import io.opentelemetry.exporter.logging.LoggingLogRecordExporter;

LogRecordExporter loggingExporter = LoggingLogRecordExporter.create();

SdkLoggerProvider loggerProvider = SdkLoggerProvider.builder()
    .addLogRecordProcessor(SimpleLogRecordProcessor.create(loggingExporter))
    .build();
```

#### 21.5.2 OTLP 格式日志导出

**输出 OTLP JSON 格式（调试）**:
```java
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingSpanExporter;
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingMetricExporter;
import io.opentelemetry.exporter.logging.otlp.OtlpJsonLoggingLogRecordExporter;

// Traces
SpanExporter otlpLoggingExporter = OtlpJsonLoggingSpanExporter.create();

// Metrics
MetricExporter otlpMetricLoggingExporter = OtlpJsonLoggingMetricExporter.create();

// Logs
LogRecordExporter otlpLogLoggingExporter = OtlpJsonLoggingLogRecordExporter.create();
```

**输出示例**（OTLP JSON 格式）:
```json
{
  "resourceSpans": [{
    "resource": {
      "attributes": [{
        "key": "service.name",
        "value": { "stringValue": "my-service" }
      }]
    },
    "scopeSpans": [{
      "spans": [{
        "traceId": "0af7651916cd43dd8448eb211c80319c",
        "spanId": "b7ad6b7169203331",
        "name": "GET /users",
        "kind": "SPAN_KIND_SERVER",
        "startTimeUnixNano": "1704715200000000000",
        "endTimeUnixNano": "1704715200150000000",
        "attributes": [{
          "key": "http.method",
          "value": { "stringValue": "GET" }
        }]
      }]
    }]
  }]
}
```

### 21.6 多导出器配置

#### 21.6.1 同时导出到多个后端

```java
// 创建多个导出器
SpanExporter otlpExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .build();

SpanExporter zipkinExporter = ZipkinSpanExporter.builder()
    .setEndpoint("http://localhost:9411/api/v2/spans")
    .build();

SpanExporter loggingExporter = LoggingSpanExporter.create();

// 添加多个 SpanProcessor
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(otlpExporter).build())
    .addSpanProcessor(BatchSpanProcessor.builder(zipkinExporter).build())
    .addSpanProcessor(SimpleSpanProcessor.create(loggingExporter))
    .build();
```

#### 21.6.2 使用 MultiSpanExporter

```java
import io.opentelemetry.sdk.trace.export.SpanExporter;
import java.util.Arrays;

// 包装多个导出器
SpanExporter multiExporter = SpanExporter.composite(
    Arrays.asList(otlpExporter, zipkinExporter, loggingExporter)
);

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(multiExporter).build())
    .build();
```

### 21.7 导出器性能优化

#### 21.7.1 批处理配置

```java
BatchSpanProcessor processor = BatchSpanProcessor.builder(exporter)
    .setScheduleDelay(Duration.ofSeconds(5))      // 批处理延迟
    .setMaxQueueSize(2048)                        // 队列大小
    .setMaxExportBatchSize(512)                   // 批大小
    .setExporterTimeout(Duration.ofSeconds(30))   // 导出超时
    .build();
```

**参数调优建议**:

| 参数 | 默认值 | 低吞吐量 | 高吞吐量 | 低延迟 |
|------|--------|----------|----------|--------|
| scheduleDelay | 5s | 10s | 1s | 500ms |
| maxQueueSize | 2048 | 512 | 8192 | 2048 |
| maxExportBatchSize | 512 | 128 | 2048 | 256 |
| exporterTimeout | 30s | 30s | 60s | 10s |

#### 21.7.2 压缩配置

```java
// 启用 gzip 压缩（推荐）
OtlpGrpcSpanExporter.builder()
    .setCompression("gzip")
    .build();

// 压缩效果：减少 70-90% 的网络传输
```

#### 21.7.3 连接池配置

**gRPC 连接管理**:
```java
// gRPC 自动管理连接池
// 默认配置已经足够好，通常不需要手动配置
```

### 21.8 故障排查

#### 21.8.1 导出失败诊断

**启用详细日志**:
```bash
java -Djava.util.logging.config.file=logging.properties YourApp
```

**logging.properties**:
```properties
io.opentelemetry.exporter.level=FINE
io.opentelemetry.sdk.trace.export.level=FINE
```

**常见问题**:

**问题 1: 连接被拒绝**
```
Error: Connection refused: localhost/127.0.0.1:4317
```
**解决**: 检查 Collector 是否运行，端口是否正确

**问题 2: 超时**
```
Error: DEADLINE_EXCEEDED: deadline exceeded after 10s
```
**解决**: 增加超时时间或检查网络延迟

**问题 3: 认证失败**
```
Error: UNAUTHENTICATED: invalid authentication credentials
```
**解决**: 检查 API Key 或 Token 配置

#### 21.8.2 性能问题诊断

**检查队列大小**:
```java
// 自定义 SpanProcessor 监控队列
public class MonitoredBatchSpanProcessor implements SpanProcessor {
    private final BatchSpanProcessor delegate;

    @Override
    public void onEnd(ReadableSpan span) {
        delegate.onEnd(span);
        // 监控队列大小
        System.out.println("Queue size: " + getCurrentQueueSize());
    }
}
```

**检查导出延迟**:
```java
SpanExporter instrumentedExporter = new SpanExporter() {
    private final SpanExporter delegate = otlpExporter;

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        long start = System.currentTimeMillis();
        CompletableResultCode result = delegate.export(spans);
        result.whenComplete(() -> {
            long duration = System.currentTimeMillis() - start;
            System.out.println("Export took " + duration + "ms for " + spans.size() + " spans");
        });
        return result;
    }
};
```

### 21.9 最佳实践

#### 21.9.1 生产环境推荐配置

```java
// 推荐：使用 OTLP + gRPC + 批处理
SpanExporter exporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://collector:4317")
    .setTimeout(Duration.ofSeconds(30))
    .setCompression("gzip")
    .build();

SpanProcessor processor = BatchSpanProcessor.builder(exporter)
    .setScheduleDelay(Duration.ofSeconds(5))
    .setMaxQueueSize(2048)
    .setMaxExportBatchSize(512)
    .build();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(processor)
    .setResource(Resource.create(
        Attributes.of(
            ResourceAttributes.SERVICE_NAME, "my-service",
            ResourceAttributes.SERVICE_VERSION, "1.0.0",
            ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production"
        )
    ))
    .build();
```

#### 21.9.2 开发环境推荐配置

```java
// 开发：使用 Logging 导出器 + OTLP JSON 格式
SpanExporter loggingExporter = OtlpJsonLoggingSpanExporter.create();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(SimpleSpanProcessor.create(loggingExporter))
    .build();
```

#### 21.9.3 混合环境配置

```java
// 生产：OTLP + 日志（错误时）
SpanExporter otlpExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://collector:4317")
    .build();

SpanExporter loggingExporter = LoggingSpanExporter.create();

// 正常：导出到 OTLP
// 错误：额外导出到日志
SpanProcessor processor = new ConditionalSpanProcessor(
    BatchSpanProcessor.builder(otlpExporter).build(),
    SimpleSpanProcessor.create(loggingExporter)
);
```

---

**相关章节**:
- ← 上一节: [10. 导出器架构](#10-导出器架构)
- → 下一节: [22. 扩展模块](#22-扩展模块)
- ↑ 返回目录: [目录](#📑-目录)

---
