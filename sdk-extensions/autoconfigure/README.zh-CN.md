# OpenTelemetry SDK 自动配置模块

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure)

**OpenTelemetry SDK Autoconfigure** 是一个便利的扩展模块，提供基于环境变量和系统属性的 OpenTelemetry SDK 自动配置功能。这是编程式配置的简便替代方案，让您无需编写代码即可配置 OpenTelemetry。

---

## 📑 目录

- [概述](#概述)
- [快速开始](#快速开始)
- [核心功能](#核心功能)
- [配置方式](#配置方式)
- [环境变量参考](#环境变量参考)
- [API 使用](#api-使用)
- [SPI 扩展](#spi-扩展)
- [高级用法](#高级用法)
- [常见问题](#常见问题)

---

## 概述

### 为什么需要自动配置？

传统的 SDK 配置需要编写大量代码：

```java
// ❌ 传统方式：需要编写配置代码
Resource resource = Resource.create(Attributes.of(
    AttributeKey.stringKey("service.name"), "my-app"
));

SpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
    .setEndpoint("http://localhost:4317")
    .build();

SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .setResource(resource)
    .addSpanProcessor(BatchSpanProcessor.builder(spanExporter).build())
    .build();

OpenTelemetrySdk sdk = OpenTelemetrySdk.builder()
    .setTracerProvider(tracerProvider)
    .buildAndRegisterGlobal();
```

使用自动配置模块，只需设置环境变量：

```bash
# ✅ 自动配置方式：零代码配置
export OTEL_SERVICE_NAME=my-app
export OTEL_TRACES_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

```java
// Java 代码极其简洁
OpenTelemetrySdk sdk = AutoConfiguredOpenTelemetrySdk.initialize()
    .getOpenTelemetrySdk();
```

### 核心特性

- ✅ **零代码配置**：通过环境变量或系统属性配置所有组件
- ✅ **SPI 自动发现**：自动加载 classpath 中的扩展实现
- ✅ **配置优先级**：系统属性 > 环境变量 > 配置文件 > 默认值
- ✅ **多导出器支持**：同时配置多个导出器（OTLP、Zipkin、Prometheus 等）
- ✅ **声明式配置**：支持 YAML 配置文件（实验性）
- ✅ **灵活定制**：提供 Customizer API 进行编程式定制

---

## 快速开始

### 1. 添加依赖

**Gradle**:
```kotlin
dependencies {
    implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
    implementation("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure")
}
```

**Maven**:
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
        <artifactId>opentelemetry-sdk-extension-autoconfigure</artifactId>
    </dependency>
</dependencies>
```

### 2. 设置环境变量

```bash
# 服务名称（必需）
export OTEL_SERVICE_NAME=my-application

# Trace 导出器
export OTEL_TRACES_EXPORTER=otlp

# OTLP 端点
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# 采样率（10% 采样）
export OTEL_TRACES_SAMPLER=traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1
```

### 3. 初始化 SDK

```java
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;
import io.opentelemetry.sdk.OpenTelemetrySdk;

public class Application {
    public static void main(String[] args) {
        // 自动配置并设置为全局实例
        OpenTelemetrySdk sdk = AutoConfiguredOpenTelemetrySdk.initialize()
            .getOpenTelemetrySdk();

        // 现在可以使用 GlobalOpenTelemetry 获取 Tracer
        // 无需手动传递 sdk 实例
        Tracer tracer = GlobalOpenTelemetry.getTracer("my-app");

        // 您的应用逻辑
        runApplication();
    }
}
```

### 4. 创建第一个 Trace

```java
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.context.Scope;

public class MyService {
    private static final Tracer tracer =
        GlobalOpenTelemetry.getTracer("my-service");

    public void processRequest() {
        Span span = tracer.spanBuilder("processRequest").startSpan();
        try (Scope scope = span.makeCurrent()) {
            // 业务逻辑
            span.setAttribute("user.id", "12345");
            doWork();
        } finally {
            span.end();
        }
    }
}
```

---

## 核心功能

### 自动配置流程

```
应用启动
    ↓
AutoConfiguredOpenTelemetrySdk.initialize()
    ↓
┌─────────────────────────────────────┐
│ 1. 加载配置属性                      │
│    - 系统属性 (-Dotel.xxx)          │
│    - 环境变量 (OTEL_XXX)            │
│    - 配置文件 (可选)                 │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ 2. 尝试声明式配置 (YAML)            │
│    (需要 incubator 依赖)             │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ 3. 环境变量驱动配置                  │
│    ├── 加载 SPI 实现                 │
│    ├── 配置 Resource                 │
│    ├── 配置 TracerProvider           │
│    ├── 配置 MeterProvider            │
│    ├── 配置 LoggerProvider           │
│    └── 配置 Propagators              │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ 4. 构建并返回 OpenTelemetrySdk      │
│ 5. 可选：设置为全局实例              │
└─────────────────────────────────────┘
```

### 配置优先级

配置属性按以下优先级加载（从高到低）：

```
1. 系统属性         (-Dotel.service.name=my-app)
2. 环境变量         (OTEL_SERVICE_NAME=my-app)
3. 配置文件         (config.yaml)
4. 程序默认值       (addPropertiesSupplier)
```

---

## 配置方式

### 1. 环境变量（推荐）

```bash
# Resource 配置
export OTEL_SERVICE_NAME=my-app
export OTEL_RESOURCE_ATTRIBUTES=environment=prod,region=us-west

# Tracer 配置
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1

# OTLP 导出器配置
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_TIMEOUT=10000
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer token"

# Span 处理器配置
export OTEL_BSP_SCHEDULE_DELAY=5000
export OTEL_BSP_MAX_QUEUE_SIZE=2048
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=512

# Metrics 配置
export OTEL_METRICS_EXPORTER=prometheus,otlp
export OTEL_METRIC_EXPORT_INTERVAL=60000

# Logs 配置
export OTEL_LOGS_EXPORTER=otlp

# Propagator 配置
export OTEL_PROPAGATORS=tracecontext,baggage,b3
```

### 2. 系统属性

```bash
java -Dotel.service.name=my-app \
     -Dotel.traces.exporter=otlp \
     -Dotel.exporter.otlp.endpoint=http://localhost:4317 \
     -jar myapp.jar
```

### 3. 配置文件（YAML，实验性）

**添加依赖**:
```kotlin
implementation("io.opentelemetry:opentelemetry-sdk-extension-incubator")
```

**config.yaml**:
```yaml
exporters:
  otlp:
    endpoint: http://localhost:4317
    headers:
      Authorization: Bearer token
    timeout: 10000

processors:
  batch:
    scheduleDelayMillis: 5000
    maxQueueSize: 2048
    maxExportBatchSize: 512

tracers:
  sdk:
    sampler:
      name: parentbased_traceidratio
      arg: 0.1
    processorsList:
      - batch

meters:
  sdk:
    readersList:
      - prometheus

loggers:
  sdk:
    processorsList:
      - batch
```

**使用配置文件**:
```bash
export OTEL_EXPERIMENTAL_CONFIG_FILE=/path/to/config.yaml

java -jar myapp.jar
```

### 4. 编程式配置

```java
Map<String, String> config = new HashMap<>();
config.put("otel.service.name", "my-app");
config.put("otel.traces.exporter", "otlp");

AutoConfiguredOpenTelemetrySdk sdk =
    AutoConfiguredOpenTelemetrySdk.builder()
        .addPropertiesSupplier(() -> config)
        .build();
```

---

## 环境变量参考

### Resource 配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_SERVICE_NAME` | String | - | 服务名称（推荐设置） |
| `OTEL_RESOURCE_ATTRIBUTES` | Map | - | Resource 属性（key1=val1,key2=val2） |
| `OTEL_JAVA_ENABLED_RESOURCE_PROVIDERS` | List | - | 启用的 ResourceProvider 类名 |
| `OTEL_JAVA_DISABLED_RESOURCE_PROVIDERS` | List | - | 禁用的 ResourceProvider 类名 |
| `OTEL_RESOURCE_DISABLED_KEYS` | List | - | 要移除的 Resource 属性键 |

### Tracer 配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_TRACES_EXPORTER` | String | otlp | 导出器名称（可多个：otlp,zipkin,logging） |
| `OTEL_TRACES_SAMPLER` | String | parentbased_always_on | 采样器名称 |
| `OTEL_TRACES_SAMPLER_ARG` | Double | - | 采样器参数（如采样率） |

**支持的采样器**:
- `always_on` - 100% 采样
- `always_off` - 0% 采样
- `traceidratio` - 基于 TraceId 的比例采样（需要 `OTEL_TRACES_SAMPLER_ARG`）
- `parentbased_always_on` - 继承父 Span 决策，根 Span 100% 采样
- `parentbased_always_off` - 继承父 Span 决策，根 Span 0% 采样
- `parentbased_traceidratio` - 继承父 Span 决策，根 Span 比例采样

### Span 处理器配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_BSP_SCHEDULE_DELAY` | Duration | 5000 | 批处理调度延迟（毫秒） |
| `OTEL_BSP_MAX_QUEUE_SIZE` | Integer | 2048 | 最大队列大小 |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | Integer | 512 | 每批最大导出 Span 数 |
| `OTEL_BSP_EXPORT_TIMEOUT` | Duration | 30000 | 导出超时（毫秒） |

### Span 限制配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Integer | - | 属性值长度限制 |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | Integer | 128 | Span 最大属性数 |
| `OTEL_SPAN_EVENT_COUNT_LIMIT` | Integer | 128 | Span 最大事件数 |
| `OTEL_SPAN_LINK_COUNT_LIMIT` | Integer | 128 | Span 最大链接数 |
| `OTEL_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Integer | - | 全局属性值长度限制 |
| `OTEL_ATTRIBUTE_COUNT_LIMIT` | Integer | 128 | 全局属性数量限制 |

### Metrics 配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_METRICS_EXPORTER` | String | otlp | 导出器名称（可多个：otlp,prometheus） |
| `OTEL_METRIC_EXPORT_INTERVAL` | Duration | 60000 | 导出间隔（毫秒） |
| `OTEL_METRICS_EXEMPLAR_FILTER` | String | trace_based | 示范过滤器（trace_based, always_on, always_off） |
| `OTEL_JAVA_METRICS_CARDINALITY_LIMIT` | Integer | 2000 | 基数限制 |

### Logs 配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_LOGS_EXPORTER` | String | otlp | 导出器名称 |
| `OTEL_BLRP_SCHEDULE_DELAY` | Duration | 1000 | 批处理调度延迟（毫秒） |
| `OTEL_BLRP_MAX_QUEUE_SIZE` | Integer | 2048 | 最大队列大小 |
| `OTEL_BLRP_MAX_EXPORT_BATCH_SIZE` | Integer | 512 | 每批最大导出日志数 |
| `OTEL_BLRP_EXPORT_TIMEOUT` | Duration | 30000 | 导出超时（毫秒） |

### OTLP 导出器配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | String | http://localhost:4317 | OTLP 端点（gRPC） |
| `OTEL_EXPORTER_OTLP_HEADERS` | Map | - | HTTP 头（key1=val1,key2=val2） |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | Duration | 10000 | 超时（毫秒） |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | String | grpc | 协议（grpc, http/protobuf, http/json） |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | String | - | 压缩方式（gzip, none） |

**信号特定配置**:
```bash
# Traces 特定端点
OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4317/v1/traces

# Metrics 特定端点
OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4318/v1/metrics

# Logs 特定端点
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs
```

### Propagator 配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_PROPAGATORS` | List | tracecontext,baggage | 传播器列表 |

**支持的传播器**:
- `tracecontext` - W3C Trace Context（推荐）
- `baggage` - W3C Baggage
- `b3` - B3 Single Header
- `b3multi` - B3 Multi Header
- `jaeger` - Jaeger Propagator
- `ottrace` - OT Trace

### 其他配置

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_SDK_DISABLED` | Boolean | false | 完全禁用 SDK |
| `OTEL_EXPERIMENTAL_CONFIG_FILE` | String | - | 声明式配置文件路径 |

---

## API 使用

### 基本用法

```java
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;

// 最简单的方式：自动初始化并设置为全局
OpenTelemetrySdk sdk = AutoConfiguredOpenTelemetrySdk.initialize()
    .getOpenTelemetrySdk();

// 获取 Tracer
Tracer tracer = GlobalOpenTelemetry.getTracer("my-app");

// 获取 Meter
Meter meter = GlobalOpenTelemetry.getMeter("my-app");
```

### 使用 Builder 定制

```java
AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.builder()
        // 添加 Resource 定制
        .addResourceCustomizer((resource, config) ->
            resource.toBuilder()
                .put("deployment.version", "1.0.0")
                .put("deployment.environment", "production")
                .build())

        // 添加 TracerProvider 定制
        .addTracerProviderCustomizer((builder, config) ->
            builder.setSampler(Sampler.traceIdRatioBased(0.1)))

        // 添加 Span 处理器定制
        .addSpanProcessorCustomizer((processor, config) ->
            new FilteringSpanProcessor(processor))

        // 添加 Span 导出器定制
        .addSpanExporterCustomizer((exporter, config) ->
            new LoggingSpanExporter(exporter))

        // 设置为全局实例
        .setResultAsGlobal()

        // 禁用关闭钩子（手动管理生命周期）
        // .disableShutdownHook()

        .build();

OpenTelemetrySdk sdk = autoConfigured.getOpenTelemetrySdk();
```

### 提供默认配置属性

```java
Map<String, String> defaultProps = new HashMap<>();
defaultProps.put("otel.service.name", "my-service");
defaultProps.put("otel.traces.exporter", "otlp");
defaultProps.put("otel.exporter.otlp.endpoint", "http://localhost:4317");

AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.builder()
        .addPropertiesSupplier(() -> defaultProps)
        .build();
```

### 定制配置属性

```java
AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.builder()
        .addPropertiesCustomizer(props -> {
            Map<String, String> customized = new HashMap<>(props);
            // 覆盖或添加属性
            customized.put("otel.traces.sampler", "always_on");
            return customized;
        })
        .build();
```

### 多导出器配置

```java
// 通过环境变量配置多个导出器
// export OTEL_TRACES_EXPORTER=otlp,zipkin,logging

AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.initialize();

// 所有配置的导出器都会被使用
```

### 手动关闭

```java
AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.builder()
        .disableShutdownHook()  // 禁用自动关闭钩子
        .build();

try {
    // 应用逻辑
    runApplication();
} finally {
    // 手动关闭
    autoConfigured.getOpenTelemetrySdk().close();
}
```

---

## SPI 扩展

### ResourceProvider SPI

自定义 Resource 提供者：

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;

public class CustomResourceProvider implements ResourceProvider {

    @Override
    public Resource createResource(ConfigProperties config) {
        // 从配置或环境中获取自定义属性
        String environment = System.getenv("ENVIRONMENT");
        String datacenter = System.getenv("DATACENTER");

        return Resource.builder()
            .put("custom.environment", environment)
            .put("custom.datacenter", datacenter)
            .put("custom.version", "1.0.0")
            .build();
    }

    @Override
    public int order() {
        // 执行顺序：数值越大越晚执行（越高优先级）
        return 100;
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider`:
```
com.example.CustomResourceProvider
```

### ConfigurableSpanExporterProvider SPI

自定义 Span 导出器：

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.trace.export.SpanExporter;

public class CustomSpanExporterProvider implements ConfigurableSpanExporterProvider {

    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        String endpoint = config.getString("otel.exporter.custom.endpoint");
        int timeout = config.getInt("otel.exporter.custom.timeout", 10000);

        return CustomSpanExporter.builder()
            .setEndpoint(endpoint)
            .setTimeout(Duration.ofMillis(timeout))
            .build();
    }

    @Override
    public String getName() {
        return "custom";  // 通过 OTEL_TRACES_EXPORTER=custom 启用
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider`:
```
com.example.CustomSpanExporterProvider
```

**使用自定义导出器**:
```bash
export OTEL_TRACES_EXPORTER=custom
export OTEL_EXPORTER_CUSTOM_ENDPOINT=http://my-backend:8080
```

### ConfigurableSamplerProvider SPI

自定义采样器：

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.trace.samplers.Sampler;

public class CustomSamplerProvider implements ConfigurableSamplerProvider {

    @Override
    public Sampler createSampler(ConfigProperties config) {
        double rate = config.getDouble("otel.traces.sampler.arg", 0.1);
        boolean enableDebug = config.getBoolean("otel.traces.sampler.debug", false);

        return new CustomSampler(rate, enableDebug);
    }

    @Override
    public String getName() {
        return "custom";  // 通过 OTEL_TRACES_SAMPLER=custom 启用
    }
}
```

**使用自定义采样器**:
```bash
export OTEL_TRACES_SAMPLER=custom
export OTEL_TRACES_SAMPLER_ARG=0.2
export OTEL_TRACES_SAMPLER_DEBUG=true
```

### AutoConfigurationCustomizerProvider SPI

全局定制自动配置：

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizer;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;

public class MyCustomizerProvider implements AutoConfigurationCustomizerProvider {

    @Override
    public void customize(AutoConfigurationCustomizer autoConfiguration) {
        autoConfiguration
            // 添加全局 Resource 属性
            .addResourceCustomizer((resource, config) ->
                resource.toBuilder()
                    .put("global.attribute", "value")
                    .build())

            // 过滤特定的 Span
            .addSpanProcessorCustomizer((processor, config) ->
                new FilteringSpanProcessor(processor))

            // 添加自定义采样逻辑
            .addSamplerCustomizer((sampler, config) ->
                new ConditionalSampler(sampler));
    }

    @Override
    public int order() {
        return 0;  // 执行顺序
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider`:
```
com.example.MyCustomizerProvider
```

---

## 高级用法

### 过滤 Span

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
        // 过滤掉内部 Span
        if (span.getName().startsWith("internal-")) {
            return;  // 不导出
        }
        delegate.onEnd(span);
    }

    @Override
    public boolean isEndRequired() {
        return true;
    }

    @Override
    public CompletableResultCode shutdown() {
        return delegate.shutdown();
    }

    @Override
    public CompletableResultCode forceFlush() {
        return delegate.forceFlush();
    }
}

// 使用
AutoConfiguredOpenTelemetrySdk.builder()
    .addSpanProcessorCustomizer((processor, config) ->
        new FilteringSpanProcessor(processor))
    .build();
```

### 增强导出器（添加日志）

```java
public class LoggingSpanExporter implements SpanExporter {

    private final SpanExporter delegate;
    private static final Logger logger = LoggerFactory.getLogger(LoggingSpanExporter.class);

    public LoggingSpanExporter(SpanExporter delegate) {
        this.delegate = delegate;
    }

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        logger.info("Exporting {} spans", spans.size());

        for (SpanData span : spans) {
            logger.debug("Span: {} [{}]", span.getName(), span.getSpanId());
        }

        return delegate.export(spans);
    }

    @Override
    public CompletableResultCode flush() {
        return delegate.flush();
    }

    @Override
    public CompletableResultCode shutdown() {
        return delegate.shutdown();
    }
}

// 使用
AutoConfiguredOpenTelemetrySdk.builder()
    .addSpanExporterCustomizer((exporter, config) ->
        new LoggingSpanExporter(exporter))
    .build();
```

### 条件采样器

```java
public class ConditionalSampler implements Sampler {

    private final Sampler delegate;

    public ConditionalSampler(Sampler delegate) {
        this.delegate = delegate;
    }

    @Override
    public SamplingResult shouldSample(
            Context parentContext,
            String traceId,
            String name,
            SpanKind spanKind,
            Attributes attributes,
            List<LinkData> parentLinks) {

        // 优先采样带有 "priority" 属性的 Span
        if (attributes.get(AttributeKey.booleanKey("priority")) == Boolean.TRUE) {
            return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
        }

        // 否则使用默认采样器
        return delegate.shouldSample(parentContext, traceId, name, spanKind, attributes, parentLinks);
    }

    @Override
    public String getDescription() {
        return "ConditionalSampler{" + delegate.getDescription() + "}";
    }
}

// 使用
AutoConfiguredOpenTelemetrySdk.builder()
    .addSamplerCustomizer((sampler, config) ->
        new ConditionalSampler(sampler))
    .build();
```

### 动态 Resource 检测

```java
public class DynamicResourceProvider implements ResourceProvider {

    @Override
    public Resource createResource(ConfigProperties config) {
        Attributes.Builder builder = Attributes.builder();

        // 检测容器环境
        if (isRunningInKubernetes()) {
            builder.put("k8s.pod.name", getPodName());
            builder.put("k8s.namespace.name", getNamespace());
        }

        // 检测云环境
        if (isRunningOnAWS()) {
            builder.put("cloud.provider", "aws");
            builder.put("cloud.region", getAwsRegion());
        }

        // 检测主机信息
        builder.put("host.name", getHostName());
        builder.put("host.arch", System.getProperty("os.arch"));

        return Resource.create(builder.build());
    }

    private boolean isRunningInKubernetes() {
        return System.getenv("KUBERNETES_SERVICE_HOST") != null;
    }

    private boolean isRunningOnAWS() {
        // 检测 AWS 元数据服务
        try {
            HttpURLConnection conn = (HttpURLConnection)
                new URL("http://169.254.169.254/latest/meta-data/").openConnection();
            conn.setConnectTimeout(100);
            conn.setReadTimeout(100);
            return conn.getResponseCode() == 200;
        } catch (Exception e) {
            return false;
        }
    }

    @Override
    public int order() {
        return 200;  // 在默认 ResourceProvider 之后执行
    }
}
```

---

## buildImpl 方法源码解析

深入解析 `AutoConfiguredOpenTelemetrySdkBuilder#buildImpl` 方法的实现逻辑，这是自动配置的核心方法。

### 方法概述

`buildImpl` 是自动配置的核心入口方法，位于 `AutoConfiguredOpenTelemetrySdkBuilder.java` 的第 447-512 行。它负责：

- ✅ 尝试声明式配置（YAML 文件）
- ✅ 加载和执行 SPI 扩展和定制器
- ✅ 加载配置属性（环境变量、系统属性）
- ✅ 配置 Resource（服务信息）
- ✅ 构建 TracerProvider、MeterProvider、LoggerProvider
- ✅ 组装最终的 OpenTelemetrySdk
- ✅ 注册关闭钩子和监听器
- ✅ 异常处理和资源清理

**方法签名**:
```java
private AutoConfiguredOpenTelemetrySdk buildImpl()
```

**在架构中的位置**:
```
AutoConfiguredOpenTelemetrySdk
└── build()
    └── buildImpl()  ← 核心配置方法
        ├── maybeConfigureFromFile()      # 声明式配置
        ├── SpiHelper.create()            # SPI 加载
        ├── getConfig()                   # 配置属性加载
        ├── ResourceConfiguration         # Resource 配置
        ├── configureSdk()                # SDK 组件配置
        │   ├── MeterProvider
        │   ├── TracerProvider
        │   └── LoggerProvider
        └── 异常处理和资源清理
```

### 执行流程图

```
buildImpl() 执行流程
│
├─┤阶段 1: 尝试声明式配置 (448-457 行)│
│  │
│  ├─ maybeConfigureFromFile()
│  │   ├─ 检查 INCUBATOR_AVAILABLE
│  │   ├─ 读取 otel.experimental.config.file
│  │   └─ 如果成功 → 直接返回 (跳过后续步骤)
│  │
│  └─ 如果失败 → 继续环境变量配置
│
├─┤阶段 2: 初始化 SPI 和定制器 (459-467 行)│
│  │
│  ├─ SpiHelper.create(componentLoader)
│  ├─ mergeSdkTracerProviderConfigurer()     # 兼容旧版 API
│  └─ 执行 AutoConfigurationCustomizerProvider
│      └─ customizer.customize(this)
│
├─┤阶段 3: 加载配置属性 (468 行)│
│  │
│  └─ getConfig() → computeConfigProperties()
│      ├─ DefaultConfigProperties.create()
│      ├─ 应用 propertiesSupplier
│      └─ 应用 propertiesCustomizers
│
├─┤阶段 4: 配置 Resource (470-471 行)│
│  │
│  └─ ResourceConfiguration.configureResource()
│      ├─ 加载 ResourceProvider SPI
│      ├─ 合并所有 Resource
│      └─ 应用 resourceCustomizer
│
├─┤阶段 5: 构建 SDK 组件 (473-494 行)│
│  │
│  ├─ 创建 closeables 列表 (资源跟踪)
│  ├─ OpenTelemetrySdkBuilder.builder()
│  │
│  ├─ 5.1: 配置 Propagators
│  │   └─ PropagatorConfiguration.configurePropagators()
│  │
│  ├─ 5.2: 检查 otel.sdk.disabled
│  │   └─ if (sdkEnabled) → configureSdk()
│  │
│  ├─ 5.3: configureSdk() 详解见下方
│  │   ├─ MeterProvider (先创建)
│  │   ├─ TracerProvider (依赖 MeterProvider)
│  │   └─ LoggerProvider (依赖 MeterProvider)
│  │
│  ├─ 5.4: 构建 OpenTelemetrySdk
│  │   └─ sdkBuilder.build()
│  │
│  ├─ 5.5: 注册关闭钩子
│  │   └─ maybeRegisterShutdownHook()
│  │
│  ├─ 5.6: 调用监听器
│  │   └─ callAutoConfigureListeners()
│  │
│  └─ 5.7: 返回结果
│      └─ AutoConfiguredOpenTelemetrySdk.create()
│
└─┤阶段 6: 异常处理 (495-511 行)│
   │
   └─ catch (RuntimeException e)
       ├─ 清理所有 closeables
       ├─ 记录错误日志
       └─ 包装为 ConfigurationException
```

### 阶段 1: 声明式配置（YAML）

**源码位置**: 第 448-457 行

```java
// 尝试从文件配置加载
AutoConfiguredOpenTelemetrySdk fromFileConfiguration =
    maybeConfigureFromFile(
        this.config != null
            ? this.config
            : DefaultConfigProperties.create(Collections.emptyMap(), componentLoader),
        componentLoader);

// 如果成功则直接返回
if (fromFileConfiguration != null) {
    maybeRegisterShutdownHook(fromFileConfiguration.getOpenTelemetrySdk());
    return fromFileConfiguration;
}
```

**关键点**:

1. **优先级最高**: 声明式配置优先于环境变量配置
2. **Incubator 依赖**: 需要 `opentelemetry-sdk-extension-incubator` 模块
3. **早期返回**: 如果成功则跳过所有后续配置步骤

**maybeConfigureFromFile 逻辑** (571-595 行):

```java
@Nullable
private static AutoConfiguredOpenTelemetrySdk maybeConfigureFromFile(
    ConfigProperties config, ComponentLoader componentLoader) {

    // 1. 检查 incubator 模块是否可用
    if (INCUBATOR_AVAILABLE) {
        // 2. 尝试从 SPI 配置
        AutoConfiguredOpenTelemetrySdk sdk = IncubatingUtil.configureFromSpi(componentLoader);
        if (sdk != null) {
            return sdk;
        }
    }

    // 3. 检查配置文件路径
    String configurationFile = config.getString("otel.experimental.config.file");
    if (configurationFile == null || configurationFile.isEmpty()) {
        return null;  // 没有配置文件,继续环境变量配置
    }

    // 4. 加载配置文件
    if (!INCUBATOR_AVAILABLE) {
        throw new ConfigurationException(
            "Cannot autoconfigure from config file without " +
            "opentelemetry-sdk-extension-incubator on the classpath");
    }

    return IncubatingUtil.configureFromFile(logger, configurationFile, componentLoader);
}
```

**使用场景**:

```bash
# 使用 YAML 配置文件
export OTEL_EXPERIMENTAL_CONFIG_FILE=/path/to/config.yaml

# config.yaml 内容
exporters:
  otlp:
    endpoint: http://localhost:4317
tracers:
  sdk:
    sampler: always_on
```

### 阶段 2: 初始化 SPI 和定制器

**源码位置**: 第 459-467 行

```java
// 创建 SPI 助手
SpiHelper spiHelper = SpiHelper.create(componentLoader);

// 确保只执行一次定制器
if (!customized) {
    customized = true;

    // 1. 兼容旧版 SdkTracerProviderConfigurer (已废弃)
    mergeSdkTracerProviderConfigurer();

    // 2. 加载并执行所有 AutoConfigurationCustomizerProvider
    for (AutoConfigurationCustomizerProvider customizer :
          spiHelper.loadOrdered(AutoConfigurationCustomizerProvider.class)) {
        customizer.customize(this);
    }
}
```

**关键点**:

1. **SpiHelper 职责**:
   - 加载 SPI 实现（ResourceProvider、SpanExporterProvider 等）
   - 管理组件的生命周期
   - 提供有序加载机制（通过 `Ordered` 接口）

2. **customized 标志**:
   - 防止重复执行定制器
   - 允许多次调用 `build()` 而不重复定制

3. **AutoConfigurationCustomizerProvider**:
   ```java
   public interface AutoConfigurationCustomizerProvider extends Ordered {
       void customize(AutoConfigurationCustomizer autoConfiguration);
       int order();  // 执行顺序
   }
   ```

**自定义 SPI 示例**:

```java
// 实现自定义定制器
public class MyCustomizerProvider implements AutoConfigurationCustomizerProvider {

    @Override
    public void customize(AutoConfigurationCustomizer autoConfiguration) {
        // 添加全局 Resource 属性
        autoConfiguration.addResourceCustomizer((resource, config) ->
            resource.toBuilder()
                .put("deployment.environment", "production")
                .put("service.namespace", "my-namespace")
                .build());

        // 添加 Span 过滤器
        autoConfiguration.addSpanProcessorCustomizer((processor, config) ->
            new FilteringSpanProcessor(processor));
    }

    @Override
    public int order() {
        return 100;  // 执行顺序
    }
}

// 注册 SPI
// 文件: META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider
// 内容: com.example.MyCustomizerProvider
```

### 阶段 3: 加载配置属性

**源码位置**: 第 468 行

```java
ConfigProperties config = getConfig();
```

**getConfig 方法** (629-635 行):

```java
private ConfigProperties getConfig() {
    ConfigProperties config = this.config;
    if (config == null) {
        config = computeConfigProperties();
    }
    return config;
}
```

**computeConfigProperties 方法** (637-645 行):

```java
private ConfigProperties computeConfigProperties() {
    // 1. 从 propertiesSupplier 创建基础配置
    DefaultConfigProperties properties =
        DefaultConfigProperties.create(propertiesSupplier.get(), componentLoader);

    // 2. 应用所有 propertiesCustomizers
    for (Function<ConfigProperties, Map<String, String>> customizer : propertiesCustomizers) {
        Map<String, String> overrides = customizer.apply(properties);
        properties = properties.withOverrides(overrides);
    }

    // 3. 应用 configPropertiesCustomizer
    return configPropertiesCustomizer.apply(properties);
}
```

**配置优先级**:

```
1. 系统属性 (java -Dotel.service.name=my-app)
   ↓ 覆盖
2. 环境变量 (OTEL_SERVICE_NAME=my-app)
   ↓ 覆盖
3. propertiesCustomizers (代码中动态添加)
   ↓ 覆盖
4. propertiesSupplier (代码中提供的默认值)
```

**示例**:

```java
// 提供默认配置
Map<String, String> defaults = new HashMap<>();
defaults.put("otel.service.name", "my-service");
defaults.put("otel.traces.exporter", "otlp");

AutoConfiguredOpenTelemetrySdk.builder()
    // 添加默认值
    .addPropertiesSupplier(() -> defaults)

    // 添加动态覆盖
    .addPropertiesCustomizer(props -> {
        Map<String, String> overrides = new HashMap<>();
        if (isProduction()) {
            overrides.put("otel.traces.sampler", "traceidratio");
            overrides.put("otel.traces.sampler.arg", "0.1");
        }
        return overrides;
    })
    .build();
```

### 阶段 4: 配置 Resource

**源码位置**: 第 470-471 行

```java
Resource resource =
    ResourceConfiguration.configureResource(config, spiHelper, resourceCustomizer);
```

**ResourceConfiguration.configureResource 流程**:

```
ResourceConfiguration.configureResource()
│
├─ 1. 从环境变量创建 Resource
│   ├─ OTEL_SERVICE_NAME
│   └─ OTEL_RESOURCE_ATTRIBUTES
│
├─ 2. 加载所有 ResourceProvider SPI
│   ├─ OsResourceProvider (操作系统信息)
│   ├─ ProcessResourceProvider (进程信息)
│   ├─ HostResourceProvider (主机信息)
│   ├─ ContainerResourceProvider (容器信息)
│   └─ 自定义 ResourceProvider
│
├─ 3. 合并所有 Resource
│   └─ Resource.merge()
│
├─ 4. 应用 resourceCustomizer
│   └─ resourceCustomizer.apply(resource, config)
│
└─ 5. 返回最终 Resource
```

**Resource 示例**:

```json
{
  "service.name": "my-service",
  "service.version": "1.0.0",
  "deployment.environment": "production",
  "host.name": "server-01",
  "host.arch": "amd64",
  "os.type": "linux",
  "process.pid": "12345",
  "process.command": "java",
  "telemetry.sdk.name": "opentelemetry",
  "telemetry.sdk.language": "java",
  "telemetry.sdk.version": "1.35.0"
}
```

### 阶段 5: 构建 SDK 组件

**源码位置**: 第 473-494 行

```java
// 创建资源跟踪列表
List<Closeable> closeables = new ArrayList<>();

try {
    OpenTelemetrySdkBuilder sdkBuilder = OpenTelemetrySdk.builder();

    // 5.1: 配置传播器 (Propagators)
    ContextPropagators propagators =
        PropagatorConfiguration.configurePropagators(config, spiHelper, propagatorCustomizer);
    sdkBuilder.setPropagators(propagators);

    // 5.2: 检查 SDK 是否启用
    boolean sdkEnabled = !config.getBoolean("otel.sdk.disabled", false);
    if (sdkEnabled) {
        // 5.3: 配置 SDK 组件
        configureSdk(sdkBuilder, config, resource, spiHelper, closeables);
    }

    // 5.4: 构建 OpenTelemetrySdk
    OpenTelemetrySdk openTelemetrySdk = sdkBuilder.build();

    // 5.5: 注册关闭钩子
    maybeRegisterShutdownHook(openTelemetrySdk);

    // 5.6: 调用监听器
    callAutoConfigureListeners(spiHelper, openTelemetrySdk);

    // 5.7: 返回结果
    return AutoConfiguredOpenTelemetrySdk.create(openTelemetrySdk, resource, config);
}
```

**关键设计**:

1. **closeables 列表**:
   - 跟踪所有需要关闭的资源
   - 配置失败时自动清理
   - 防止资源泄漏

2. **Propagators 独立配置**:
   - 传播器属于 API 层（不是 SDK 层）
   - 即使 SDK 禁用，传播器仍然工作
   - 支持跨进程 Context 传播

3. **otel.sdk.disabled 检查**:
   ```bash
   # 完全禁用 SDK (仅保留 API 和 Propagators)
   export OTEL_SDK_DISABLED=true
   ```

### configureSdk 方法详解

**源码位置**: 第 515-568 行

这是构建 SDK 三大 Provider 的核心方法。

```java
void configureSdk(
    OpenTelemetrySdkBuilder sdkBuilder,
    ConfigProperties config,
    Resource resource,
    SpiHelper spiHelper,
    List<Closeable> closeables)
```

**执行顺序和原因**:

#### 1. 配置 MeterProvider (521-533 行)

```java
// 创建 MeterProvider Builder
SdkMeterProviderBuilder meterProviderBuilder = SdkMeterProvider.builder();
meterProviderBuilder.setResource(resource);

// 配置 MetricReader 和 MetricExporter
MeterProviderConfiguration.configureMeterProvider(
    meterProviderBuilder,
    config,
    spiHelper,
    metricReaderCustomizer,
    metricExporterCustomizer,
    closeables);

// 应用用户定制器
meterProviderBuilder = meterProviderCustomizer.apply(meterProviderBuilder, config);

// 构建并添加到 closeables
SdkMeterProvider meterProvider = meterProviderBuilder.build();
closeables.add(meterProvider);
```

**为什么 MeterProvider 先创建？**

TracerProvider 和 LoggerProvider 需要 MeterProvider 来记录内部指标：
- `SpanProcessor` 的处理速度、队列长度
- `LogRecordProcessor` 的处理速度、队列长度
- 导出器的成功/失败次数

#### 2. 配置 TracerProvider (535-548 行)

```java
// 创建 TracerProvider Builder
SdkTracerProviderBuilder tracerProviderBuilder = SdkTracerProvider.builder();
tracerProviderBuilder.setResource(resource);

// 配置 Sampler, SpanProcessor, SpanExporter
TracerProviderConfiguration.configureTracerProvider(
    tracerProviderBuilder,
    config,
    spiHelper,
    meterProvider,  // ← 传递 MeterProvider
    spanExporterCustomizer,
    spanProcessorCustomizer,
    samplerCustomizer,
    closeables);

// 应用用户定制器
tracerProviderBuilder = tracerProviderCustomizer.apply(tracerProviderBuilder, config);

// 构建并添加到 closeables
SdkTracerProvider tracerProvider = tracerProviderBuilder.build();
closeables.add(tracerProvider);
```

#### 3. 配置 LoggerProvider (550-562 行)

```java
// 创建 LoggerProvider Builder
SdkLoggerProviderBuilder loggerProviderBuilder = SdkLoggerProvider.builder();
loggerProviderBuilder.setResource(resource);

// 配置 LogRecordProcessor 和 LogRecordExporter
LoggerProviderConfiguration.configureLoggerProvider(
    loggerProviderBuilder,
    config,
    spiHelper,
    meterProvider,  // ← 传递 MeterProvider
    logRecordExporterCustomizer,
    logRecordProcessorCustomizer,
    closeables);

// 应用用户定制器
loggerProviderBuilder = loggerProviderCustomizer.apply(loggerProviderBuilder, config);

// 构建并添加到 closeables
SdkLoggerProvider loggerProvider = loggerProviderBuilder.build();
closeables.add(loggerProvider);
```

#### 4. 组装 SDK (564-567 行)

```java
sdkBuilder
    .setTracerProvider(tracerProvider)
    .setLoggerProvider(loggerProvider)
    .setMeterProvider(meterProvider);
```

**Provider 配置架构**:

```
┌─────────────────────────────────────────┐
│         configureSdk()                  │
├─────────────────────────────────────────┤
│                                         │
│  1. MeterProvider                       │
│     ↓ 先创建                            │
│     ├─ MetricReader                     │
│     ├─ MetricExporter                   │
│     └─ 添加到 closeables                │
│                                         │
│  2. TracerProvider                      │
│     ↓ 依赖 MeterProvider               │
│     ├─ Sampler                          │
│     ├─ SpanProcessor (使用 MeterProvider)│
│     ├─ SpanExporter                     │
│     └─ 添加到 closeables                │
│                                         │
│  3. LoggerProvider                      │
│     ↓ 依赖 MeterProvider               │
│     ├─ LogRecordProcessor (使用 MeterProvider)│
│     ├─ LogRecordExporter                │
│     └─ 添加到 closeables                │
│                                         │
│  4. 组装到 OpenTelemetrySdkBuilder      │
│     └─ sdkBuilder.set*Provider()        │
└─────────────────────────────────────────┘
```

### 阶段 6: 异常处理和资源清理

**源码位置**: 第 495-511 行

```java
catch (RuntimeException e) {
    // 记录错误信息
    logger.info(
        "Error encountered during autoconfiguration. " +
        "Closing partially configured components.");

    // 清理所有已创建的 Closeable 资源
    for (Closeable closeable : closeables) {
        try {
            logger.fine("Closing " + closeable.getClass().getName());
            closeable.close();
        } catch (IOException ex) {
            logger.warning(
                "Error closing " + closeable.getClass().getName() + ": " + ex.getMessage());
        }
    }

    // 包装异常
    if (e instanceof ConfigurationException) {
        throw e;  // 已经是 ConfigurationException,直接抛出
    }
    throw new ConfigurationException("Unexpected configuration error", e);
}
```

**资源清理流程**:

```
配置失败
    ↓
遍历 closeables 列表
    ├─ MeterProvider.close()
    │   └─ 停止 MetricReader
    │   └─ 关闭 MetricExporter
    │
    ├─ TracerProvider.close()
    │   └─ 停止 SpanProcessor
    │   └─ 刷新并关闭 SpanExporter
    │
    └─ LoggerProvider.close()
        └─ 停止 LogRecordProcessor
        └─ 刷新并关闭 LogRecordExporter
    ↓
包装为 ConfigurationException
    ↓
抛出异常
```

**为什么需要 closeables 列表？**

1. **部分配置成功**: 如果 TracerProvider 配置成功但 LoggerProvider 失败，需要清理 TracerProvider
2. **资源泄漏防止**: 确保所有连接、线程、文件句柄都被正确释放
3. **一致性保证**: 失败状态下不留下任何部分初始化的组件

**异常示例**:

```java
// 场景 1: 导出器配置错误
// export OTEL_TRACES_EXPORTER=invalid-exporter
// 结果: ConfigurationException: Unknown exporter: invalid-exporter

// 场景 2: 端点连接失败
// export OTEL_EXPORTER_OTLP_ENDPOINT=http://invalid:4317
// 结果: 导出器创建成功,但运行时连接失败

// 场景 3: 配置文件格式错误
// export OTEL_EXPERIMENTAL_CONFIG_FILE=invalid.yaml
// 结果: ConfigurationException: Failed to parse configuration file
```

### 关键设计模式

#### 1. Builder 模式

**目的**: 提供灵活的配置方式

```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addResourceCustomizer(...)          // 链式调用
    .addTracerProviderCustomizer(...)
    .addSpanProcessorCustomizer(...)
    .setResultAsGlobal()
    .build();
```

#### 2. 责任链模式 (Customizer)

**目的**: 多个定制器按顺序执行

```java
// Customizer 合并实现 (662-669 行)
private static <I, O1, O2> BiFunction<I, ConfigProperties, O2> mergeCustomizer(
    BiFunction<? super I, ConfigProperties, ? extends O1> first,
    BiFunction<? super O1, ConfigProperties, ? extends O2> second) {
    return (I configured, ConfigProperties config) -> {
        O1 firstResult = first.apply(configured, config);
        return second.apply(firstResult, config);
    };
}

// 使用示例
autoConfiguration
    .addResourceCustomizer((resource, config) ->
        resource.toBuilder().put("attr1", "value1").build())  // Customizer 1
    .addResourceCustomizer((resource, config) ->
        resource.toBuilder().put("attr2", "value2").build())  // Customizer 2
    // 最终 Resource 包含: attr1=value1, attr2=value2
```

#### 3. 资源管理模式 (RAII)

**目的**: 确保资源正确释放

```java
// closeables 列表模式
List<Closeable> closeables = new ArrayList<>();
try {
    SdkMeterProvider meterProvider = ...;
    closeables.add(meterProvider);  // 跟踪资源

    SdkTracerProvider tracerProvider = ...;
    closeables.add(tracerProvider);  // 跟踪资源

    // 更多资源...
} catch (Exception e) {
    // 自动清理所有 closeables
    for (Closeable closeable : closeables) {
        closeable.close();
    }
    throw e;
}
```

#### 4. 模板方法模式 (configureSdk)

**目的**: 标准化配置流程

```java
void configureSdk(...) {
    // 模板方法定义标准流程
    // 1. 配置 MeterProvider
    SdkMeterProvider meterProvider = configureMeterProvider(...);

    // 2. 配置 TracerProvider (依赖 MeterProvider)
    SdkTracerProvider tracerProvider = configureTracerProvider(meterProvider, ...);

    // 3. 配置 LoggerProvider (依赖 MeterProvider)
    SdkLoggerProvider loggerProvider = configureLoggerProvider(meterProvider, ...);

    // 4. 组装
    sdkBuilder.set*Provider(...);
}
```

#### 5. SPI 模式

**目的**: 可扩展的组件加载

```java
// SpiHelper 加载扩展
SpiHelper spiHelper = SpiHelper.create(componentLoader);

// 加载所有 ResourceProvider 实现
List<ResourceProvider> providers = spiHelper.loadOrdered(ResourceProvider.class);

// 按 order() 排序执行
for (ResourceProvider provider : providers) {
    Resource r = provider.createResource(config);
    // 合并到最终 Resource
}
```

### 代码示例和最佳实践

#### 示例 1: 理解 closeables 机制

```java
public class CloseablesExample {
    public static void main(String[] args) {
        List<Closeable> closeables = new ArrayList<>();

        try {
            // 模拟配置过程
            Closeable resource1 = createResource1();
            closeables.add(resource1);  // 成功

            Closeable resource2 = createResource2();
            closeables.add(resource2);  // 成功

            Closeable resource3 = createResource3();  // 失败!
            // resource3 不会添加到 closeables

        } catch (Exception e) {
            System.out.println("Configuration failed. Cleaning up...");

            // 自动清理已创建的资源
            for (Closeable closeable : closeables) {
                try {
                    closeable.close();
                    System.out.println("Closed: " + closeable.getClass().getName());
                } catch (IOException ex) {
                    System.err.println("Error closing: " + ex.getMessage());
                }
            }
            throw new RuntimeException("Configuration failed", e);
        }
    }
}
```

#### 示例 2: 自定义 AutoConfigurationCustomizerProvider

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizer;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;

/**
 * 全局定制自动配置
 */
public class ProductionCustomizerProvider implements AutoConfigurationCustomizerProvider {

    @Override
    public void customize(AutoConfigurationCustomizer autoConfiguration) {
        // 1. 添加生产环境 Resource 属性
        autoConfiguration.addResourceCustomizer((resource, config) ->
            resource.toBuilder()
                .put("deployment.environment", "production")
                .put("service.namespace", "my-company")
                .build());

        // 2. 过滤内部 Span
        autoConfiguration.addSpanProcessorCustomizer((processor, config) ->
            new FilteringSpanProcessor(processor) {
                @Override
                public void onEnd(ReadableSpan span) {
                    if (!span.getName().startsWith("internal.")) {
                        super.onEnd(span);
                    }
                }
            });

        // 3. 添加日志记录导出器
        autoConfiguration.addSpanExporterCustomizer((exporter, config) ->
            new LoggingSpanExporter(exporter));
    }

    @Override
    public int order() {
        return 100;  // 在默认定制器之后执行
    }
}

// 注册 SPI
// 文件: META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider
// 内容: com.example.ProductionCustomizerProvider
```

#### 示例 3: 调试配置过程

```java
public class DebugAutoConfiguration {
    public static void main(String[] args) {
        // 启用 debug 日志
        System.setProperty("java.util.logging.config.file", "logging.properties");

        // 创建自定义 Builder
        AutoConfiguredOpenTelemetrySdk autoConfigured =
            AutoConfiguredOpenTelemetrySdk.builder()
                // 打印配置属性
                .addPropertiesCustomizer(props -> {
                    System.out.println("=== Configuration Properties ===");
                    props.getPropertyKeys().forEach(key ->
                        System.out.println(key + " = " + props.getString(key)));
                    return Collections.emptyMap();
                })

                // 打印 Resource
                .addResourceCustomizer((resource, config) -> {
                    System.out.println("=== Resource Attributes ===");
                    resource.getAttributes().forEach((key, value) ->
                        System.out.println(key + " = " + value));
                    return resource;
                })

                // 打印 TracerProvider
                .addTracerProviderCustomizer((builder, config) -> {
                    System.out.println("=== TracerProvider Configuration ===");
                    System.out.println("Sampler: " + config.getString("otel.traces.sampler"));
                    System.out.println("Exporter: " + config.getString("otel.traces.exporter"));
                    return builder;
                })

                .build();

        OpenTelemetrySdk sdk = autoConfigured.getOpenTelemetrySdk();
        System.out.println("=== SDK Initialized Successfully ===");
    }
}
```

### 常见问题

#### Q1: 为什么 MeterProvider 需要先创建？

**答**: TracerProvider 和 LoggerProvider 的内部组件（SpanProcessor、LogRecordProcessor）需要使用 MeterProvider 来记录内部指标。

示例指标:
- `otel.span_processor.queue.size` - Span 处理队列长度
- `otel.span_processor.processed` - 已处理的 Span 数量
- `otel.span_processor.dropped` - 丢弃的 Span 数量

如果 MeterProvider 后创建，这些内部指标将无法记录。

#### Q2: 配置失败时如何确保资源被清理？

**答**: 通过 `closeables` 列表机制：

```java
List<Closeable> closeables = new ArrayList<>();
try {
    // 每创建一个资源就添加到列表
    Resource r = createResource();
    closeables.add(r);
} catch (Exception e) {
    // 自动清理所有已创建的资源
    for (Closeable c : closeables) {
        c.close();
    }
    throw e;
}
```

这确保了即使配置中途失败，已创建的资源也会被正确释放。

#### Q3: 如何完全禁用某个 Provider？

**答**: 使用 `otel.sdk.disabled` 或信号特定的导出器配置：

```bash
# 方法 1: 完全禁用 SDK
export OTEL_SDK_DISABLED=true

# 方法 2: 禁用 Traces
export OTEL_TRACES_EXPORTER=none

# 方法 3: 禁用 Metrics
export OTEL_METRICS_EXPORTER=none

# 方法 4: 禁用 Logs
export OTEL_LOGS_EXPORTER=none
```

#### Q4: 声明式配置和环境变量配置有什么区别？

**答**:

| 特性 | 声明式配置 (YAML) | 环境变量配置 |
|------|------------------|-------------|
| **优先级** | 更高（先执行） | 较低 |
| **配置方式** | YAML 文件 | 环境变量/系统属性 |
| **依赖** | 需要 incubator 模块 | 无额外依赖 |
| **灵活性** | 结构化、可读性强 | 简单、直接 |
| **适用场景** | 复杂配置、多环境 | 简单配置、容器化 |

#### Q5: 如何调试配置过程？

**答**: 使用以下方法：

1. **启用 debug 日志**:
```bash
export OTEL_JAVAAGENT_DEBUG=true
```

2. **使用 Customizer 打印配置**:
```java
.addPropertiesCustomizer(props -> {
    System.out.println("Config: " + props.getString("otel.service.name"));
    return Collections.emptyMap();
})
```

3. **检查 Resource**:
```java
Resource resource = autoConfigured.getResource();
resource.getAttributes().forEach((key, value) ->
    System.out.println(key + " = " + value));
```

4. **使用 LoggingExporter**:
```bash
export OTEL_TRACES_EXPORTER=logging
```

#### Q6: customized 标志的作用是什么？

**答**: 防止重复执行定制器。

```java
if (!customized) {
    customized = true;
    // 执行 AutoConfigurationCustomizerProvider
}
```

如果用户多次调用 `build()`，定制器只会在第一次执行，避免重复配置。

#### Q7: 如何处理配置异常？

**答**: 所有异常都会被包装为 `ConfigurationException`：

```java
try {
    AutoConfiguredOpenTelemetrySdk sdk =
        AutoConfiguredOpenTelemetrySdk.initialize();
} catch (ConfigurationException e) {
    logger.error("Failed to configure OpenTelemetry", e);

    // 检查具体原因
    if (e.getCause() instanceof ClassNotFoundException) {
        logger.error("Missing dependency: " + e.getCause().getMessage());
    }

    // 使用降级策略
    OpenTelemetry fallback = OpenTelemetry.noop();
}
```

---

## 核心组件

本章节详细介绍 autoconfigure 模块的核心组件，包括核心配置类、组件工厂类和辅助类。这些类共同实现了基于环境变量和系统属性的 SDK 自动配置功能。

### 组件架构概览

autoconfigure 模块的核心组件分为 3 大类：

```
AutoConfigure 模块架构
├── 核心配置类（6 个）
│   ├── AutoConfiguredOpenTelemetrySdk          # 入口和包装器
│   ├── AutoConfiguredOpenTelemetrySdkBuilder   # 主构建器
│   ├── ResourceConfiguration                   # Resource 配置
│   ├── TracerProviderConfiguration             # TracerProvider 配置
│   ├── MeterProviderConfiguration              # MeterProvider 配置
│   └── LoggerProviderConfiguration             # LoggerProvider 配置
│
├── 组件工厂类（4 个）
│   ├── SpanExporterConfiguration               # Span 导出器工厂
│   ├── MetricExporterConfiguration             # Metric 导出器/读取器工厂
│   ├── LogRecordExporterConfiguration          # 日志导出器工厂
│   └── PropagatorConfiguration                 # 传播器工厂
│
└── 辅助类（6 个）
    ├── SpiHelper                                # SPI 加载管理
    ├── NamedSpiManager                          # 命名 SPI 延迟加载
    ├── ComponentLoader                          # 组件类加载器
    ├── AutoConfigureUtil                        # 配置工具类
    ├── IncubatingUtil                           # 实验性功能工具
    └── EnvironmentResourceProvider              # 环境资源检测
```

### 1. 核心配置类

核心配置类负责整体配置流程和 SDK 组件构建。

#### 1.1 AutoConfiguredOpenTelemetrySdk

**包路径**: `io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk`

**职责**: 不可变包装器，持有配置后的 OpenTelemetrySdk 实例及相关配置信息。

**关键方法**:

```java
public final class AutoConfiguredOpenTelemetrySdk {
    // 静态工厂方法：最简单的初始化方式
    public static AutoConfiguredOpenTelemetrySdk initialize() {
        return builder().build();
    }

    // 获取构建器
    public static AutoConfiguredOpenTelemetrySdkBuilder builder() {
        return new AutoConfiguredOpenTelemetrySdkBuilder();
    }

    // 获取配置后的 SDK 实例
    public OpenTelemetrySdk getOpenTelemetrySdk() {
        return openTelemetrySdk;
    }

    // 获取 Resource 配置
    public Resource getResource() {
        return resource;
    }

    // 获取配置属性
    public ConfigProperties getConfig() {
        return config;
    }

    // 创建包装器（内部使用）
    static AutoConfiguredOpenTelemetrySdk create(
        OpenTelemetrySdk openTelemetrySdk,
        Resource resource,
        ConfigProperties config) {
        return new AutoConfiguredOpenTelemetrySdk(openTelemetrySdk, resource, config);
    }
}
```

**使用示例**:

```java
// 方式 1: 最简单的初始化
OpenTelemetrySdk sdk = AutoConfiguredOpenTelemetrySdk.initialize()
    .getOpenTelemetrySdk();

// 方式 2: 获取额外信息
AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.initialize();

OpenTelemetrySdk sdk = autoConfigured.getOpenTelemetrySdk();
Resource resource = autoConfigured.getResource();
ConfigProperties config = autoConfigured.getConfig();

// 查看配置的服务名称
String serviceName = resource.getAttribute(AttributeKey.stringKey("service.name"));
System.out.println("Service name: " + serviceName);
```

**生命周期管理**:

- `AutoConfiguredOpenTelemetrySdk` 是不可变对象，通常作为应用程序级单例使用
- 内部的 `OpenTelemetrySdk` 会注册 JVM 关闭钩子，自动清理资源
- 如果需要手动控制生命周期，使用 `disableShutdownHook()` 并手动调用 `close()`

#### 1.2 AutoConfiguredOpenTelemetrySdkBuilder

**包路径**: `io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdkBuilder`

**职责**: 主构建器，提供 14+ 定制方法，实现所有配置逻辑。

**核心方法**: `buildImpl()` - 所有配置逻辑的入口点（详见前面章节的源码解析）

**定制器方法详解**:

##### 1.2.1 Resource 相关定制器

```java
// 定制 Resource 属性
public AutoConfiguredOpenTelemetrySdkBuilder addResourceCustomizer(
    BiFunction<? super Resource, ConfigProperties, ? extends Resource> resourceCustomizer)
```

**使用场景**: 添加或修改 Resource 属性（服务信息、部署环境等）

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addResourceCustomizer((resource, config) ->
        resource.toBuilder()
            .put("deployment.environment", "production")
            .put("service.namespace", "my-company")
            .put("service.version", "1.2.3")
            .build())
    .build();
```

##### 1.2.2 Propagator 相关定制器

```java
// 定制传播器
public AutoConfiguredOpenTelemetrySdkBuilder addPropagatorCustomizer(
    BiFunction<? super TextMapPropagator, ConfigProperties, ? extends TextMapPropagator> propagatorCustomizer)
```

**使用场景**: 修改或包装传播器

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addPropagatorCustomizer((propagator, config) ->
        // 添加日志记录
        new LoggingPropagator(propagator))
    .build();
```

##### 1.2.3 Sampler 相关定制器

```java
// 定制采样器
public AutoConfiguredOpenTelemetrySdkBuilder addSamplerCustomizer(
    BiFunction<? super Sampler, ConfigProperties, ? extends Sampler> samplerCustomizer)
```

**使用场景**: 修改采样决策逻辑

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addSamplerCustomizer((sampler, config) ->
        // 包装原采样器，添加额外逻辑
        new ConditionalSampler(sampler))
    .build();
```

##### 1.2.4 Span 导出器相关定制器

```java
// 定制 Span 导出器
public AutoConfiguredOpenTelemetrySdkBuilder addSpanExporterCustomizer(
    BiFunction<? super SpanExporter, ConfigProperties, ? extends SpanExporter> exporterCustomizer)
```

**使用场景**: 包装导出器，添加日志、监控、过滤等逻辑

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addSpanExporterCustomizer((exporter, config) ->
        new LoggingSpanExporter(exporter) {
            @Override
            public CompletableResultCode export(Collection<SpanData> spans) {
                logger.info("Exporting {} spans", spans.size());
                return super.export(spans);
            }
        })
    .build();
```

##### 1.2.5 Span 处理器相关定制器

```java
// 定制 Span 处理器
public AutoConfiguredOpenTelemetrySdkBuilder addSpanProcessorCustomizer(
    BiFunction<? super SpanProcessor, ConfigProperties, ? extends SpanProcessor> processorCustomizer)
```

**使用场景**: 过滤 Span、添加属性、修改处理逻辑

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addSpanProcessorCustomizer((processor, config) ->
        new FilteringSpanProcessor(processor) {
            @Override
            public void onEnd(ReadableSpan span) {
                // 过滤掉内部 Span
                if (!span.getName().startsWith("internal.")) {
                    super.onEnd(span);
                }
            }
        })
    .build();
```

##### 1.2.6 TracerProvider 相关定制器

```java
// 定制 TracerProvider Builder
public AutoConfiguredOpenTelemetrySdkBuilder addTracerProviderCustomizer(
    BiFunction<SdkTracerProviderBuilder, ConfigProperties, SdkTracerProviderBuilder> tracerProviderCustomizer)
```

**使用场景**: 直接修改 TracerProvider Builder

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addTracerProviderCustomizer((builder, config) ->
        builder.setSampler(Sampler.alwaysOn())
               .setSpanLimits(SpanLimits.builder()
                   .setMaxNumberOfAttributes(64)
                   .build()))
    .build();
```

##### 1.2.7 Metric 导出器相关定制器

```java
// 定制 Metric 导出器
public AutoConfiguredOpenTelemetrySdkBuilder addMetricExporterCustomizer(
    BiFunction<? super MetricExporter, ConfigProperties, ? extends MetricExporter> exporterCustomizer)
```

**使用场景**: 包装 Metric 导出器

##### 1.2.8 Metric 读取器相关定制器

```java
// 定制 Metric 读取器
public AutoConfiguredOpenTelemetrySdkBuilder addMetricReaderCustomizer(
    BiFunction<? super MetricReader, ConfigProperties, ? extends MetricReader> readerCustomizer)
```

**使用场景**: 修改 Metric 读取器配置

##### 1.2.9 MeterProvider 相关定制器

```java
// 定制 MeterProvider Builder
public AutoConfiguredOpenTelemetrySdkBuilder addMeterProviderCustomizer(
    BiFunction<SdkMeterProviderBuilder, ConfigProperties, SdkMeterProviderBuilder> meterProviderCustomizer)
```

**使用场景**: 直接修改 MeterProvider Builder

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addMeterProviderCustomizer((builder, config) ->
        builder.registerView(
            InstrumentSelector.builder()
                .setType(InstrumentType.HISTOGRAM)
                .build(),
            View.builder()
                .setAggregation(Aggregation.explicitBucketHistogram(
                    Arrays.asList(0.1, 0.5, 1.0, 5.0, 10.0)))
                .build()))
    .build();
```

##### 1.2.10 日志导出器相关定制器

```java
// 定制日志导出器
public AutoConfiguredOpenTelemetrySdkBuilder addLogRecordExporterCustomizer(
    BiFunction<? super LogRecordExporter, ConfigProperties, ? extends LogRecordExporter> exporterCustomizer)
```

**使用场景**: 包装日志导出器

##### 1.2.11 日志处理器相关定制器

```java
// 定制日志处理器
public AutoConfiguredOpenTelemetrySdkBuilder addLogRecordProcessorCustomizer(
    BiFunction<? super LogRecordProcessor, ConfigProperties, ? extends LogRecordProcessor> processorCustomizer)
```

**使用场景**: 过滤日志记录

##### 1.2.12 LoggerProvider 相关定制器

```java
// 定制 LoggerProvider Builder
public AutoConfiguredOpenTelemetrySdkBuilder addLoggerProviderCustomizer(
    BiFunction<SdkLoggerProviderBuilder, ConfigProperties, SdkLoggerProviderBuilder> loggerProviderCustomizer)
```

**使用场景**: 直接修改 LoggerProvider Builder

##### 1.2.13 配置属性供应器

```java
// 添加默认配置属性
public AutoConfiguredOpenTelemetrySdkBuilder addPropertiesSupplier(
    Supplier<Map<String, String>> propertiesSupplier)
```

**使用场景**: 提供默认配置值（优先级低于环境变量）

**示例**:
```java
Map<String, String> defaults = new HashMap<>();
defaults.put("otel.service.name", "my-service");
defaults.put("otel.traces.sampler", "parentbased_traceidratio");
defaults.put("otel.traces.sampler.arg", "0.1");

AutoConfiguredOpenTelemetrySdk.builder()
    .addPropertiesSupplier(() -> defaults)
    .build();
```

##### 1.2.14 配置属性定制器

```java
// 定制配置属性
public AutoConfiguredOpenTelemetrySdkBuilder addPropertiesCustomizer(
    Function<ConfigProperties, Map<String, String>> propertiesCustomizer)
```

**使用场景**: 动态修改或覆盖配置属性

**示例**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addPropertiesCustomizer(props -> {
        Map<String, String> overrides = new HashMap<>();
        // 根据条件覆盖配置
        if (isProduction()) {
            overrides.put("otel.traces.sampler", "traceidratio");
            overrides.put("otel.traces.sampler.arg", "0.01");
        }
        return overrides;
    })
    .build();
```

**定制器执行顺序**:

```
buildImpl() 执行流程
├── 1. 声明式配置尝试
├── 2. SPI 和定制器初始化
├── 3. 配置属性加载
│   ├── 应用 propertiesSupplier
│   └── 应用 propertiesCustomizer
├── 4. 配置 Resource
│   └── 应用 resourceCustomizer
├── 5. 配置 SDK 组件
│   ├── 配置 Propagators → 应用 propagatorCustomizer
│   ├── 配置 MeterProvider
│   │   ├── 应用 metricExporterCustomizer
│   │   ├── 应用 metricReaderCustomizer
│   │   └── 应用 meterProviderCustomizer
│   ├── 配置 TracerProvider
│   │   ├── 应用 samplerCustomizer
│   │   ├── 应用 spanExporterCustomizer
│   │   ├── 应用 spanProcessorCustomizer
│   │   └── 应用 tracerProviderCustomizer
│   └── 配置 LoggerProvider
│       ├── 应用 logRecordExporterCustomizer
│       ├── 应用 logRecordProcessorCustomizer
│       └── 应用 loggerProviderCustomizer
└── 6. 构建并返回 SDK
```

#### 1.3 ResourceConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.ResourceConfiguration`

**职责**: 配置 Resource（服务元数据），包括加载 ResourceProvider SPI、合并 Resource、应用定制器。

**核心方法**:

```java
final class ResourceConfiguration {
    /**
     * 配置 Resource
     *
     * @param config 配置属性
     * @param spiHelper SPI 助手
     * @param resourceCustomizer Resource 定制器
     * @return 配置后的 Resource
     */
    static Resource configureResource(
        ConfigProperties config,
        SpiHelper spiHelper,
        BiFunction<? super Resource, ConfigProperties, ? extends Resource> resourceCustomizer) {

        // 1. 从环境变量创建 Resource
        Resource result = Resource.getDefault();

        // 2. 加载所有 ResourceProvider SPI（按 order() 排序）
        for (ResourceProvider resourceProvider : spiHelper.loadOrdered(ResourceProvider.class)) {
            Resource providerResource = resourceProvider.createResource(config);
            result = result.merge(providerResource);
        }

        // 3. 从 OTEL_RESOURCE_ATTRIBUTES 读取属性
        String resourceAttributes = config.getString("otel.resource.attributes");
        if (resourceAttributes != null) {
            result = result.merge(parseResourceAttributes(resourceAttributes));
        }

        // 4. 应用定制器
        result = resourceCustomizer.apply(result, config);

        return result;
    }
}
```

**配置流程**:

```
configureResource()
│
├── 1. Resource.getDefault()
│   └── 返回 SDK 默认 Resource（包含 SDK 版本等）
│
├── 2. 加载 ResourceProvider SPI
│   ├── OsResourceProvider (操作系统)
│   ├── ProcessResourceProvider (进程 PID)
│   ├── ProcessRuntimeResourceProvider (Java 运行时)
│   ├── HostResourceProvider (主机名)
│   ├── ContainerResourceProvider (容器 ID)
│   └── 自定义 ResourceProvider
│
├── 3. 解析 OTEL_RESOURCE_ATTRIBUTES
│   └── 格式: key1=val1,key2=val2
│
├── 4. 应用 resourceCustomizer
│   └── 用户自定义逻辑
│
└── 5. 返回最终 Resource
```

**相关环境变量**:

| 环境变量 | 说明 | 示例 |
|---------|------|------|
| `OTEL_SERVICE_NAME` | 服务名称 | `my-service` |
| `OTEL_RESOURCE_ATTRIBUTES` | Resource 属性（逗号分隔） | `environment=prod,region=us-west` |
| `OTEL_JAVA_ENABLED_RESOURCE_PROVIDERS` | 启用的 ResourceProvider 类名 | `com.example.MyResourceProvider` |
| `OTEL_JAVA_DISABLED_RESOURCE_PROVIDERS` | 禁用的 ResourceProvider 类名 | `io.opentelemetry.sdk.extension.resources.HostResourceProvider` |

**Resource 合并规则**:

- 后加载的 Resource 会覆盖前面的同名属性
- `ResourceProvider` 按 `order()` 值从小到大执行
- `OTEL_RESOURCE_ATTRIBUTES` 在所有 ResourceProvider 之后应用
- `resourceCustomizer` 最后执行，优先级最高

**示例 Resource**:

```json
{
  "service.name": "my-service",
  "service.version": "1.0.0",
  "deployment.environment": "production",
  "host.name": "server-01",
  "host.arch": "amd64",
  "os.type": "linux",
  "os.description": "Linux 5.10.0",
  "process.pid": "12345",
  "process.runtime.name": "OpenJDK Runtime Environment",
  "process.runtime.version": "17.0.1",
  "telemetry.sdk.name": "opentelemetry",
  "telemetry.sdk.language": "java",
  "telemetry.sdk.version": "1.35.0"
}
```

#### 1.4 TracerProviderConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.TracerProviderConfiguration`

**职责**: 配置 TracerProvider（Traces 信号），包括采样器、导出器、处理器。

**核心方法**:

```java
final class TracerProviderConfiguration {
    /**
     * 配置 TracerProvider
     */
    static void configureTracerProvider(
        SdkTracerProviderBuilder tracerProviderBuilder,
        ConfigProperties config,
        SpiHelper spiHelper,
        MeterProvider meterProvider,
        BiFunction<? super SpanExporter, ConfigProperties, ? extends SpanExporter> exporterCustomizer,
        BiFunction<? super SpanProcessor, ConfigProperties, ? extends SpanProcessor> processorCustomizer,
        BiFunction<? super Sampler, ConfigProperties, ? extends Sampler> samplerCustomizer,
        List<Closeable> closeables) {

        // 1. 配置采样器
        Sampler sampler = configureSampler(config, spiHelper);
        sampler = samplerCustomizer.apply(sampler, config);
        tracerProviderBuilder.setSampler(sampler);

        // 2. 配置 Span 限制
        SpanLimits spanLimits = configureSpanLimits(config);
        tracerProviderBuilder.setSpanLimits(spanLimits);

        // 3. 配置 Span 导出器
        List<SpanExporter> exporters = SpanExporterConfiguration.configureSpanExporters(
            config, spiHelper, meterProvider, closeables);

        for (SpanExporter exporter : exporters) {
            exporter = exporterCustomizer.apply(exporter, config);

            // 4. 创建批处理器
            SpanProcessor processor = configureBatchSpanProcessor(config, exporter, meterProvider);
            processor = processorCustomizer.apply(processor, config);

            tracerProviderBuilder.addSpanProcessor(processor);
            closeables.add(processor);
        }
    }
}
```

**配置流程**:

```
configureTracerProvider()
│
├── 1. 配置采样器
│   ├── 读取 OTEL_TRACES_SAMPLER
│   ├── 加载 ConfigurableSamplerProvider SPI
│   ├── 应用 samplerCustomizer
│   └── 默认: parentbased_always_on
│
├── 2. 配置 Span 限制
│   ├── OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT
│   ├── OTEL_SPAN_EVENT_COUNT_LIMIT
│   ├── OTEL_SPAN_LINK_COUNT_LIMIT
│   └── OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT
│
├── 3. 配置 Span 导出器
│   ├── 读取 OTEL_TRACES_EXPORTER（支持多个）
│   ├── 加载 ConfigurableSpanExporterProvider SPI
│   └── 应用 exporterCustomizer
│
├── 4. 为每个导出器创建批处理器
│   ├── OTEL_BSP_SCHEDULE_DELAY
│   ├── OTEL_BSP_MAX_QUEUE_SIZE
│   ├── OTEL_BSP_MAX_EXPORT_BATCH_SIZE
│   ├── OTEL_BSP_EXPORT_TIMEOUT
│   └── 应用 processorCustomizer
│
└── 5. 添加到 TracerProviderBuilder
```

**相关环境变量**:

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_TRACES_EXPORTER` | String | `otlp` | 导出器名称（可多个，逗号分隔） |
| `OTEL_TRACES_SAMPLER` | String | `parentbased_always_on` | 采样器名称 |
| `OTEL_TRACES_SAMPLER_ARG` | Double | - | 采样器参数（如采样率） |
| `OTEL_BSP_SCHEDULE_DELAY` | Duration | `5000ms` | 批处理调度延迟 |
| `OTEL_BSP_MAX_QUEUE_SIZE` | Integer | `2048` | 最大队列大小 |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | Integer | `512` | 每批最大 Span 数 |
| `OTEL_BSP_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | Integer | `128` | Span 最大属性数 |
| `OTEL_SPAN_EVENT_COUNT_LIMIT` | Integer | `128` | Span 最大事件数 |
| `OTEL_SPAN_LINK_COUNT_LIMIT` | Integer | `128` | Span 最大链接数 |

**内置采样器**:

- `always_on` - 100% 采样
- `always_off` - 0% 采样
- `traceidratio` - 基于 TraceId 的比例采样
- `parentbased_always_on` - 继承父 Span，根 Span 100% 采样
- `parentbased_always_off` - 继承父 Span，根 Span 0% 采样
- `parentbased_traceidratio` - 继承父 Span，根 Span 比例采样

#### 1.5 MeterProviderConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.MeterProviderConfiguration`

**职责**: 配置 MeterProvider（Metrics 信号），包括导出器、读取器。

**核心方法**:

```java
final class MeterProviderConfiguration {
    /**
     * 配置 MeterProvider
     */
    static void configureMeterProvider(
        SdkMeterProviderBuilder meterProviderBuilder,
        ConfigProperties config,
        SpiHelper spiHelper,
        BiFunction<? super MetricReader, ConfigProperties, ? extends MetricReader> metricReaderCustomizer,
        BiFunction<? super MetricExporter, ConfigProperties, ? extends MetricExporter> metricExporterCustomizer,
        List<Closeable> closeables) {

        // 1. 配置 MetricReader
        List<MetricReader> readers = MetricExporterConfiguration.configureMetricReaders(
            config, spiHelper, metricExporterCustomizer, closeables);

        for (MetricReader reader : readers) {
            reader = metricReaderCustomizer.apply(reader, config);
            meterProviderBuilder.registerMetricReader(reader);
        }

        // 2. 配置 Exemplar Filter
        meterProviderBuilder.setExemplarFilter(configureExemplarFilter(config));

        // 3. 配置基数限制
        int cardinalityLimit = config.getInt("otel.java.metrics.cardinality.limit", 2000);
        meterProviderBuilder.setCardinalityLimit(cardinalityLimit);
    }
}
```

**配置流程**:

```
configureMeterProvider()
│
├── 1. 配置 MetricReader
│   ├── 读取 OTEL_METRICS_EXPORTER（支持多个）
│   ├── 加载 ConfigurableMetricExporterProvider SPI
│   ├── 创建 PeriodicMetricReader（Push 模式）
│   │   ├── OTEL_METRIC_EXPORT_INTERVAL
│   │   └── OTEL_METRIC_EXPORT_TIMEOUT
│   ├── 或加载 ConfigurableMetricReaderProvider（Pull 模式）
│   └── 应用 metricReaderCustomizer
│
├── 2. 配置 Exemplar Filter
│   ├── trace_based（默认）
│   ├── always_on
│   └── always_off
│
└── 3. 配置基数限制
    └── OTEL_JAVA_METRICS_CARDINALITY_LIMIT
```

**相关环境变量**:

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_METRICS_EXPORTER` | String | `otlp` | 导出器名称（可多个） |
| `OTEL_METRIC_EXPORT_INTERVAL` | Duration | `60000ms` | 导出间隔 |
| `OTEL_METRIC_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |
| `OTEL_METRICS_EXEMPLAR_FILTER` | String | `trace_based` | Exemplar 过滤器 |
| `OTEL_JAVA_METRICS_CARDINALITY_LIMIT` | Integer | `2000` | 基数限制 |

**Push vs Pull 模式**:

**Push 模式**（主动导出）:
- 使用 `PeriodicMetricReader` 周期性导出
- 适用于 OTLP、Logging 等导出器
- 配置项: `OTEL_METRIC_EXPORT_INTERVAL`

**Pull 模式**（被动拉取）:
- 使用 `PrometheusHttpServer` 等
- 适用于 Prometheus
- 配置项: `OTEL_EXPORTER_PROMETHEUS_PORT`, `OTEL_EXPORTER_PROMETHEUS_HOST`

**示例**:
```bash
# Push 模式（OTLP）
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=30000

# Pull 模式（Prometheus）
export OTEL_METRICS_EXPORTER=prometheus
export OTEL_EXPORTER_PROMETHEUS_PORT=9464

# 同时使用
export OTEL_METRICS_EXPORTER=otlp,prometheus
```

#### 1.6 LoggerProviderConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.LoggerProviderConfiguration`

**职责**: 配置 LoggerProvider（Logs 信号），包括日志导出器、处理器。

**核心方法**:

```java
final class LoggerProviderConfiguration {
    /**
     * 配置 LoggerProvider
     */
    static void configureLoggerProvider(
        SdkLoggerProviderBuilder loggerProviderBuilder,
        ConfigProperties config,
        SpiHelper spiHelper,
        MeterProvider meterProvider,
        BiFunction<? super LogRecordExporter, ConfigProperties, ? extends LogRecordExporter> exporterCustomizer,
        BiFunction<? super LogRecordProcessor, ConfigProperties, ? extends LogRecordProcessor> processorCustomizer,
        List<Closeable> closeables) {

        // 1. 配置日志导出器
        List<LogRecordExporter> exporters = LogRecordExporterConfiguration.configureExporters(
            config, spiHelper, closeables);

        for (LogRecordExporter exporter : exporters) {
            exporter = exporterCustomizer.apply(exporter, config);

            // 2. 创建批处理器
            LogRecordProcessor processor = configureBatchLogRecordProcessor(
                config, exporter, meterProvider);
            processor = processorCustomizer.apply(processor, config);

            loggerProviderBuilder.addLogRecordProcessor(processor);
            closeables.add(processor);
        }

        // 3. 配置日志限制
        loggerProviderBuilder.setLogLimits(configureLogLimits(config));
    }
}
```

**配置流程**:

```
configureLoggerProvider()
│
├── 1. 配置日志导出器
│   ├── 读取 OTEL_LOGS_EXPORTER
│   ├── 加载 ConfigurableLogRecordExporterProvider SPI
│   └── 应用 exporterCustomizer
│
├── 2. 为每个导出器创建批处理器
│   ├── OTEL_BLRP_SCHEDULE_DELAY
│   ├── OTEL_BLRP_MAX_QUEUE_SIZE
│   ├── OTEL_BLRP_MAX_EXPORT_BATCH_SIZE
│   ├── OTEL_BLRP_EXPORT_TIMEOUT
│   └── 应用 processorCustomizer
│
└── 3. 配置日志限制
    └── OTEL_ATTRIBUTE_COUNT_LIMIT
```

**相关环境变量**:

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_LOGS_EXPORTER` | String | `otlp` | 日志导出器名称 |
| `OTEL_BLRP_SCHEDULE_DELAY` | Duration | `1000ms` | 批处理调度延迟 |
| `OTEL_BLRP_MAX_QUEUE_SIZE` | Integer | `2048` | 最大队列大小 |
| `OTEL_BLRP_MAX_EXPORT_BATCH_SIZE` | Integer | `512` | 每批最大日志数 |
| `OTEL_BLRP_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |

### 2. 组件工厂类

组件工厂类负责创建特定类型的 SDK 组件，使用 `NamedSpiManager` 实现延迟加载和名称映射。

#### 2.1 SpanExporterConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.SpanExporterConfiguration`

**职责**: 创建 SpanExporter 实例，支持多导出器配置。

**核心方法**:

```java
final class SpanExporterConfiguration {
    /**
     * 配置 Span 导出器
     *
     * @return 导出器列表（支持多个）
     */
    static List<SpanExporter> configureSpanExporters(
        ConfigProperties config,
        SpiHelper spiHelper,
        MeterProvider meterProvider,
        List<Closeable> closeables) {

        // 读取 OTEL_TRACES_EXPORTER
        List<String> exporterNames = config.getList("otel.traces.exporter", Arrays.asList("otlp"));

        // 特殊值 "none" 表示不使用导出器
        if (exporterNames.contains("none")) {
            return Collections.emptyList();
        }

        // 创建 NamedSpiManager
        NamedSpiManager<SpanExporter> spiExportersManager =
            spiHelper.loadConfigured(
                ConfigurableSpanExporterProvider.class,
                ConfigurableSpanExporterProvider::getName,
                ConfigurableSpanExporterProvider::createExporter,
                config);

        // 为每个名称创建导出器
        List<SpanExporter> exporters = new ArrayList<>();
        for (String exporterName : exporterNames) {
            SpanExporter exporter = spiExportersManager.getByName(exporterName);
            if (exporter == null) {
                throw new ConfigurationException("Unrecognized value for otel.traces.exporter: " + exporterName);
            }
            exporters.add(exporter);
            closeables.add(exporter);
        }

        return exporters;
    }
}
```

**NamedSpiManager 使用模式**:

```
读取配置
OTEL_TRACES_EXPORTER=otlp,zipkin
    ↓
解析为名称列表
["otlp", "zipkin"]
    ↓
NamedSpiManager.getByName("otlp")
    ├── 检查缓存
    ├── ServiceLoader.load(ConfigurableSpanExporterProvider)
    ├── 遍历所有 Provider
    ├── 找到 getName() == "otlp" 的 Provider
    ├── 调用 createExporter(config)
    ├── 缓存实例
    └── 返回 SpanExporter
    ↓
NamedSpiManager.getByName("zipkin")
    └── （同样流程）
    ↓
返回 [OtlpSpanExporter, ZipkinSpanExporter]
```

**内置导出器**:

| 名称 | 类 | 说明 |
|------|---|------|
| `otlp` | `OtlpGrpcSpanExporter` / `OtlpHttpSpanExporter` | OTLP 协议（默认） |
| `zipkin` | `ZipkinSpanExporter` | Zipkin 格式 |
| `logging` | `LoggingSpanExporter` | 输出到控制台 |
| `none` | - | 禁用导出 |

**多导出器配置**:

```bash
# 同时导出到 OTLP 和 Zipkin
export OTEL_TRACES_EXPORTER=otlp,zipkin

# OTLP 配置
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Zipkin 配置
export OTEL_EXPORTER_ZIPKIN_ENDPOINT=http://localhost:9411/api/v2/spans
```

#### 2.2 MetricExporterConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.MetricExporterConfiguration`

**职责**: 创建 MetricExporter 和 MetricReader 实例，支持 Push 和 Pull 模式。

**核心方法**:

```java
final class MetricExporterConfiguration {
    /**
     * 配置 Metric 读取器
     *
     * @return MetricReader 列表
     */
    static List<MetricReader> configureMetricReaders(
        ConfigProperties config,
        SpiHelper spiHelper,
        BiFunction<? super MetricExporter, ConfigProperties, ? extends MetricExporter> exporterCustomizer,
        List<Closeable> closeables) {

        List<String> exporterNames = config.getList("otel.metrics.exporter", Arrays.asList("otlp"));

        if (exporterNames.contains("none")) {
            return Collections.emptyList();
        }

        // 1. 尝试加载 ConfigurableMetricReaderProvider (Pull 模式)
        NamedSpiManager<MetricReader> readerManager =
            spiHelper.loadConfigured(
                ConfigurableMetricReaderProvider.class,
                ConfigurableMetricReaderProvider::getName,
                ConfigurableMetricReaderProvider::createMetricReader,
                config);

        // 2. 加载 ConfigurableMetricExporterProvider (Push 模式)
        NamedSpiManager<MetricExporter> exporterManager =
            spiHelper.loadConfigured(
                ConfigurableMetricExporterProvider.class,
                ConfigurableMetricExporterProvider::getName,
                ConfigurableMetricExporterProvider::createExporter,
                config);

        List<MetricReader> readers = new ArrayList<>();

        for (String exporterName : exporterNames) {
            // 先尝试 MetricReader（Pull 模式）
            MetricReader reader = readerManager.getByName(exporterName);
            if (reader != null) {
                readers.add(reader);
                closeables.add(reader);
                continue;
            }

            // 再尝试 MetricExporter（Push 模式）
            MetricExporter exporter = exporterManager.getByName(exporterName);
            if (exporter != null) {
                exporter = exporterCustomizer.apply(exporter, config);

                // 包装为 PeriodicMetricReader
                long exportInterval = config.getDuration("otel.metric.export.interval", 60000);
                reader = PeriodicMetricReader.builder(exporter)
                    .setInterval(Duration.ofMillis(exportInterval))
                    .build();

                readers.add(reader);
                closeables.add(reader);
            } else {
                throw new ConfigurationException(
                    "Unrecognized value for otel.metrics.exporter: " + exporterName);
            }
        }

        return readers;
    }
}
```

**Push vs Pull 决策流程**:

```
读取 OTEL_METRICS_EXPORTER
    ↓
遍历每个导出器名称
    ↓
    ├── 步骤 1: 尝试 ConfigurableMetricReaderProvider
    │   └── 找到 → 使用 MetricReader（Pull 模式）
    │       └── 示例: PrometheusHttpServer
    │
    └── 步骤 2: 尝试 ConfigurableMetricExporterProvider
        └── 找到 → 创建 PeriodicMetricReader（Push 模式）
            └── 示例: OtlpGrpcMetricExporter + PeriodicMetricReader
```

**内置导出器/读取器**:

| 名称 | 模式 | 类 | 说明 |
|------|-----|---|------|
| `otlp` | Push | `OtlpGrpcMetricExporter` | OTLP 协议 |
| `prometheus` | Pull | `PrometheusHttpServer` | Prometheus 拉取 |
| `logging` | Push | `LoggingMetricExporter` | 输出到控制台 |
| `none` | - | - | 禁用导出 |

**配置示例**:

```bash
# Push 模式（OTLP）
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=30000
export OTEL_EXPORTER_OTLP_METRICS_ENDPOINT=http://localhost:4317

# Pull 模式（Prometheus）
export OTEL_METRICS_EXPORTER=prometheus
export OTEL_EXPORTER_PROMETHEUS_PORT=9464
export OTEL_EXPORTER_PROMETHEUS_HOST=0.0.0.0

# 混合模式
export OTEL_METRICS_EXPORTER=otlp,prometheus
```

#### 2.3 LogRecordExporterConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.LogRecordExporterConfiguration`

**职责**: 创建 LogRecordExporter 实例。

**核心方法**:

```java
final class LogRecordExporterConfiguration {
    /**
     * 配置日志导出器
     */
    static List<LogRecordExporter> configureExporters(
        ConfigProperties config,
        SpiHelper spiHelper,
        List<Closeable> closeables) {

        List<String> exporterNames = config.getList("otel.logs.exporter", Arrays.asList("otlp"));

        if (exporterNames.contains("none")) {
            return Collections.emptyList();
        }

        NamedSpiManager<LogRecordExporter> exporterManager =
            spiHelper.loadConfigured(
                ConfigurableLogRecordExporterProvider.class,
                ConfigurableLogRecordExporterProvider::getName,
                ConfigurableLogRecordExporterProvider::createExporter,
                config);

        List<LogRecordExporter> exporters = new ArrayList<>();
        for (String exporterName : exporterNames) {
            LogRecordExporter exporter = exporterManager.getByName(exporterName);
            if (exporter == null) {
                throw new ConfigurationException(
                    "Unrecognized value for otel.logs.exporter: " + exporterName);
            }
            exporters.add(exporter);
            closeables.add(exporter);
        }

        return exporters;
    }
}
```

**内置导出器**:

| 名称 | 类 | 说明 |
|------|---|------|
| `otlp` | `OtlpHttpLogRecordExporter` | OTLP 协议 |
| `logging` | `SystemOutLogRecordExporter` | 输出到控制台 |
| `none` | - | 禁用导出 |

#### 2.4 PropagatorConfiguration

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.PropagatorConfiguration`

**职责**: 创建 TextMapPropagator 实例，支持组合传播器。

**核心方法**:

```java
final class PropagatorConfiguration {
    /**
     * 配置传播器
     */
    static ContextPropagators configurePropagators(
        ConfigProperties config,
        SpiHelper spiHelper,
        BiFunction<? super TextMapPropagator, ConfigProperties, ? extends TextMapPropagator> propagatorCustomizer) {

        // 读取 OTEL_PROPAGATORS
        List<String> propagatorNames = config.getList(
            "otel.propagators",
            Arrays.asList("tracecontext", "baggage"));

        NamedSpiManager<TextMapPropagator> propagatorManager =
            spiHelper.loadConfigured(
                ConfigurablePropagatorProvider.class,
                ConfigurablePropagatorProvider::getName,
                ConfigurablePropagatorProvider::getPropagator,
                config);

        // 加载每个传播器
        List<TextMapPropagator> propagators = new ArrayList<>();
        for (String propagatorName : propagatorNames) {
            TextMapPropagator propagator = propagatorManager.getByName(propagatorName);
            if (propagator == null) {
                throw new ConfigurationException(
                    "Unrecognized value for otel.propagators: " + propagatorName);
            }
            propagators.add(propagator);
        }

        // 组合所有传播器
        TextMapPropagator compositePropagator;
        if (propagators.isEmpty()) {
            compositePropagator = TextMapPropagator.noop();
        } else if (propagators.size() == 1) {
            compositePropagator = propagators.get(0);
        } else {
            compositePropagator = TextMapPropagator.composite(
                propagators.toArray(new TextMapPropagator[0]));
        }

        // 应用定制器
        compositePropagator = propagatorCustomizer.apply(compositePropagator, config);

        return ContextPropagators.create(compositePropagator);
    }
}
```

**内置传播器**:

| 名称 | 类 | 说明 |
|------|---|------|
| `tracecontext` | `W3CTraceContextPropagator` | W3C Trace Context（推荐） |
| `baggage` | `W3CBaggagePropagator` | W3C Baggage |
| `b3` | `B3Propagator` | B3 Single Header |
| `b3multi` | `B3Propagator` | B3 Multi Header |
| `jaeger` | `JaegerPropagator` | Jaeger 格式 |
| `ottrace` | `OtTracePropagator` | OT Trace |

**组合传播器示例**:

```bash
# 使用 W3C Trace Context + Baggage
export OTEL_PROPAGATORS=tracecontext,baggage

# 使用 B3 + Jaeger（多协议兼容）
export OTEL_PROPAGATORS=b3,jaeger

# 使用单个传播器
export OTEL_PROPAGATORS=tracecontext
```

**传播器工作原理**:

```
出站请求（Inject）
    ↓
TextMapPropagator.inject(context, carrier, setter)
    ├── W3CTraceContextPropagator
    │   ├── 设置 traceparent: 00-<trace-id>-<span-id>-01
    │   └── 设置 tracestate: ...
    ├── W3CBaggagePropagator
    │   └── 设置 baggage: key1=value1,key2=value2
    └── B3Propagator
        └── 设置 X-B3-TraceId, X-B3-SpanId, ...
    ↓
HTTP Headers 包含所有传播信息

入站请求（Extract）
    ↓
TextMapPropagator.extract(context, carrier, getter)
    ├── W3CTraceContextPropagator
    │   └── 从 traceparent 提取 SpanContext
    ├── W3CBaggagePropagator
    │   └── 从 baggage 提取 Baggage
    └── B3Propagator
        └── 从 X-B3-* 提取 SpanContext
    ↓
Context 包含提取的 SpanContext 和 Baggage
```

### 3. 辅助类

辅助类提供核心功能支持，包括 SPI 加载、配置工具等。

#### 3.1 SpiHelper

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.SpiHelper`

**职责**: SPI 加载和生命周期管理，是 ServiceLoader 的包装器。

**核心方法**:

```java
final class SpiHelper {
    /**
     * 加载 SPI 实现（无序）
     */
    public <T> Iterable<T> load(Class<T> spiClass) {
        return componentLoader.load(spiClass);
    }

    /**
     * 加载 SPI 实现（按 Ordered 接口排序）
     */
    public <T extends Ordered> List<T> loadOrdered(Class<T> spiClass) {
        List<T> result = new ArrayList<>();
        for (T implementation : componentLoader.load(spiClass)) {
            result.add(implementation);
        }

        // 按 order() 值排序（升序）
        result.sort(Comparator.comparingInt(Ordered::order));

        return result;
    }

    /**
     * 加载命名 SPI（用于 NamedSpiManager）
     */
    public <T, C> NamedSpiManager<C> loadConfigured(
        Class<T> providerClass,
        Function<T, String> getName,
        BiFunction<T, ConfigProperties, C> createComponent,
        ConfigProperties config) {

        return NamedSpiManager.create(this, providerClass, getName, createComponent, config);
    }

    /**
     * 获取 ComponentLoader
     */
    public ComponentLoader getComponentLoader() {
        return componentLoader;
    }

    /**
     * 获取 AutoConfigureListener 列表
     */
    public List<AutoConfigureListener> getListeners() {
        List<AutoConfigureListener> listeners = new ArrayList<>();
        for (AutoConfigureListener listener : componentLoader.load(AutoConfigureListener.class)) {
            listeners.add(listener);
        }
        return listeners;
    }
}
```

**Ordered 接口排序**:

```java
// 示例：三个 ResourceProvider
public class Provider1 implements ResourceProvider {
    @Override public int order() { return -100; }  // 最早执行
}

public class Provider2 implements ResourceProvider {
    @Override public int order() { return 0; }     // 中间执行（默认）
}

public class Provider3 implements ResourceProvider {
    @Override public int order() { return 100; }   // 最晚执行（优先级最高）
}

// SpiHelper.loadOrdered(ResourceProvider.class)
// 返回顺序: [Provider1, Provider2, Provider3]
// Resource 合并: Resource1 → Resource2 → Resource3
// Provider3 的属性会覆盖 Provider1 和 Provider2
```

#### 3.2 NamedSpiManager

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.NamedSpiManager`

**职责**: 命名 SPI 的延迟加载管理器，根据名称查找和创建 SPI 实例。

**核心方法**:

```java
final class NamedSpiManager<T> {
    /**
     * 根据名称获取组件（延迟加载）
     */
    @Nullable
    public T getByName(String name) {
        // 1. 检查缓存
        T cached = cache.get(name);
        if (cached != null) {
            return cached;
        }

        // 2. 加载所有 Provider
        for (P provider : spiHelper.load(providerClass)) {
            String providerName = getName.apply(provider);

            // 3. 找到匹配名称的 Provider
            if (providerName.equals(name)) {
                // 4. 创建组件
                T component = createComponent.apply(provider, config);

                // 5. 缓存
                cache.put(name, component);

                return component;
            }
        }

        // 6. 未找到
        return null;
    }

    /**
     * 获取所有可用的名称
     */
    public Set<String> getNames() {
        Set<String> names = new HashSet<>();
        for (P provider : spiHelper.load(providerClass)) {
            names.add(getName.apply(provider));
        }
        return names;
    }
}
```

**延迟加载设计**:

```
时间轴
    │
    ├── T1: 创建 NamedSpiManager
    │   └── 仅保存配置，不加载任何 SPI
    │
    ├── T2: 第一次调用 getByName("otlp")
    │   ├── ServiceLoader.load(ConfigurableSpanExporterProvider)
    │   ├── 遍历所有 Provider
    │   ├── 找到 getName() == "otlp" 的 Provider
    │   ├── 调用 createExporter(config)
    │   ├── 缓存 "otlp" → OtlpSpanExporter
    │   └── 返回 OtlpSpanExporter
    │
    └── T3: 第二次调用 getByName("otlp")
        ├── 从缓存读取
        └── 直接返回 OtlpSpanExporter（不重新创建）
```

**优势**:

1. **按需加载**: 只加载实际使用的组件
2. **避免浪费**: 不创建未配置的导出器
3. **性能优化**: 缓存避免重复创建
4. **失败快速**: 配置错误时立即抛出异常

#### 3.3 ComponentLoader

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.ComponentLoader`

**职责**: 组件类加载器，是 ServiceLoader 的抽象。

**核心方法**:

```java
interface ComponentLoader {
    /**
     * 加载 SPI 实现
     */
    <T> Iterable<T> load(Class<T> spiClass);

    /**
     * 创建默认 ComponentLoader
     */
    static ComponentLoader defaultComponentLoader() {
        return forClassLoader(ComponentLoader.class.getClassLoader());
    }

    /**
     * 为指定 ClassLoader 创建 ComponentLoader
     */
    static ComponentLoader forClassLoader(ClassLoader classLoader) {
        return new ServiceLoaderComponentLoader(classLoader);
    }
}

// 实现类
final class ServiceLoaderComponentLoader implements ComponentLoader {
    private final ClassLoader classLoader;

    @Override
    public <T> Iterable<T> load(Class<T> spiClass) {
        return ServiceLoader.load(spiClass, classLoader);
    }
}
```

**使用场景**:

- 支持自定义 ClassLoader（如 OSGi、模块化系统）
- 测试时使用 mock ComponentLoader
- 隔离不同模块的 SPI 实现

#### 3.4 AutoConfigureUtil

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.AutoConfigureUtil`

**职责**: 配置工具类，提供各种配置辅助方法。

**核心方法**:

```java
final class AutoConfigureUtil {
    /**
     * 解析时间间隔字符串
     */
    static long parseDuration(String value) {
        // 支持: "5000", "5s", "5000ms"
    }

    /**
     * 解析布尔值
     */
    static boolean parseBoolean(String value) {
        // 支持: "true", "false", "1", "0"
    }

    /**
     * 解析 Map（key1=val1,key2=val2 格式）
     */
    static Map<String, String> parseMap(String value) {
        // 示例: "environment=prod,region=us" → {"environment": "prod", "region": "us"}
    }

    /**
     * 解析列表（逗号分隔）
     */
    static List<String> parseList(String value) {
        // 示例: "otlp,zipkin,logging" → ["otlp", "zipkin", "logging"]
    }
}
```

#### 3.5 IncubatingUtil

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.IncubatingUtil`

**职责**: 实验性功能工具，支持声明式配置（YAML 文件）。

**核心方法**:

```java
final class IncubatingUtil {
    /**
     * 从 YAML 文件配置 SDK
     */
    static AutoConfiguredOpenTelemetrySdk configureFromFile(
        Logger logger,
        String configurationFile,
        ComponentLoader componentLoader) {

        try (FileInputStream inputStream = new FileInputStream(configurationFile)) {
            return DeclarativeConfiguration.parseAndCreate(inputStream);
        } catch (IOException e) {
            throw new ConfigurationException("Failed to load configuration file: " + configurationFile, e);
        }
    }

    /**
     * 从 SPI 配置（experimental.sdk.config.provider）
     */
    @Nullable
    static AutoConfiguredOpenTelemetrySdk configureFromSpi(ComponentLoader componentLoader) {
        // 加载 OpenTelemetrySdkConfigProvider SPI
        // 用于从自定义源加载配置
    }
}
```

**YAML 配置示例**:

```yaml
file_format: "1.0"

resource:
  attributes:
    service.name: my-service
    deployment.environment: production

tracer_provider:
  sampler:
    parent_based:
      root:
        trace_id_ratio_based:
          ratio: 0.1
  processors:
    - batch:
        schedule_delay: 5000
        max_queue_size: 2048
        max_export_batch_size: 512
        exporter: otlp

exporters:
  otlp:
    endpoint: http://localhost:4317
    timeout: 10000
    compression: gzip
```

**使用方式**:

```bash
# 通过环境变量指定配置文件
export OTEL_EXPERIMENTAL_CONFIG_FILE=/path/to/config.yaml

# Java 代码会自动检测并使用
```

#### 3.6 EnvironmentResourceProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.internal.EnvironmentResourceProvider`

**职责**: 环境资源属性检测，自动识别运行环境。

**检测的资源类型**:

1. **操作系统信息** (`OsResourceProvider`):
   - `os.type` - 操作系统类型（linux, windows, darwin）
   - `os.description` - 操作系统描述

2. **进程信息** (`ProcessResourceProvider`, `ProcessRuntimeResourceProvider`):
   - `process.pid` - 进程 ID
   - `process.executable.name` - 可执行文件名
   - `process.command_line` - 命令行
   - `process.runtime.name` - 运行时名称（Java）
   - `process.runtime.version` - 运行时版本

3. **主机信息** (`HostResourceProvider`):
   - `host.name` - 主机名
   - `host.arch` - 主机架构（amd64, arm64）

4. **容器信息** (`ContainerResourceProvider`):
   - `container.id` - 容器 ID（从 cgroup 读取）

**实现示例**:

```java
public class HostResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        String hostname;
        try {
            hostname = InetAddress.getLocalHost().getHostName();
        } catch (UnknownHostException e) {
            hostname = "unknown";
        }

        return Resource.create(Attributes.of(
            AttributeKey.stringKey("host.name"), hostname,
            AttributeKey.stringKey("host.arch"), System.getProperty("os.arch")
        ));
    }
}
```

### 4. 组件关系图

#### 4.1 类依赖关系图

```
┌─────────────────────────────────────────────────────────────┐
│                 AutoConfiguredOpenTelemetrySdk               │
│                     (不可变包装器)                            │
│  - OpenTelemetrySdk openTelemetrySdk                        │
│  - Resource resource                                        │
│  - ConfigProperties config                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ 由 Builder 创建
                           ▼
┌─────────────────────────────────────────────────────────────┐
│            AutoConfiguredOpenTelemetrySdkBuilder            │
│                     (主构建器)                               │
│  - buildImpl() 核心方法                                      │
│  - 14+ customizer 方法                                       │
└───┬─────┬─────┬─────┬─────┬─────┬─────────────────────────┘
    │     │     │     │     │     │
    │     │     │     │     │     └────────┐
    ▼     ▼     ▼     ▼     ▼     ▼        ▼
┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐┌────────┐
│Res││Trac││Mete││Logg││Prop││SpiH││Incubat│
│our││erPr││rPro││erPr││agat││elpe││ingUtil│
│ceC││ovid││vide││ovid││orCo││r   ││       │
│onf││erCo││rCon││erCo││nfig││    ││       │
│ig ││nfig││fig ││nfig││    ││    ││       │
└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└─┬──┘└───────┘
  │     │     │     │     │     │
  │     │     │     │     │     ├──────┐
  │     │     │     │     │     │      │
  │     ▼     ▼     ▼     ▼     ▼      ▼
  │  ┌────┐┌────┐┌────┐┌────┐┌────┐┌────┐
  │  │Span││Metr││LogR││Prop││Name││Comp│
  │  │Expo││icEx││ecor││agat││dSpi││onen│
  │  │rter││port││dExp││orCo││Mana││tLoa│
  │  │Conf││erCo││orte││nfig││ger ││der │
  │  │ig  ││nfig││rCon││    ││    ││    │
  │  └────┘└────┘└────┘└────┘└────┘└────┘
  │     │     │     │     │
  │     └─────┴─────┴─────┴──────────┐
  │                                   │
  └───────────────────────────────────┼──────┐
                                      │      │
                                      ▼      ▼
                                  ┌────┐  ┌────┐
                                  │SPI │  │环境│
                                  │实现│  │资源│
                                  │    │  │检测│
                                  └────┘  └────┘
```

#### 4.2 配置流程序列图

```
用户代码                Builder              各配置类              SPI实现
  │                      │                     │                     │
  │  initialize()        │                     │                     │
  ├─────────────────────>│                     │                     │
  │                      │  buildImpl()        │                     │
  │                      ├────────────────────>│                     │
  │                      │                     │                     │
  │                      │  1. 尝试 YAML 配置  │                     │
  │                      │  maybeConfigureFromFile()                 │
  │                      │─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│                     │
  │                      │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─│                     │
  │                      │                     │                     │
  │                      │  2. 加载 SPI 和定制器                     │
  │                      │  SpiHelper.create()│                     │
  │                      │────────────────────>│                     │
  │                      │  load SPI           │  ServiceLoader     │
  │                      │                     ├───────────────────>│
  │                      │                     │<───────────────────┤
  │                      │<────────────────────┤                     │
  │                      │                     │                     │
  │                      │  3. 加载配置属性    │                     │
  │                      │  getConfig()        │                     │
  │                      │────────────────────>│                     │
  │                      │<────────────────────┤                     │
  │                      │                     │                     │
  │                      │  4. 配置 Resource   │                     │
  │                      │  ResourceConfiguration.configureResource()│
  │                      │────────────────────>│                     │
  │                      │                     │  load ResourceProvider
  │                      │                     ├───────────────────>│
  │                      │                     │  createResource()  │
  │                      │                     │<───────────────────┤
  │                      │<────────────────────┤                     │
  │                      │                     │                     │
  │                      │  5. 配置 SDK 组件   │                     │
  │                      │  configureSdk()     │                     │
  │                      │────────────────────>│                     │
  │                      │                     │                     │
  │                      │  5.1 MeterProvider  │                     │
  │                      │  MeterProviderConfiguration               │
  │                      │                     ├────────────────────>│
  │                      │                     │  load MetricExporter
  │                      │                     │<────────────────────┤
  │                      │                     │                     │
  │                      │  5.2 TracerProvider │                     │
  │                      │  TracerProviderConfiguration              │
  │                      │                     ├────────────────────>│
  │                      │                     │  load SpanExporter │
  │                      │                     │<────────────────────┤
  │                      │                     │                     │
  │                      │  5.3 LoggerProvider │                     │
  │                      │  LoggerProviderConfiguration              │
  │                      │                     ├────────────────────>│
  │                      │                     │  load LogExporter  │
  │                      │                     │<────────────────────┤
  │                      │<────────────────────┤                     │
  │                      │                     │                     │
  │                      │  6. 构建 SDK        │                     │
  │                      │  build()            │                     │
  │                      │────────────────────>│                     │
  │                      │<────────────────────┤                     │
  │                      │                     │                     │
  │<─────────────────────┤                     │                     │
  │  AutoConfiguredOpenTelemetrySdk            │                     │
  │                      │                     │                     │
```

#### 4.3 SPI 加载时序图

```
NamedSpiManager        ServiceLoader        Provider1        Provider2
      │                     │                   │               │
      │  getByName("otlp")  │                   │               │
      ├────────────────────>│                   │               │
      │                     │  load()           │               │
      │                     ├──────────────────>│               │
      │                     │  new Provider1()  │               │
      │                     │<──────────────────┤               │
      │                     │                   │               │
      │                     │  load()           │               │
      │                     ├──────────────────────────────────>│
      │                     │  new Provider2()  │               │
      │                     │<──────────────────────────────────┤
      │<────────────────────┤                   │               │
      │  Iterable<Provider> │                   │               │
      │                     │                   │               │
      │  遍历 Provider      │                   │               │
      ├────────────────────────────────────────>│               │
      │  getName()          │                   │               │
      │<────────────────────────────────────────┤               │
      │  "jaeger"           │                   │               │
      │                     │                   │               │
      ├──────────────────────────────────────────────────────────>│
      │  getName()          │                   │               │
      │<────────────────────────────────────────────────────────┤
      │  "otlp" ✓ 匹配！     │                   │               │
      │                     │                   │               │
      ├──────────────────────────────────────────────────────────>│
      │  createExporter(config)                 │               │
      │<────────────────────────────────────────────────────────┤
      │  OtlpSpanExporter   │                   │               │
      │                     │                   │               │
      │  缓存 "otlp" → OtlpSpanExporter         │               │
      │                     │                   │               │
      │  返回 OtlpSpanExporter                  │               │
      │                     │                   │               │
```

### 5. 配置属性全景

#### 5.1 按组件分类的配置属性表

##### Resource 配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_SERVICE_NAME` | String | - | 服务名称 |
| `OTEL_RESOURCE_ATTRIBUTES` | Map | - | Resource 属性（key1=val1,key2=val2） |
| `OTEL_JAVA_ENABLED_RESOURCE_PROVIDERS` | List | - | 启用的 ResourceProvider 类名 |
| `OTEL_JAVA_DISABLED_RESOURCE_PROVIDERS` | List | - | 禁用的 ResourceProvider 类名 |
| `OTEL_RESOURCE_DISABLED_KEYS` | List | - | 要移除的 Resource 属性键 |

##### TracerProvider 配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_TRACES_EXPORTER` | String | `otlp` | 导出器名称（可多个：otlp,zipkin,logging） |
| `OTEL_TRACES_SAMPLER` | String | `parentbased_always_on` | 采样器名称 |
| `OTEL_TRACES_SAMPLER_ARG` | Double | - | 采样器参数（如采样率） |
| `OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT` | Integer | `128` | Span 最大属性数 |
| `OTEL_SPAN_EVENT_COUNT_LIMIT` | Integer | `128` | Span 最大事件数 |
| `OTEL_SPAN_LINK_COUNT_LIMIT` | Integer | `128` | Span 最大链接数 |
| `OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT` | Integer | - | 属性值长度限制 |
| `OTEL_BSP_SCHEDULE_DELAY` | Duration | `5000ms` | 批处理调度延迟 |
| `OTEL_BSP_MAX_QUEUE_SIZE` | Integer | `2048` | 最大队列大小 |
| `OTEL_BSP_MAX_EXPORT_BATCH_SIZE` | Integer | `512` | 每批最大 Span 数 |
| `OTEL_BSP_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |

##### MeterProvider 配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_METRICS_EXPORTER` | String | `otlp` | 导出器名称（可多个） |
| `OTEL_METRIC_EXPORT_INTERVAL` | Duration | `60000ms` | 导出间隔 |
| `OTEL_METRIC_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |
| `OTEL_METRICS_EXEMPLAR_FILTER` | String | `trace_based` | Exemplar 过滤器 |
| `OTEL_JAVA_METRICS_CARDINALITY_LIMIT` | Integer | `2000` | 基数限制 |

##### LoggerProvider 配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_LOGS_EXPORTER` | String | `otlp` | 日志导出器名称 |
| `OTEL_BLRP_SCHEDULE_DELAY` | Duration | `1000ms` | 批处理调度延迟 |
| `OTEL_BLRP_MAX_QUEUE_SIZE` | Integer | `2048` | 最大队列大小 |
| `OTEL_BLRP_MAX_EXPORT_BATCH_SIZE` | Integer | `512` | 每批最大日志数 |
| `OTEL_BLRP_EXPORT_TIMEOUT` | Duration | `30000ms` | 导出超时 |

##### Propagator 配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_PROPAGATORS` | List | `tracecontext,baggage` | 传播器列表 |

##### OTLP 导出器配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | String | `http://localhost:4317` | OTLP 端点 |
| `OTEL_EXPORTER_OTLP_HEADERS` | Map | - | HTTP 头 |
| `OTEL_EXPORTER_OTLP_TIMEOUT` | Duration | `10000ms` | 超时 |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | String | `grpc` | 协议（grpc, http/protobuf） |
| `OTEL_EXPORTER_OTLP_COMPRESSION` | String | - | 压缩方式（gzip, none） |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | String | - | Traces 特定端点 |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | String | - | Metrics 特定端点 |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | String | - | Logs 特定端点 |

##### Prometheus 导出器配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_EXPORTER_PROMETHEUS_PORT` | Integer | `9464` | HTTP 服务器端口 |
| `OTEL_EXPORTER_PROMETHEUS_HOST` | String | `localhost` | HTTP 服务器主机 |

##### Zipkin 导出器配置属性

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|-------|------|
| `OTEL_EXPORTER_ZIPKIN_ENDPOINT` | String | `http://localhost:9411/api/v2/spans` | Zipkin 端点 |
| `OTEL_EXPORTER_ZIPKIN_TIMEOUT` | Duration | `10000ms` | 超时 |

### 6. 最佳实践

#### 6.1 定制器使用指南

##### 何时使用 BiFunction 定制器

**BiFunction 定制器**适用于包装或修改已创建的组件：

```java
// 包装导出器添加日志
.addSpanExporterCustomizer((exporter, config) ->
    new LoggingSpanExporter(exporter))

// 包装处理器添加过滤
.addSpanProcessorCustomizer((processor, config) ->
    new FilteringSpanProcessor(processor))

// 修改 Resource 属性
.addResourceCustomizer((resource, config) ->
    resource.toBuilder()
        .put("additional.attribute", "value")
        .build())
```

##### 何时使用 Builder 定制器

**Builder 定制器**适用于直接配置 Provider Builder：

```java
// 直接配置 TracerProvider
.addTracerProviderCustomizer((builder, config) ->
    builder.setSampler(Sampler.alwaysOn())
           .setSpanLimits(SpanLimits.builder()
               .setMaxNumberOfAttributes(64)
               .build()))

// 配置 MeterProvider 的 View
.addMeterProviderCustomizer((builder, config) ->
    builder.registerView(
        InstrumentSelector.builder()
            .setType(InstrumentType.HISTOGRAM)
            .build(),
        View.builder()
            .setAggregation(Aggregation.explicitBucketHistogram(
                Arrays.asList(0.1, 0.5, 1.0, 5.0, 10.0)))
            .build()))
```

##### 定制器执行顺序

定制器按以下顺序执行：

1. **配置属性层**: `propertiesSupplier` → `propertiesCustomizer`
2. **Resource 层**: `resourceCustomizer`
3. **传播器层**: `propagatorCustomizer`
4. **Meter 层**: `metricExporterCustomizer` → `metricReaderCustomizer` → `meterProviderCustomizer`
5. **Tracer 层**: `samplerCustomizer` → `spanExporterCustomizer` → `spanProcessorCustomizer` → `tracerProviderCustomizer`
6. **Logger 层**: `logRecordExporterCustomizer` → `logRecordProcessorCustomizer` → `loggerProviderCustomizer`

##### 链式定制器模式

多个定制器会依次执行，形成链式调用：

```java
AutoConfiguredOpenTelemetrySdk.builder()
    // 定制器 1
    .addSpanExporterCustomizer((exporter, config) -> {
        System.out.println("Customizer 1: Adding logging");
        return new LoggingSpanExporter(exporter);
    })
    // 定制器 2（包装定制器 1 的结果）
    .addSpanExporterCustomizer((exporter, config) -> {
        System.out.println("Customizer 2: Adding retry logic");
        return new RetrySpanExporter(exporter);
    })
    .build();

// 执行结果:
// RetrySpanExporter → LoggingSpanExporter → OtlpSpanExporter
```

#### 6.2 性能优化

##### 批处理器调优

```bash
# 高吞吐量场景（牺牲实时性）
export OTEL_BSP_SCHEDULE_DELAY=10000           # 10秒导出一次
export OTEL_BSP_MAX_QUEUE_SIZE=8192            # 增大队列
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=2048     # 增大批大小

# 低延迟场景（更实时）
export OTEL_BSP_SCHEDULE_DELAY=1000            # 1秒导出一次
export OTEL_BSP_MAX_QUEUE_SIZE=1024            # 较小队列
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=256      # 较小批大小

# 内存受限场景
export OTEL_BSP_MAX_QUEUE_SIZE=512             # 减小队列
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=128      # 减小批大小
```

##### 采样率配置

```bash
# 生产环境（1% 采样）
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.01

# 预发布环境（10% 采样）
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1

# 开发环境（100% 采样）
export OTEL_TRACES_SAMPLER=always_on

# 智能采样（优先采样错误）
# 实现自定义 ConfigurableSamplerProvider
```

##### 资源限制配置

```bash
# 限制 Span 属性数量（防止高基数）
export OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT=32
export OTEL_SPAN_EVENT_COUNT_LIMIT=32
export OTEL_SPAN_LINK_COUNT_LIMIT=16

# 限制属性值长度（防止大对象）
export OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT=512

# 限制 Metric 基数
export OTEL_JAVA_METRICS_CARDINALITY_LIMIT=1000
```

##### 导出器性能调优

```bash
# OTLP gRPC 压缩
export OTEL_EXPORTER_OTLP_COMPRESSION=gzip

# 调整超时
export OTEL_EXPORTER_OTLP_TIMEOUT=15000        # 15秒

# Metric 导出间隔（降低频率）
export OTEL_METRIC_EXPORT_INTERVAL=120000      # 2分钟
```

#### 6.3 生产环境配置

##### 推荐的环境变量设置

```bash
# ============ 服务标识 ============
export OTEL_SERVICE_NAME=my-microservice
export OTEL_SERVICE_VERSION=1.2.3
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=production,service.namespace=my-company,cloud.provider=aws,cloud.region=us-west-2"

# ============ Traces 配置 ============
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.01              # 1% 采样

# Span 批处理器
export OTEL_BSP_SCHEDULE_DELAY=5000
export OTEL_BSP_MAX_QUEUE_SIZE=4096
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=1024
export OTEL_BSP_EXPORT_TIMEOUT=30000

# Span 限制
export OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT=64
export OTEL_SPAN_EVENT_COUNT_LIMIT=64
export OTEL_SPAN_LINK_COUNT_LIMIT=32
export OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT=1024

# ============ Metrics 配置 ============
export OTEL_METRICS_EXPORTER=otlp,prometheus
export OTEL_METRIC_EXPORT_INTERVAL=60000
export OTEL_METRICS_EXEMPLAR_FILTER=trace_based
export OTEL_JAVA_METRICS_CARDINALITY_LIMIT=2000

# Prometheus
export OTEL_EXPORTER_PROMETHEUS_PORT=9464
export OTEL_EXPORTER_PROMETHEUS_HOST=0.0.0.0

# ============ Logs 配置 ============
export OTEL_LOGS_EXPORTER=otlp

# 日志批处理器
export OTEL_BLRP_SCHEDULE_DELAY=1000
export OTEL_BLRP_MAX_QUEUE_SIZE=2048
export OTEL_BLRP_MAX_EXPORT_BATCH_SIZE=512

# ============ OTLP 导出器配置 ============
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_COMPRESSION=gzip
export OTEL_EXPORTER_OTLP_TIMEOUT=10000
export OTEL_EXPORTER_OTLP_HEADERS="api-key=your-api-key"

# ============ Propagators 配置 ============
export OTEL_PROPAGATORS=tracecontext,baggage
```

##### 高可用配置

**1. 使用多个导出器（容错）**:
```bash
# 主导出器 + 备份导出器
export OTEL_TRACES_EXPORTER=otlp,logging
export OTEL_EXPORTER_OTLP_ENDPOINT=http://primary-collector:4317
```

**2. 增大队列和批大小**:
```bash
export OTEL_BSP_MAX_QUEUE_SIZE=8192
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=2048
```

**3. 使用本地 Collector**:
```bash
# 导出到本地 Collector（降低网络延迟）
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

##### 监控和告警

**关键指标监控**:

```java
// 内置的 Metric 指标
// - otel.span_processor.queue.size
// - otel.span_processor.processed
// - otel.span_processor.dropped
// - otel.span_processor.export.duration

// 使用 JMX 或 Prometheus 监控这些指标
```

**告警规则示例**:
- `otel.span_processor.dropped > 100`: Span 丢失过多
- `otel.span_processor.queue.size > 1500`: 队列接近满
- `exporter_failure_rate > 0.05`: 导出失败率超过 5%

##### 故障恢复

**1. 优雅降级**:
```java
AutoConfiguredOpenTelemetrySdk.builder()
    .addSpanExporterCustomizer((exporter, config) ->
        new FallbackSpanExporter(exporter) {
            @Override
            public CompletableResultCode export(Collection<SpanData> spans) {
                try {
                    return super.export(spans);
                } catch (Exception e) {
                    logger.warn("Export failed, falling back to logging", e);
                    return CompletableResultCode.ofSuccess();
                }
            }
        })
    .build();
```

**2. 断路器模式**:
```java
// 实现带断路器的导出器
public class CircuitBreakerSpanExporter implements SpanExporter {
    private final CircuitBreaker circuitBreaker;
    private final SpanExporter delegate;

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        if (circuitBreaker.isOpen()) {
            logger.warn("Circuit breaker is open, skipping export");
            return CompletableResultCode.ofSuccess();
        }

        try {
            CompletableResultCode result = delegate.export(spans);
            circuitBreaker.recordSuccess();
            return result;
        } catch (Exception e) {
            circuitBreaker.recordFailure();
            throw e;
        }
    }
}
```

### 7. 扩展开发指南

#### 7.1 自定义导出器开发

##### 实现 ConfigurableSpanExporterProvider

```java
package com.example.exporter;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;

import java.util.Collection;
import java.util.concurrent.TimeUnit;

/**
 * 自定义导出器提供者
 */
public class CustomSpanExporterProvider implements ConfigurableSpanExporterProvider {

    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        // 读取配置
        String endpoint = config.getString("otel.exporter.custom.endpoint");
        if (endpoint == null) {
            throw new IllegalArgumentException("otel.exporter.custom.endpoint is required");
        }

        int timeout = config.getInt("otel.exporter.custom.timeout", 10000);
        boolean compression = config.getBoolean("otel.exporter.custom.compression", false);

        // 创建并返回导出器
        return new CustomSpanExporter(endpoint, timeout, compression);
    }

    @Override
    public String getName() {
        // 返回导出器名称（用于 OTEL_TRACES_EXPORTER）
        return "custom";
    }
}

/**
 * 自定义导出器实现
 */
class CustomSpanExporter implements SpanExporter {

    private final String endpoint;
    private final int timeoutMs;
    private final boolean compression;

    public CustomSpanExporter(String endpoint, int timeoutMs, boolean compression) {
        this.endpoint = endpoint;
        this.timeoutMs = timeoutMs;
        this.compression = compression;
    }

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        CompletableResultCode result = new CompletableResultCode();

        // 异步导出
        CompletableFuture.runAsync(() -> {
            try {
                // 1. 序列化 Span
                byte[] data = serializeSpans(spans);

                // 2. 压缩（如果启用）
                if (compression) {
                    data = compress(data);
                }

                // 3. 发送到后端
                boolean success = sendToBackend(endpoint, data, timeoutMs);

                if (success) {
                    result.succeed();
                } else {
                    result.fail();
                }
            } catch (Exception e) {
                logger.error("Export failed", e);
                result.fail();
            }
        });

        return result;
    }

    @Override
    public CompletableResultCode flush() {
        // 刷新缓冲区（如果有）
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        // 关闭连接、释放资源
        logger.info("Shutting down CustomSpanExporter");
        return CompletableResultCode.ofSuccess();
    }

    private byte[] serializeSpans(Collection<SpanData> spans) {
        // 序列化实现
        return ...;
    }

    private byte[] compress(byte[] data) {
        // 压缩实现
        return ...;
    }

    private boolean sendToBackend(String endpoint, byte[] data, int timeout) {
        // HTTP/gRPC 发送实现
        return ...;
    }
}
```

##### 注册 SPI

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider`:

```
com.example.exporter.CustomSpanExporterProvider
```

##### 配置属性命名约定

遵循以下命名规范：

- **导出器特定配置**: `otel.exporter.<exporter-name>.*`
  - 示例: `otel.exporter.custom.endpoint`
  - 示例: `otel.exporter.custom.timeout`

- **使用小写和点分隔**: `otel.exporter.mycustom.property.name`

- **布尔值**: 使用 `true`/`false`
- **时间间隔**: 支持毫秒数字或带单位字符串（`5000` 或 `5s`）

##### 测试指南

```java
@Test
public void testCustomSpanExporter() {
    // 1. 准备配置
    Map<String, String> config = new HashMap<>();
    config.put("otel.service.name", "test-service");
    config.put("otel.traces.exporter", "custom");
    config.put("otel.exporter.custom.endpoint", "http://localhost:8080");

    // 2. 构建 SDK
    AutoConfiguredOpenTelemetrySdk sdk =
        AutoConfiguredOpenTelemetrySdk.builder()
            .addPropertiesSupplier(() -> config)
            .build();

    // 3. 创建 Span
    Tracer tracer = sdk.getOpenTelemetrySdk().getTracer("test");
    Span span = tracer.spanBuilder("test-span").startSpan();
    span.end();

    // 4. 等待导出
    Thread.sleep(6000);

    // 5. 验证（检查后端是否收到 Span）
    // ...

    // 6. 清理
    sdk.getOpenTelemetrySdk().close();
}
```

#### 7.2 自定义采样器开发

##### 实现 ConfigurableSamplerProvider

```java
package com.example.sampler;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.sdk.trace.samplers.SamplingResult;
import io.opentelemetry.sdk.trace.samplers.SamplingDecision;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.context.Context;
import io.opentelemetry.sdk.trace.data.LinkData;

import java.util.List;

/**
 * 自定义采样器提供者
 */
public class CustomSamplerProvider implements ConfigurableSamplerProvider {

    @Override
    public Sampler createSampler(ConfigProperties config) {
        // 读取配置
        double fallbackRate = config.getDouble("otel.traces.sampler.arg", 0.1);
        boolean priorityEnabled = config.getBoolean("otel.traces.sampler.priority", true);

        return new PrioritySampler(fallbackRate, priorityEnabled);
    }

    @Override
    public String getName() {
        return "priority";  // 使用: OTEL_TRACES_SAMPLER=priority
    }
}

/**
 * 优先级采样器：优先采样重要请求
 */
class PrioritySampler implements Sampler {

    private final double fallbackRate;
    private final boolean priorityEnabled;
    private final Sampler fallbackSampler;

    public PrioritySampler(double fallbackRate, boolean priorityEnabled) {
        this.fallbackRate = fallbackRate;
        this.priorityEnabled = priorityEnabled;
        this.fallbackSampler = Sampler.traceIdRatioBased(fallbackRate);
    }

    @Override
    public SamplingResult shouldSample(
        Context parentContext,
        String traceId,
        String name,
        SpanKind spanKind,
        Attributes attributes,
        List<LinkData> parentLinks) {

        if (priorityEnabled) {
            // 规则 1: 带有 priority 属性 → 100% 采样
            Boolean priority = attributes.get(AttributeKey.booleanKey("priority"));
            if (Boolean.TRUE.equals(priority)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 2: 错误请求 → 100% 采样
            Boolean error = attributes.get(AttributeKey.booleanKey("error"));
            if (Boolean.TRUE.equals(error)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 3: 特定端点 → 100% 采样
            String endpoint = attributes.get(AttributeKey.stringKey("http.target"));
            if ("/api/critical".equals(endpoint)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }
        }

        // 规则 4: 默认采样逻辑
        return fallbackSampler.shouldSample(
            parentContext, traceId, name, spanKind, attributes, parentLinks);
    }

    @Override
    public String getDescription() {
        return String.format("PrioritySampler{fallbackRate=%f, priorityEnabled=%b}",
            fallbackRate, priorityEnabled);
    }
}
```

##### 采样决策逻辑

采样器可以返回三种决策：

1. **`RECORD_AND_SAMPLE`**: 记录并导出 Span
2. **`RECORD_ONLY`**: 仅记录 Span，不导出
3. **`DROP`**: 不记录也不导出

##### 性能考虑

- ✅ **快速决策**: `shouldSample()` 会被频繁调用，必须快速返回
- ✅ **避免阻塞**: 不要执行 I/O 操作
- ✅ **线程安全**: 采样器是线程共享的，必须线程安全
- ✅ **无状态**: 避免维护状态，否则需要考虑并发问题

##### 测试指南

```java
@Test
public void testPrioritySampler() {
    ConfigProperties config = DefaultConfigProperties.create(
        Map.of("otel.traces.sampler.arg", "0.1"));

    Sampler sampler = new PrioritySamplerProvider().createSampler(config);

    // 测试优先级请求
    Attributes priorityAttrs = Attributes.of(
        AttributeKey.booleanKey("priority"), true);
    SamplingResult result = sampler.shouldSample(
        Context.root(), "trace-id", "test-span",
        SpanKind.SERVER, priorityAttrs, Collections.emptyList());

    assertEquals(SamplingDecision.RECORD_AND_SAMPLE, result.getDecision());

    // 测试普通请求
    Attributes normalAttrs = Attributes.empty();
    // 由于使用 TraceId 比例采样，需要多次测试
    int sampledCount = 0;
    for (int i = 0; i < 1000; i++) {
        result = sampler.shouldSample(
            Context.root(), "trace-id-" + i, "test-span",
            SpanKind.SERVER, normalAttrs, Collections.emptyList());
        if (result.getDecision() == SamplingDecision.RECORD_AND_SAMPLE) {
            sampledCount++;
        }
    }
    // 应该接近 10% 采样率
    assertTrue(sampledCount >= 50 && sampledCount <= 150);
}
```

#### 7.3 自定义 Resource 提供者

##### 实现 ResourceProvider

```java
package com.example.resource;

import io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;

import java.io.BufferedReader;
import java.io.InputStreamReader;

/**
 * Kubernetes 资源提供者
 */
public class KubernetesResourceProvider implements ResourceProvider {

    @Override
    public Resource createResource(ConfigProperties config) {
        Attributes.Builder builder = Attributes.builder();

        // 从环境变量读取 K8s 信息
        String podName = System.getenv("POD_NAME");
        if (podName != null) {
            builder.put(AttributeKey.stringKey("k8s.pod.name"), podName);
        }

        String namespace = System.getenv("NAMESPACE");
        if (namespace != null) {
            builder.put(AttributeKey.stringKey("k8s.namespace.name"), namespace);
        }

        String deploymentName = System.getenv("DEPLOYMENT_NAME");
        if (deploymentName != null) {
            builder.put(AttributeKey.stringKey("k8s.deployment.name"), deploymentName);
        }

        // 从 Downward API 读取节点名称
        String nodeName = readFromFile("/etc/podinfo/node-name");
        if (nodeName != null) {
            builder.put(AttributeKey.stringKey("k8s.node.name"), nodeName);
        }

        // 从配置读取集群名称
        String clusterName = config.getString("otel.k8s.cluster.name");
        if (clusterName != null) {
            builder.put(AttributeKey.stringKey("k8s.cluster.name"), clusterName);
        }

        return Resource.create(builder.build());
    }

    @Override
    public int order() {
        // 在默认 ResourceProvider 之后执行
        return 100;
    }

    private String readFromFile(String path) {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(new FileInputStream(path)))) {
            return reader.readLine();
        } catch (IOException e) {
            return null;
        }
    }
}
```

##### 环境检测最佳实践

**1. 使用条件检测**:
```java
@Override
public Resource createResource(ConfigProperties config) {
    // 只在 Kubernetes 环境中返回 Resource
    if (!isRunningInKubernetes()) {
        return Resource.empty();
    }

    // ... 创建 K8s Resource
}

private boolean isRunningInKubernetes() {
    return System.getenv("KUBERNETES_SERVICE_HOST") != null;
}
```

**2. 失败容错**:
```java
private String getMetadata(String endpoint) {
    try {
        // 尝试从 metadata 服务读取
        return httpGet(endpoint);
    } catch (Exception e) {
        logger.debug("Failed to read metadata from " + endpoint, e);
        return null;  // 失败时返回 null
    }
}
```

**3. 超时控制**:
```java
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
conn.setConnectTimeout(100);  // 100ms 超时
conn.setReadTimeout(100);
```

##### Ordered 接口使用

`order()` 值决定执行顺序和属性覆盖优先级：

```java
// 执行顺序示例
DefaultResourceProvider: order() = 0      // 最早执行
EnvironmentResourceProvider: order() = 50  // 中间执行
CustomResourceProvider: order() = 100      // 最晚执行（优先级最高）

// 属性覆盖
// 如果三个 Provider 都提供 "environment" 属性：
// CustomResourceProvider 的值会覆盖其他两个
```

##### 测试指南

```java
@Test
public void testKubernetesResourceProvider() {
    // 模拟环境变量
    Map<String, String> env = new HashMap<>();
    env.put("POD_NAME", "my-pod-12345");
    env.put("NAMESPACE", "production");

    // 使用 SystemStub 或类似库设置环境变量
    withEnvironmentVariables(env).execute(() -> {
        ConfigProperties config = DefaultConfigProperties.createForTest(
            Map.of("otel.k8s.cluster.name", "my-cluster"));

        Resource resource = new KubernetesResourceProvider().createResource(config);

        assertEquals("my-pod-12345",
            resource.getAttribute(AttributeKey.stringKey("k8s.pod.name")));
        assertEquals("production",
            resource.getAttribute(AttributeKey.stringKey("k8s.namespace.name")));
        assertEquals("my-cluster",
            resource.getAttribute(AttributeKey.stringKey("k8s.cluster.name")));
    });
}
```

---

## 常见问题

### Q1: 环境变量未生效？

**检查清单**:
1. 确认环境变量名称正确（注意大小写）
2. 确认环境变量在应用启动前已设置
3. 检查是否有系统属性覆盖了环境变量
4. 启用 debug 日志查看配置加载：
```bash
export OTEL_JAVAAGENT_DEBUG=true
```

### Q2: 如何查看当前的配置？

```java
AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.initialize();

// 获取配置属性（仅用于调试）
ConfigProperties config = autoConfigured.getConfig();
System.out.println("Service name: " + config.getString("otel.service.name"));
```

### Q3: 如何同时使用多个导出器？

```bash
# 用逗号分隔多个导出器
export OTEL_TRACES_EXPORTER=otlp,zipkin,logging
export OTEL_METRICS_EXPORTER=otlp,prometheus
```

### Q4: 采样率如何配置？

```bash
# 10% 采样
export OTEL_TRACES_SAMPLER=traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1

# 继承父 Span 决策，根 Span 10% 采样
export OTEL_TRACES_SAMPLER=parentbased_traceidratio
export OTEL_TRACES_SAMPLER_ARG=0.1
```

### Q5: 如何禁用自动配置？

```bash
# 完全禁用 SDK
export OTEL_SDK_DISABLED=true
```

### Q6: 如何配置 OTLP 认证？

```bash
# 通过 Headers
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer your-token"

# 或通过自定义导出器
```

### Q7: 批处理器性能调优？

```bash
# 增加队列大小
export OTEL_BSP_MAX_QUEUE_SIZE=4096

# 增加批大小
export OTEL_BSP_MAX_EXPORT_BATCH_SIZE=1024

# 减少延迟（更频繁导出）
export OTEL_BSP_SCHEDULE_DELAY=1000
```

### Q8: 如何处理高基数属性？

```bash
# 限制属性数量
export OTEL_SPAN_ATTRIBUTE_COUNT_LIMIT=64

# 限制属性值长度
export OTEL_SPAN_ATTRIBUTE_VALUE_LENGTH_LIMIT=512
```

### Q9: 声明式配置文件不生效？

确保添加了 incubator 依赖：
```kotlin
implementation("io.opentelemetry:opentelemetry-sdk-extension-incubator")
```

### Q10: 如何在测试中使用？

```java
// 使用 No-op 实现
Map<String, String> testConfig = new HashMap<>();
testConfig.put("otel.traces.exporter", "none");
testConfig.put("otel.metrics.exporter", "none");

AutoConfiguredOpenTelemetrySdk autoConfigured =
    AutoConfiguredOpenTelemetrySdk.builder()
        .addPropertiesSupplier(() -> testConfig)
        .build();
```

---

## 相关资源

- **官方文档**: [OpenTelemetry Java Documentation](https://opentelemetry.io/docs/instrumentation/java/)
- **环境变量规范**: [OpenTelemetry Environment Variable Specification](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)
- **API 文档**: [JavaDoc](https://javadoc.io/doc/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure)
- **示例代码**: [examples/](../../examples/)
- **问题反馈**: [GitHub Issues](https://github.com/open-telemetry/opentelemetry-java/issues)

---

## 许可证

本项目采用 Apache License 2.0 许可证。详见 [LICENSE](../../LICENSE) 文件。

---

**维护者**: OpenTelemetry Java SIG
**最后更新**: 2026-01-12
**文档版本**: 1.0.0
