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
