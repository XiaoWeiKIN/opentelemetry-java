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
