# OpenTelemetry SDK 自动配置 SPI

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.opentelemetry/opentelemetry-sdk-extension-autoconfigure-spi)

---

## 📑 目录

- [概述](#概述)
- [快速开始](#快速开始)
- [核心概念](#核心概念)
- [SPI 接口参考](#spi-接口参考)
  - [通用配置 SPI](#通用配置-spi)
  - [Traces 信号 SPI](#traces-信号-spi)
  - [Metrics 信号 SPI](#metrics-信号-spi)
  - [Logs 信号 SPI](#logs-信号-spi)
- [高级主题](#高级主题)
- [内部 API](#内部-api)
- [完整示例](#完整示例)
- [最佳实践](#最佳实践)
- [故障排查](#故障排查)
- [迁移指南](#迁移指南)
- [API 稳定性](#api-稳定性)
- [相关文档](#相关文档)

---

## 概述

**OpenTelemetry SDK Autoconfigure SPI** 是 OpenTelemetry Java SDK 自动配置的 **Service Provider Interface (SPI)** 模块，定义了扩展自动配置功能的所有公共接口。

### 模块定位

```
用户应用程序
    ↓ 使用
OpenTelemetry API
    ↓ 实现
OpenTelemetry SDK
    ↓ 自动配置
autoconfigure 模块 ← 使用 SPI 接口
    ↓ 定义扩展点
autoconfigure-spi 模块 ← 本模块
    ↓ 实现
用户自定义扩展（SpanExporter、Sampler、ResourceProvider 等）
```

### 核心职责

- ✅ **定义 SPI 扩展点**：允许用户自定义 SDK 组件（导出器、采样器、资源检测器等）
- ✅ **提供配置属性访问接口**：类型安全地读取环境变量和系统属性
- ✅ **定义信号专用的提供者接口**：Traces、Metrics、Logs 各有专用 SPI
- ✅ **支持声明式配置和编程式定制**：通过环境变量或代码定制 SDK
- ✅ **控制加载顺序**：通过 `Ordered` 接口控制多个扩展的执行顺序

### 为什么需要 SPI？

传统的 SDK 配置需要编写大量代码来注册自定义组件。使用 SPI，您可以：

```java
// ❌ 传统方式：手动注册自定义导出器
SpanExporter customExporter = new MyCustomSpanExporter();
SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
    .addSpanProcessor(BatchSpanProcessor.builder(customExporter).build())
    .build();

// ✅ SPI 方式：自动发现和注册
// 1. 实现 ConfigurableSpanExporterProvider 接口
// 2. 注册到 META-INF/services/
// 3. 设置环境变量 OTEL_TRACES_EXPORTER=my-custom
// SDK 自动加载并使用您的导出器
```

### 模块结构

```
sdk-extensions/autoconfigure-spi/
├── src/main/java/io/opentelemetry/sdk/autoconfigure/spi/
│   ├── (root) - 核心 SPI 接口 (7 个)
│   │   ├── AutoConfigurationCustomizer.java           # 定制器构建接口
│   │   ├── AutoConfigurationCustomizerProvider.java   # 定制器提供者 SPI
│   │   ├── ConfigProperties.java                      # 配置属性访问接口
│   │   ├── ConfigurablePropagatorProvider.java        # 自定义传播器
│   │   ├── ResourceProvider.java                      # 资源属性提供者
│   │   ├── Ordered.java                               # 加载顺序控制
│   │   └── ConfigurationException.java                # 配置异常
│   ├── traces/ - Trace 专用 SPI (3 个)
│   │   ├── ConfigurableSpanExporterProvider.java      # 自定义 Span 导出器
│   │   ├── ConfigurableSamplerProvider.java           # 自定义采样器
│   │   └── SdkTracerProviderConfigurer.java           # TracerProvider 定制器 (已废弃)
│   ├── metrics/ - Metrics 专用 SPI (1 个)
│   │   └── ConfigurableMetricExporterProvider.java    # 自定义 Metric 导出器
│   ├── logs/ - Logs 专用 SPI (1 个)
│   │   └── ConfigurableLogRecordExporterProvider.java # 自定义日志导出器
│   └── internal/ - 内部 API (5 个, 标记 @Internal)
│       ├── ComponentProvider.java                     # 声明式配置组件提供者
│       ├── ConfigurableMetricReaderProvider.java      # 自定义 Metric 读取器
│       ├── ConditionalResourceProvider.java           # 条件 Resource 提供者
│       ├── AutoConfigureListener.java                 # 配置完成监听器
│       └── DefaultConfigProperties.java               # ConfigProperties 实现
└── build.gradle.kts
```

**17 个接口/类分为 6 大类**：
1. **通用定制接口** (2 个)：`AutoConfigurationCustomizer`、`AutoConfigurationCustomizerProvider`
2. **配置基础设施** (3 个)：`ConfigProperties`、`Ordered`、`ConfigurationException`
3. **资源和传播器** (2 个)：`ResourceProvider`、`ConfigurablePropagatorProvider`
4. **Traces 信号 SPI** (3 个)：`ConfigurableSpanExporterProvider`、`ConfigurableSamplerProvider`、`SdkTracerProviderConfigurer`
5. **Metrics 信号 SPI** (1 个)：`ConfigurableMetricExporterProvider`
6. **Logs 信号 SPI** (1 个)：`ConfigurableLogRecordExporterProvider`
7. **内部 API** (5 个)：标记为 `@Internal`，不保证稳定性

### 与 autoconfigure 模块的关系

```
autoconfigure 模块                    autoconfigure-spi 模块
┌─────────────────────────┐          ┌─────────────────────────┐
│ AutoConfiguredOpenTele- │          │ SPI 接口定义             │
│ metrySdkBuilder         │   依赖   │ - SpanExporterProvider  │
│                         │ ───────> │ - SamplerProvider       │
│ - 加载 SPI 实现         │          │ - ResourceProvider      │
│ - 构建 SDK 组件         │          │ - ConfigProperties      │
│ - 应用定制器            │          │ - AutoConfiguration-    │
│ - 环境变量配置          │          │   Customizer            │
└─────────────────────────┘          └─────────────────────────┘
         │                                      ▲
         │ 使用                                 │ 实现
         ▼                                      │
┌─────────────────────────┐          ┌─────────────────────────┐
│ 构建的 OpenTelemetry    │          │ 用户自定义实现           │
│ SDK 实例                 │          │ - MyCustomExporter      │
│ - TracerProvider        │          │ - MyCustomSampler       │
│ - MeterProvider         │          │ - MyResourceProvider    │
│ - LoggerProvider        │          │                         │
└─────────────────────────┘          └─────────────────────────┘
```

**简单来说**：
- **autoconfigure-spi**：定义"可以扩展什么"（接口定义）
- **autoconfigure**：实现"如何扩展"（SPI 加载和配置逻辑）
- **用户扩展**：实现"扩展内容"（自定义组件）

---

## 快速开始

### 1. 添加依赖

**Gradle**:
```kotlin
dependencies {
    // 如果只需要定义 SPI 扩展
    compileOnly("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi")

    // 通常还需要 SDK API
    implementation("io.opentelemetry:opentelemetry-sdk")
}
```

**Maven**:
```xml
<dependencies>
    <!-- SPI 接口 -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk-extension-autoconfigure-spi</artifactId>
        <scope>provided</scope>
    </dependency>

    <!-- SDK API -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
    </dependency>
</dependencies>
```

### 2. 实现您的第一个 SPI

**示例：自定义 Span 导出器**

```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;

import java.util.Collection;

/**
 * 自定义 Span 导出器提供者
 */
public class MyCustomSpanExporterProvider implements ConfigurableSpanExporterProvider {

    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        // 从配置读取自定义属性
        String endpoint = config.getString("otel.exporter.mycustom.endpoint", "http://localhost:8080");
        int timeout = config.getInt("otel.exporter.mycustom.timeout", 5000);

        // 创建并返回自定义导出器
        return new MyCustomSpanExporter(endpoint, timeout);
    }

    @Override
    public String getName() {
        // 返回导出器名称，用于环境变量 OTEL_TRACES_EXPORTER=mycustom
        return "mycustom";
    }
}

/**
 * 自定义 Span 导出器实现
 */
class MyCustomSpanExporter implements SpanExporter {

    private final String endpoint;
    private final int timeout;

    public MyCustomSpanExporter(String endpoint, int timeout) {
        this.endpoint = endpoint;
        this.timeout = timeout;
    }

    @Override
    public CompletableResultCode export(Collection<SpanData> spans) {
        // 导出逻辑：发送 Span 到自定义后端
        System.out.println("Exporting " + spans.size() + " spans to " + endpoint);

        for (SpanData span : spans) {
            // 转换为自定义格式并发送
            String json = convertToJson(span);
            sendToBackend(json);
        }

        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode flush() {
        // 刷新缓冲区
        return CompletableResultCode.ofSuccess();
    }

    @Override
    public CompletableResultCode shutdown() {
        // 关闭连接、释放资源
        System.out.println("Shutting down MyCustomSpanExporter");
        return CompletableResultCode.ofSuccess();
    }

    private String convertToJson(SpanData span) {
        // 简化的 JSON 转换
        return String.format(
            "{\"traceId\":\"%s\",\"spanId\":\"%s\",\"name\":\"%s\"}",
            span.getTraceId(),
            span.getSpanId(),
            span.getName()
        );
    }

    private void sendToBackend(String json) {
        // 发送到自定义后端（HTTP、Kafka、数据库等）
        // 实际实现省略
    }
}
```

### 3. 注册 SPI

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider`

**文件内容**:
```
com.example.MyCustomSpanExporterProvider
```

**注意**：
- 文件名必须是接口的**完全限定名**
- 文件内容是实现类的**完全限定名**（每行一个）
- 文件必须使用 **UTF-8** 编码

### 4. 配置并使用

**设置环境变量**:
```bash
# 启用自定义导出器
export OTEL_TRACES_EXPORTER=mycustom

# 自定义配置（可选）
export OTEL_EXPORTER_MYCUSTOM_ENDPOINT=http://my-backend:9090
export OTEL_EXPORTER_MYCUSTOM_TIMEOUT=10000

# 服务名称
export OTEL_SERVICE_NAME=my-service
```

**Java 代码**:
```java
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;

public class Application {
    public static void main(String[] args) {
        // 自动配置会发现并使用您的自定义导出器
        OpenTelemetry openTelemetry = AutoConfiguredOpenTelemetrySdk.initialize()
            .getOpenTelemetrySdk();

        // 使用 OpenTelemetry
        Tracer tracer = openTelemetry.getTracer("my-app");
        Span span = tracer.spanBuilder("myOperation").startSpan();
        try {
            // 业务逻辑
            System.out.println("Doing work...");
        } finally {
            span.end();  // Span 会被导出到自定义后端
        }
    }
}
```

**运行结果**:
```
Exporting 1 spans to http://my-backend:9090
{"traceId":"0af7651916cd43dd8448eb211c80319c","spanId":"b7ad6b7169203331","name":"myOperation"}
Shutting down MyCustomSpanExporter
```

---

## 核心概念

### Service Provider Interface (SPI)

#### 什么是 SPI？

**SPI** (Service Provider Interface) 是 Java 的一种服务发现机制，允许在运行时动态加载接口的实现类。

**核心思想**：
- **接口定义者**（OpenTelemetry）定义扩展点接口
- **实现者**（用户）提供具体实现
- **加载器**（ServiceLoader）在运行时自动发现实现

**Java SPI 工作原理**:
```
1. 定义接口
   io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider

2. 实现接口
   com.example.MyCustomSpanExporterProvider implements ConfigurableSpanExporterProvider

3. 注册实现
   META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider
   内容: com.example.MyCustomSpanExporterProvider

4. 加载实现
   ServiceLoader<ConfigurableSpanExporterProvider> loader =
       ServiceLoader.load(ConfigurableSpanExporterProvider.class);
   for (ConfigurableSpanExporterProvider provider : loader) {
       // 使用 provider
   }
```

#### OpenTelemetry 如何使用 SPI

OpenTelemetry 自动配置模块使用 SPI 加载用户提供的扩展：

```
应用启动
    ↓
AutoConfiguredOpenTelemetrySdk.initialize()
    ↓
┌─────────────────────────────────────────┐
│ 1. 读取环境变量                          │
│    OTEL_TRACES_EXPORTER=mycustom        │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 2. 加载所有 SpanExporterProvider        │
│    ServiceLoader.load(                  │
│      ConfigurableSpanExporterProvider)  │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 3. 查找名称匹配的 Provider              │
│    provider.getName() == "mycustom"     │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 4. 创建导出器                            │
│    SpanExporter exporter =              │
│      provider.createExporter(config)    │
└──────────────┬──────────────────────────┘
               ↓
┌─────────────────────────────────────────┐
│ 5. 注册到 TracerProvider                │
│    tracerProvider.addSpanProcessor(     │
│      BatchSpanProcessor.builder(        │
│        exporter).build())               │
└─────────────────────────────────────────┘
```

#### SPI 加载机制

**加载顺序**：
1. **发现**：扫描 classpath 中所有的 `META-INF/services/` 文件
2. **实例化**：反射创建实现类的实例（要求有无参构造函数）
3. **排序**：如果实现了 `Ordered` 接口，按 `order()` 值排序
4. **调用**：按顺序调用实现类的方法

**示例**：
```java
// 实现 Ordered 接口控制加载顺序
public class MyResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(
            Attributes.of(AttributeKey.stringKey("my.key"), "my.value")
        );
    }

    @Override
    public int order() {
        return 100;  // 数值越大，越晚执行（优先级越高）
    }
}
```

**顺序规则**：
- 数值越小，越早执行
- 数值越大，越晚执行（优先级越高）
- 后执行的 ResourceProvider 可以覆盖前面的属性
- 默认 `order()` 返回 0

### 配置优先级

OpenTelemetry 自动配置使用以下优先级读取配置属性：

```
┌─────────────────────────────────────────┐
│ 1. 系统属性 (最高优先级)                │
│    -Dotel.service.name=my-service       │
└──────────────┬──────────────────────────┘
               ↓ 覆盖
┌─────────────────────────────────────────┐
│ 2. 环境变量                              │
│    OTEL_SERVICE_NAME=my-service         │
└──────────────┬──────────────────────────┘
               ↓ 覆盖
┌─────────────────────────────────────────┐
│ 3. 自定义属性提供者                      │
│    addPropertiesSupplier(() -> map)     │
└──────────────┬──────────────────────────┘
               ↓ 覆盖
┌─────────────────────────────────────────┐
│ 4. 默认值 (最低优先级)                  │
│    config.getString("key", "default")   │
└─────────────────────────────────────────┘
```

**示例**：
```bash
# 1. 环境变量
export OTEL_SERVICE_NAME=my-service

# 2. 系统属性覆盖环境变量
java -Dotel.service.name=overridden-service -jar myapp.jar

# 结果：使用 "overridden-service"
```

### ConfigProperties 接口

`ConfigProperties` 接口提供类型安全的配置属性访问：

```java
public interface ConfigProperties {
    // 字符串
    String getString(String name);
    String getString(String name, String defaultValue);

    // 布尔值
    Boolean getBoolean(String name);
    boolean getBoolean(String name, boolean defaultValue);

    // 整数
    Integer getInt(String name);
    int getInt(String name, int defaultValue);

    // 长整数
    Long getLong(String name);
    long getLong(String name, long defaultValue);

    // 双精度浮点数
    Double getDouble(String name);
    double getDouble(String name, double defaultValue);

    // 时间间隔
    Duration getDuration(String name);
    Duration getDuration(String name, Duration defaultValue);

    // 字符串列表（逗号分隔）
    List<String> getList(String name);
    List<String> getList(String name, List<String> defaultValue);

    // Map（key1=val1,key2=val2 格式）
    Map<String, String> getMap(String name);
    Map<String, String> getMap(String name, Map<String, String> defaultValue);
}
```

**使用示例**：
```java
public class MyProvider implements ConfigurableSpanExporterProvider {
    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        // 读取字符串（带默认值）
        String endpoint = config.getString(
            "otel.exporter.mycustom.endpoint",
            "http://localhost:8080"
        );

        // 读取整数
        int timeout = config.getInt("otel.exporter.mycustom.timeout", 5000);

        // 读取布尔值
        boolean enableCompression = config.getBoolean(
            "otel.exporter.mycustom.compression",
            false
        );

        // 读取时间间隔
        Duration batchDelay = config.getDuration(
            "otel.exporter.mycustom.batch.delay",
            Duration.ofSeconds(5)
        );

        // 读取列表（逗号分隔）
        List<String> headers = config.getList("otel.exporter.mycustom.headers");

        // 读取 Map（key1=val1,key2=val2）
        Map<String, String> metadata = config.getMap("otel.exporter.mycustom.metadata");

        return new MyCustomSpanExporter(endpoint, timeout, enableCompression);
    }

    @Override
    public String getName() {
        return "mycustom";
    }
}
```

**环境变量映射规则**：
```
Java 属性名                      环境变量名
─────────────────────────────────────────────────
otel.service.name               OTEL_SERVICE_NAME
otel.traces.exporter            OTEL_TRACES_EXPORTER
otel.exporter.otlp.endpoint     OTEL_EXPORTER_OTLP_ENDPOINT
```

**规则**：
- 小写字母 → 大写字母
- `.` → `_`
- `-` → `_`

---

## SPI 接口参考

本节详细介绍 autoconfigure-spi 模块中的所有 17 个接口和类。

### 通用配置 SPI

#### AutoConfigurationCustomizerProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供全局定制自动配置的入口点，允许用户在 SDK 构建过程中注入自定义逻辑。

**使用场景**:
- 需要定制多个 SDK 组件（而不仅仅是单个导出器或采样器）
- 需要在自动配置流程中添加全局逻辑
- 需要修改已配置的组件（例如包装导出器添加日志）

**接口定义**:
```java
public interface AutoConfigurationCustomizerProvider extends Ordered {
    /**
     * 定制自动配置
     * @param customizer 定制器构建接口
     */
    void customize(AutoConfigurationCustomizer customizer);

    /**
     * 执行顺序（可选）
     * @return 顺序值，数值越大越晚执行
     */
    default int order() {
        return 0;
    }
}
```

**完整实现示例**:
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizer;
import io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.trace.SpanProcessor;
import io.opentelemetry.sdk.trace.data.SpanData;
import io.opentelemetry.context.Context;

/**
 * 生产环境的全局定制器
 */
public class ProductionCustomizerProvider implements AutoConfigurationCustomizerProvider {

    @Override
    public void customize(AutoConfigurationCustomizer customizer) {
        // 1. 添加全局 Resource 属性
        customizer.addResourceCustomizer((resource, config) ->
            resource.toBuilder()
                .put(AttributeKey.stringKey("deployment.environment"), "production")
                .put(AttributeKey.stringKey("service.namespace"), "my-company")
                .put(AttributeKey.stringKey("deployment.version"), getDeploymentVersion())
                .build()
        );

        // 2. 过滤内部 Span（性能优化）
        customizer.addSpanProcessorCustomizer((processor, config) ->
            new FilteringSpanProcessor(processor) {
                @Override
                public void onEnd(ReadableSpan span) {
                    // 只导出非内部 Span
                    if (!span.getName().startsWith("internal.")) {
                        super.onEnd(span);
                    }
                }
            }
        );

        // 3. 添加日志记录导出器（审计）
        customizer.addSpanExporterCustomizer((exporter, config) ->
            new LoggingSpanExporter(exporter) {
                @Override
                public CompletableResultCode export(Collection<SpanData> spans) {
                    System.out.println("Exporting " + spans.size() + " spans");
                    return super.export(spans);
                }
            }
        );

        // 4. 添加默认配置属性
        customizer.addPropertiesSupplier(() -> {
            Map<String, String> defaults = new HashMap<>();
            defaults.put("otel.traces.sampler", "parentbased_traceidratio");
            defaults.put("otel.traces.sampler.arg", "0.1");
            return defaults;
        });
    }

    @Override
    public int order() {
        return 100;  // 在默认定制器之后执行
    }

    private String getDeploymentVersion() {
        // 从环境或构建信息读取版本
        return System.getenv().getOrDefault("APP_VERSION", "1.0.0");
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider`

```
com.example.ProductionCustomizerProvider
```

**执行时机**: 在 SDK 构建开始前，所有 `AutoConfigurationCustomizerProvider` 被加载并按 `order()` 顺序执行。

**最佳实践**:
- ✅ 使用 `order()` 控制多个定制器的执行顺序
- ✅ 定制器应该是幂等的（多次调用产生相同结果）
- ✅ 避免在定制器中执行耗时操作
- ✅ 使用 `ConfigProperties` 读取配置，而不是直接访问环境变量

---

#### AutoConfigurationCustomizer

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizer`

**稳定性**: ✅ 公共 API（稳定）

**目的**: 提供定制自动配置的构建接口，包含所有可定制的 SDK 组件。

**使用场景**: 通过 `AutoConfigurationCustomizerProvider` 访问，用于定制 SDK 组件。

**接口定义** (精简版，共 14 个方法):
```java
public interface AutoConfigurationCustomizer {
    // 传播器定制
    AutoConfigurationCustomizer addPropagatorCustomizer(
        BiFunction<? super TextMapPropagator, ConfigProperties, ? extends TextMapPropagator> customizer
    );

    // Resource 定制
    AutoConfigurationCustomizer addResourceCustomizer(
        BiFunction<? super Resource, ConfigProperties, ? extends Resource> customizer
    );

    // 采样器定制
    AutoConfigurationCustomizer addSamplerCustomizer(
        BiFunction<? super Sampler, ConfigProperties, ? extends Sampler> customizer
    );

    // Span 导出器定制
    AutoConfigurationCustomizer addSpanExporterCustomizer(
        BiFunction<? super SpanExporter, ConfigProperties, ? extends SpanExporter> customizer
    );

    // Span 处理器定制
    AutoConfigurationCustomizer addSpanProcessorCustomizer(
        BiFunction<? super SpanProcessor, ConfigProperties, ? extends SpanProcessor> customizer
    );

    // TracerProvider 定制
    AutoConfigurationCustomizer addTracerProviderCustomizer(
        BiFunction<SdkTracerProviderBuilder, ConfigProperties, SdkTracerProviderBuilder> customizer
    );

    // Metric 导出器定制
    AutoConfigurationCustomizer addMetricExporterCustomizer(
        BiFunction<? super MetricExporter, ConfigProperties, ? extends MetricExporter> customizer
    );

    // Metric 读取器定制
    AutoConfigurationCustomizer addMetricReaderCustomizer(
        BiFunction<? super MetricReader, ConfigProperties, ? extends MetricReader> customizer
    );

    // MeterProvider 定制
    AutoConfigurationCustomizer addMeterProviderCustomizer(
        BiFunction<SdkMeterProviderBuilder, ConfigProperties, SdkMeterProviderBuilder> customizer
    );

    // 日志导出器定制
    AutoConfigurationCustomizer addLogRecordExporterCustomizer(
        BiFunction<? super LogRecordExporter, ConfigProperties, ? extends LogRecordExporter> customizer
    );

    // 日志处理器定制
    AutoConfigurationCustomizer addLogRecordProcessorCustomizer(
        BiFunction<? super LogRecordProcessor, ConfigProperties, ? extends LogRecordProcessor> customizer
    );

    // LoggerProvider 定制
    AutoConfigurationCustomizer addLoggerProviderCustomizer(
        BiFunction<SdkLoggerProviderBuilder, ConfigProperties, SdkLoggerProviderBuilder> customizer
    );

    // 配置属性提供者
    AutoConfigurationCustomizer addPropertiesSupplier(
        Supplier<Map<String, String>> propertiesSupplier
    );

    // 配置属性定制
    AutoConfigurationCustomizer addPropertiesCustomizer(
        Function<ConfigProperties, Map<String, String>> propertiesCustomizer
    );
}
```

**定制器类型**:

1. **组件定制器** (`BiFunction<Component, ConfigProperties, Component>`):
   - 输入：自动配置的组件实例
   - 输出：定制后的组件实例（可以是包装器）
   - 示例：`addSpanExporterCustomizer((exporter, config) -> new LoggingWrapper(exporter))`

2. **Builder 定制器** (`BiFunction<Builder, ConfigProperties, Builder>`):
   - 输入：SDK Builder 实例
   - 输出：定制后的 Builder
   - 示例：`addTracerProviderCustomizer((builder, config) -> builder.setSampler(...))`

**执行顺序**:
```
自动配置流程
    ↓
1. 加载配置属性
    ↓
2. 应用 addPropertiesSupplier
    ↓
3. 应用 addPropertiesCustomizer
    ↓
4. 创建 Resource
    ↓
5. 应用 addResourceCustomizer
    ↓
6. 创建 Propagators
    ↓
7. 应用 addPropagatorCustomizer
    ↓
8. 创建 Sampler
    ↓
9. 应用 addSamplerCustomizer
    ↓
10. 创建 SpanExporter
    ↓
11. 应用 addSpanExporterCustomizer
    ↓
12. 创建 SpanProcessor
    ↓
13. 应用 addSpanProcessorCustomizer
    ↓
... (Metrics、Logs 类似)
    ↓
14. 构建 SDK
```

**示例：链式定制器**:
```java
customizer
    // 定制器 1: 添加 Resource 属性
    .addResourceCustomizer((resource, config) ->
        resource.toBuilder()
            .put("attr1", "value1")
            .build())
    // 定制器 2: 继续添加属性（在定制器 1 之后）
    .addResourceCustomizer((resource, config) ->
        resource.toBuilder()
            .put("attr2", "value2")
            .build());
// 最终 Resource 包含: attr1=value1, attr2=value2
```

---

#### ResourceProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供 Resource 属性，Resource 包含服务、进程、主机等元数据。

**使用场景**:
- 添加自定义 Resource 属性（如 Git commit、构建版本）
- 检测运行环境（Kubernetes、AWS、GCP）
- 添加业务相关的元数据

**接口定义**:
```java
public interface ResourceProvider extends Ordered {
    /**
     * 创建 Resource
     * @param config 配置属性
     * @return Resource 实例
     */
    Resource createResource(ConfigProperties config);

    /**
     * 执行顺序（可选）
     * @return 顺序值，数值越大越晚执行
     */
    default int order() {
        return 0;
    }
}
```

**完整实现示例**:
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;

import java.io.BufferedReader;
import java.io.InputStreamReader;

/**
 * 添加 Git 和构建信息到 Resource
 */
public class GitResourceProvider implements ResourceProvider {

    @Override
    public Resource createResource(ConfigProperties config) {
        Attributes.Builder builder = Attributes.builder();

        // 添加 Git commit hash
        String gitCommit = getGitCommit();
        if (gitCommit != null) {
            builder.put(AttributeKey.stringKey("vcs.commit.id"), gitCommit);
        }

        // 添加 Git branch
        String gitBranch = getGitBranch();
        if (gitBranch != null) {
            builder.put(AttributeKey.stringKey("vcs.branch"), gitBranch);
        }

        // 添加构建时间
        String buildTime = System.getenv("BUILD_TIME");
        if (buildTime != null) {
            builder.put(AttributeKey.stringKey("build.time"), buildTime);
        }

        // 添加构建版本
        String buildVersion = System.getenv("BUILD_VERSION");
        if (buildVersion != null) {
            builder.put(AttributeKey.stringKey("service.version"), buildVersion);
        }

        return Resource.create(builder.build());
    }

    @Override
    public int order() {
        return 100;  // 在默认 ResourceProvider 之后执行
    }

    private String getGitCommit() {
        try {
            Process process = Runtime.getRuntime().exec("git rev-parse HEAD");
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream())
            );
            return reader.readLine();
        } catch (Exception e) {
            return null;
        }
    }

    private String getGitBranch() {
        try {
            Process process = Runtime.getRuntime().exec("git rev-parse --abbrev-ref HEAD");
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream())
            );
            return reader.readLine();
        } catch (Exception e) {
            return null;
        }
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider`

```
com.example.GitResourceProvider
```

**Resource 合并**:

多个 ResourceProvider 的结果会被合并：
```
Resource.getDefault()                    # 默认 Resource (SDK 版本等)
    .merge(OsResourceProvider)           # 操作系统信息
    .merge(ProcessResourceProvider)      # 进程信息
    .merge(HostResourceProvider)         # 主机信息
    .merge(GitResourceProvider)          # 自定义 Git 信息
    .merge(config.getResource())         # 环境变量 Resource
```

**合并规则**:
- 后执行的 Provider 可以覆盖前面的属性
- `order()` 值越大，越晚执行（优先级越高）
- 使用 `order()` 控制属性覆盖行为

**最佳实践**:
- ✅ 使用标准的语义约定属性名（`service.*`, `host.*`, `process.*`）
- ✅ 检查属性是否可用再添加（避免 null 值）
- ✅ 避免在 `createResource()` 中执行耗时操作
- ✅ 使用 `order()` 确保属性按正确顺序覆盖

---

#### ConfigurablePropagatorProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.ConfigurablePropagatorProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供自定义的 Context 传播器，用于跨进程传播 Trace Context 和 Baggage。

**使用场景**:
- 支持非标准的传播协议（如自定义 HTTP header）
- 支持遗留系统的传播格式
- 添加额外的传播逻辑

**相关环境变量**: `OTEL_PROPAGATORS`（逗号分隔的传播器名称列表）

**接口定义**:
```java
public interface ConfigurablePropagatorProvider {
    /**
     * 获取传播器
     * @param config 配置属性
     * @return TextMapPropagator 实例
     */
    TextMapPropagator getPropagator(ConfigProperties config);

    /**
     * 传播器名称（用于 OTEL_PROPAGATORS）
     * @return 名称
     */
    String getName();
}
```

**完整实现示例**:
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigurablePropagatorProvider;
import io.opentelemetry.context.propagation.TextMapGetter;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.context.propagation.TextMapSetter;
import io.opentelemetry.context.Context;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.SpanContext;
import io.opentelemetry.api.trace.TraceFlags;
import io.opentelemetry.api.trace.TraceState;

import java.util.Arrays;
import java.util.Collection;

/**
 * 自定义传播器：支持 X-Custom-Trace-Id header
 */
public class CustomPropagatorProvider implements ConfigurablePropagatorProvider {

    @Override
    public TextMapPropagator getPropagator(ConfigProperties config) {
        return new CustomPropagator();
    }

    @Override
    public String getName() {
        return "custom";  // 使用: OTEL_PROPAGATORS=custom 或 OTEL_PROPAGATORS=tracecontext,custom
    }

    private static class CustomPropagator implements TextMapPropagator {
        private static final String TRACE_ID_HEADER = "X-Custom-Trace-Id";
        private static final String SPAN_ID_HEADER = "X-Custom-Span-Id";

        @Override
        public Collection<String> fields() {
            return Arrays.asList(TRACE_ID_HEADER, SPAN_ID_HEADER);
        }

        @Override
        public <C> void inject(Context context, C carrier, TextMapSetter<C> setter) {
            Span span = Span.fromContext(context);
            SpanContext spanContext = span.getSpanContext();

            if (spanContext.isValid()) {
                // 注入自定义 headers
                setter.set(carrier, TRACE_ID_HEADER, spanContext.getTraceId());
                setter.set(carrier, SPAN_ID_HEADER, spanContext.getSpanId());
            }
        }

        @Override
        public <C> Context extract(Context context, C carrier, TextMapGetter<C> getter) {
            String traceId = getter.get(carrier, TRACE_ID_HEADER);
            String spanId = getter.get(carrier, SPAN_ID_HEADER);

            if (traceId != null && spanId != null) {
                // 从自定义 headers 提取 SpanContext
                SpanContext spanContext = SpanContext.create(
                    traceId,
                    spanId,
                    TraceFlags.getSampled(),
                    TraceState.getDefault()
                );

                // 创建包含提取 SpanContext 的 Context
                return context.with(Span.wrap(spanContext));
            }

            return context;
        }
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.ConfigurablePropagatorProvider`

```
com.example.CustomPropagatorProvider
```

**配置使用**:
```bash
# 使用自定义传播器
export OTEL_PROPAGATORS=custom

# 组合使用（先 W3C Trace Context，再自定义）
export OTEL_PROPAGATORS=tracecontext,custom

# 组合使用（W3C Trace Context + Baggage + 自定义）
export OTEL_PROPAGATORS=tracecontext,baggage,custom
```

**内置传播器**:
- `tracecontext` - W3C Trace Context（推荐）
- `baggage` - W3C Baggage
- `b3` - B3 Single Header
- `b3multi` - B3 Multi Header
- `jaeger` - Jaeger Propagator
- `ottrace` - OT Trace

**最佳实践**:
- ✅ 实现 `fields()` 方法返回所有相关的 header 名称
- ✅ 验证提取的数据有效性（TraceId、SpanId 格式）
- ✅ 处理缺失或无效的 headers（返回原 Context）
- ✅ 考虑与标准传播器组合使用

---

### Traces 信号 SPI

#### ConfigurableSpanExporterProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供自定义的 Span 导出器，将 Span 数据发送到自定义后端。

**使用场景**:
- 导出到自定义后端（HTTP、Kafka、数据库等）
- 实现自定义格式的导出器
- 支持特殊协议或认证方式

**相关环境变量**: `OTEL_TRACES_EXPORTER`（导出器名称，可以是逗号分隔的多个导出器）

**接口定义**:
```java
public interface ConfigurableSpanExporterProvider {
    /**
     * 创建 Span 导出器
     * @param config 配置属性
     * @return SpanExporter 实例
     */
    SpanExporter createExporter(ConfigProperties config);

    /**
     * 导出器名称（用于 OTEL_TRACES_EXPORTER）
     * @return 名称
     */
    String getName();
}
```

**完整实现示例** (已在"快速开始"中展示，此处展示高级特性):
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;

import java.util.Collection;
import java.util.concurrent.TimeUnit;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URI;

/**
 * HTTP JSON 导出器（带重试和连接池）
 */
public class HttpJsonSpanExporterProvider implements ConfigurableSpanExporterProvider {

    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        String endpoint = config.getString("otel.exporter.httpjson.endpoint");
        if (endpoint == null) {
            throw new IllegalArgumentException("otel.exporter.httpjson.endpoint is required");
        }

        int timeout = config.getInt("otel.exporter.httpjson.timeout", 10000);
        int maxRetries = config.getInt("otel.exporter.httpjson.max.retries", 3);
        String authToken = config.getString("otel.exporter.httpjson.auth.token");

        return new HttpJsonSpanExporter(endpoint, timeout, maxRetries, authToken);
    }

    @Override
    public String getName() {
        return "httpjson";
    }

    private static class HttpJsonSpanExporter implements SpanExporter {
        private final String endpoint;
        private final int timeoutMs;
        private final int maxRetries;
        private final String authToken;
        private final HttpClient httpClient;

        public HttpJsonSpanExporter(String endpoint, int timeoutMs, int maxRetries, String authToken) {
            this.endpoint = endpoint;
            this.timeoutMs = timeoutMs;
            this.maxRetries = maxRetries;
            this.authToken = authToken;

            // 创建 HTTP 客户端（带连接池）
            this.httpClient = HttpClient.newBuilder()
                .connectTimeout(java.time.Duration.ofMillis(timeoutMs))
                .build();
        }

        @Override
        public CompletableResultCode export(Collection<SpanData> spans) {
            CompletableResultCode result = new CompletableResultCode();

            // 异步导出
            CompletableFuture.runAsync(() -> {
                try {
                    // 转换为 JSON
                    String json = convertToJson(spans);

                    // 发送 HTTP 请求（带重试）
                    boolean success = sendWithRetry(json);

                    if (success) {
                        result.succeed();
                    } else {
                        result.fail();
                    }
                } catch (Exception e) {
                    System.err.println("Export failed: " + e.getMessage());
                    result.fail();
                }
            });

            return result;
        }

        private boolean sendWithRetry(String json) {
            int retries = 0;
            while (retries < maxRetries) {
                try {
                    // 构建 HTTP 请求
                    HttpRequest.Builder requestBuilder = HttpRequest.newBuilder()
                        .uri(URI.create(endpoint))
                        .header("Content-Type", "application/json")
                        .POST(HttpRequest.BodyPublishers.ofString(json));

                    // 添加认证 header
                    if (authToken != null) {
                        requestBuilder.header("Authorization", "Bearer " + authToken);
                    }

                    HttpRequest request = requestBuilder.build();

                    // 发送请求
                    HttpResponse<String> response = httpClient.send(
                        request,
                        HttpResponse.BodyHandlers.ofString()
                    );

                    // 检查响应状态
                    if (response.statusCode() >= 200 && response.statusCode() < 300) {
                        return true;  // 成功
                    } else if (response.statusCode() >= 500) {
                        // 5xx 错误：重试
                        retries++;
                        Thread.sleep((long) Math.pow(2, retries) * 1000);  // 指数退避
                    } else {
                        // 4xx 错误：不重试
                        System.err.println("HTTP error: " + response.statusCode());
                        return false;
                    }
                } catch (Exception e) {
                    retries++;
                    if (retries >= maxRetries) {
                        System.err.println("Max retries reached: " + e.getMessage());
                        return false;
                    }
                    try {
                        Thread.sleep((long) Math.pow(2, retries) * 1000);  // 指数退避
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        return false;
                    }
                }
            }
            return false;
        }

        private String convertToJson(Collection<SpanData> spans) {
            // 简化的 JSON 转换（实际应使用 JSON 库）
            StringBuilder json = new StringBuilder("[");
            boolean first = true;
            for (SpanData span : spans) {
                if (!first) json.append(",");
                json.append(String.format(
                    "{\"traceId\":\"%s\",\"spanId\":\"%s\",\"name\":\"%s\"}",
                    span.getTraceId(),
                    span.getSpanId(),
                    span.getName()
                ));
                first = false;
            }
            json.append("]");
            return json.toString();
        }

        @Override
        public CompletableResultCode flush() {
            // 刷新缓冲区（如果有）
            return CompletableResultCode.ofSuccess();
        }

        @Override
        public CompletableResultCode shutdown() {
            // 关闭 HTTP 客户端
            System.out.println("Shutting down HttpJsonSpanExporter");
            return CompletableResultCode.ofSuccess();
        }
    }
}
```

**注册和配置**:

```
# SPI 注册文件
META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider
内容: com.example.HttpJsonSpanExporterProvider

# 环境变量配置
export OTEL_TRACES_EXPORTER=httpjson
export OTEL_EXPORTER_HTTPJSON_ENDPOINT=https://my-backend.com/api/spans
export OTEL_EXPORTER_HTTPJSON_TIMEOUT=15000
export OTEL_EXPORTER_HTTPJSON_MAX_RETRIES=5
export OTEL_EXPORTER_HTTPJSON_AUTH_TOKEN=secret-token
```

**多导出器**:
```bash
# 同时导出到 OTLP 和自定义后端
export OTEL_TRACES_EXPORTER=otlp,httpjson
```

**最佳实践**:
- ✅ 实现异步导出（返回 `CompletableResultCode`）
- ✅ 实现重试机制（网络失败、5xx 错误）
- ✅ 实现连接池和超时控制
- ✅ 正确实现 `shutdown()` 清理资源
- ✅ 使用 `ConfigProperties` 读取配置
- ✅ 处理序列化错误（JSON 转换失败）

---

#### ConfigurableSamplerProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供自定义采样器，控制哪些 Span 被记录和导出。

**使用场景**:
- 实现自定义采样逻辑（如基于属性采样）
- 实现动态采样率
- 支持远程采样配置

**相关环境变量**:
- `OTEL_TRACES_SAMPLER`（采样器名称）
- `OTEL_TRACES_SAMPLER_ARG`（采样器参数，通常是采样率）

**接口定义**:
```java
public interface ConfigurableSamplerProvider {
    /**
     * 创建采样器
     * @param config 配置属性
     * @return Sampler 实例
     */
    Sampler createSampler(ConfigProperties config);

    /**
     * 采样器名称（用于 OTEL_TRACES_SAMPLER）
     * @return 名称
     */
    String getName();
}
```

**完整实现示例**:
```java
package com.example;

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
 * 自定义采样器：优先采样标记为"priority"的 Span
 */
public class PrioritySamplerProvider implements ConfigurableSamplerProvider {

    @Override
    public Sampler createSampler(ConfigProperties config) {
        // 读取基础采样率
        double fallbackRate = config.getDouble("otel.traces.sampler.arg", 0.1);

        // 读取是否启用 debug 模式
        boolean debugMode = config.getBoolean("otel.traces.sampler.debug", false);

        return new PrioritySampler(fallbackRate, debugMode);
    }

    @Override
    public String getName() {
        return "priority";  // 使用: OTEL_TRACES_SAMPLER=priority
    }

    private static class PrioritySampler implements Sampler {
        private final double fallbackRate;
        private final boolean debugMode;
        private final Sampler fallbackSampler;

        public PrioritySampler(double fallbackRate, boolean debugMode) {
            this.fallbackRate = fallbackRate;
            this.debugMode = debugMode;
            this.fallbackSampler = Sampler.traceIdRatioBased(fallbackRate);
        }

        @Override
        public SamplingResult shouldSample(
            Context parentContext,
            String traceId,
            String name,
            SpanKind spanKind,
            Attributes attributes,
            List<LinkData> parentLinks
        ) {
            // 规则 1: Debug 模式 - 100% 采样
            if (debugMode) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 2: 带有 "priority" 属性 - 100% 采样
            Boolean priority = attributes.get(AttributeKey.booleanKey("priority"));
            if (Boolean.TRUE.equals(priority)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 3: 错误 Span - 100% 采样
            Boolean isError = attributes.get(AttributeKey.booleanKey("error"));
            if (Boolean.TRUE.equals(isError)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 4: 特定端点 - 更高采样率
            String endpoint = attributes.get(AttributeKey.stringKey("http.target"));
            if ("/api/critical".equals(endpoint)) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            // 规则 5: 默认 - 使用基础采样率
            return fallbackSampler.shouldSample(
                parentContext,
                traceId,
                name,
                spanKind,
                attributes,
                parentLinks
            );
        }

        @Override
        public String getDescription() {
            return String.format("PrioritySampler{fallbackRate=%f, debugMode=%b}",
                fallbackRate, debugMode);
        }
    }
}
```

**注册和配置**:

```
# SPI 注册文件
META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider
内容: com.example.PrioritySamplerProvider

# 环境变量配置
export OTEL_TRACES_SAMPLER=priority
export OTEL_TRACES_SAMPLER_ARG=0.1        # 基础采样率 10%
export OTEL_TRACES_SAMPLER_DEBUG=false    # 禁用 debug 模式
```

**内置采样器**:
- `always_on` - 100% 采样
- `always_off` - 0% 采样
- `traceidratio` - 基于 TraceId 的比例采样
- `parentbased_always_on` - 继承父 Span，根 Span 100% 采样
- `parentbased_always_off` - 继承父 Span，根 Span 0% 采样
- `parentbased_traceidratio` - 继承父 Span，根 Span 比例采样

**最佳实践**:
- ✅ 实现高效的采样决策（避免耗时操作）
- ✅ 考虑父 Span 的采样决策（继承采样状态）
- ✅ 使用 `SamplingDecision.RECORD_AND_SAMPLE` 表示采样
- ✅ 使用 `SamplingDecision.DROP` 表示不采样
- ✅ 使用 `SamplingDecision.RECORD_ONLY` 表示记录但不导出
- ✅ 提供清晰的 `getDescription()` 用于调试

---

### Metrics 信号 SPI

#### ConfigurableMetricExporterProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.metrics.ConfigurableMetricExporterProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供自定义的 Metric 导出器，将 Metric 数据推送到自定义后端。

**使用场景**:
- 导出到自定义时序数据库
- 实现自定义格式的 Metric 导出
- Push 模式的 Metric 导出

**相关环境变量**: `OTEL_METRICS_EXPORTER`（导出器名称，可以是逗号分隔的多个导出器）

**接口定义**:
```java
public interface ConfigurableMetricExporterProvider {
    /**
     * 创建 Metric 导出器
     * @param config 配置属性
     * @return MetricExporter 实例
     */
    MetricExporter createExporter(ConfigProperties config);

    /**
     * 导出器名称（用于 OTEL_METRICS_EXPORTER）
     * @return 名称
     */
    String getName();
}
```

**完整实现示例**:
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.metrics.ConfigurableMetricExporterProvider;
import io.opentelemetry.sdk.metrics.export.MetricExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.metrics.data.MetricData;
import io.opentelemetry.sdk.metrics.data.PointData;
import io.opentelemetry.sdk.metrics.data.LongPointData;
import io.opentelemetry.sdk.metrics.data.DoublePointData;

import java.util.Collection;

/**
 * InfluxDB Metric 导出器
 */
public class InfluxDBMetricExporterProvider implements ConfigurableMetricExporterProvider {

    @Override
    public MetricExporter createExporter(ConfigProperties config) {
        String url = config.getString("otel.exporter.influxdb.url");
        if (url == null) {
            throw new IllegalArgumentException("otel.exporter.influxdb.url is required");
        }

        String database = config.getString("otel.exporter.influxdb.database", "opentelemetry");
        String username = config.getString("otel.exporter.influxdb.username");
        String password = config.getString("otel.exporter.influxdb.password");

        return new InfluxDBMetricExporter(url, database, username, password);
    }

    @Override
    public String getName() {
        return "influxdb";
    }

    private static class InfluxDBMetricExporter implements MetricExporter {
        private final String url;
        private final String database;
        private final String username;
        private final String password;

        public InfluxDBMetricExporter(String url, String database, String username, String password) {
            this.url = url;
            this.database = database;
            this.username = username;
            this.password = password;
        }

        @Override
        public CompletableResultCode export(Collection<MetricData> metrics) {
            CompletableResultCode result = new CompletableResultCode();

            try {
                // 转换为 InfluxDB Line Protocol 格式
                String lineProtocol = convertToLineProtocol(metrics);

                // 发送到 InfluxDB
                boolean success = sendToInfluxDB(lineProtocol);

                if (success) {
                    result.succeed();
                } else {
                    result.fail();
                }
            } catch (Exception e) {
                System.err.println("Export failed: " + e.getMessage());
                result.fail();
            }

            return result;
        }

        private String convertToLineProtocol(Collection<MetricData> metrics) {
            StringBuilder lines = new StringBuilder();

            for (MetricData metric : metrics) {
                String measurement = metric.getName();

                for (PointData point : metric.getData().getPoints()) {
                    // InfluxDB Line Protocol 格式:
                    // measurement,tag1=value1,tag2=value2 field1=value1,field2=value2 timestamp

                    // Tags（来自 Attributes）
                    StringBuilder tags = new StringBuilder();
                    point.getAttributes().forEach((key, value) -> {
                        tags.append(",").append(key).append("=").append(value);
                    });

                    // Fields（Metric 值）
                    String field;
                    if (point instanceof LongPointData) {
                        long value = ((LongPointData) point).getValue();
                        field = "value=" + value + "i";
                    } else if (point instanceof DoublePointData) {
                        double value = ((DoublePointData) point).getValue();
                        field = "value=" + value;
                    } else {
                        continue;
                    }

                    // Timestamp（纳秒）
                    long timestamp = point.getEpochNanos();

                    // 组装 Line Protocol
                    lines.append(measurement)
                         .append(tags)
                         .append(" ")
                         .append(field)
                         .append(" ")
                         .append(timestamp)
                         .append("\n");
                }
            }

            return lines.toString();
        }

        private boolean sendToInfluxDB(String lineProtocol) {
            // 实际实现：发送 HTTP POST 到 InfluxDB API
            // POST /write?db={database}
            // Authorization: Basic {base64(username:password)}
            // Body: {lineProtocol}

            System.out.println("Sending to InfluxDB:");
            System.out.println(lineProtocol);

            return true;  // 简化示例
        }

        @Override
        public CompletableResultCode flush() {
            return CompletableResultCode.ofSuccess();
        }

        @Override
        public CompletableResultCode shutdown() {
            System.out.println("Shutting down InfluxDBMetricExporter");
            return CompletableResultCode.ofSuccess();
        }

        @Override
        public AggregationTemporality getAggregationTemporality(InstrumentType instrumentType) {
            // InfluxDB 使用 Cumulative（累计）模式
            return AggregationTemporality.CUMULATIVE;
        }
    }
}
```

**注册和配置**:

```
# SPI 注册文件
META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.metrics.ConfigurableMetricExporterProvider
内容: com.example.InfluxDBMetricExporterProvider

# 环境变量配置
export OTEL_METRICS_EXPORTER=influxdb
export OTEL_EXPORTER_INFLUXDB_URL=http://localhost:8086
export OTEL_EXPORTER_INFLUXDB_DATABASE=opentelemetry
export OTEL_EXPORTER_INFLUXDB_USERNAME=admin
export OTEL_EXPORTER_INFLUXDB_PASSWORD=secret
```

**AggregationTemporality（聚合时间性）**:
- `CUMULATIVE`（累计）：每次导出累计值（Prometheus 模式）
- `DELTA`（增量）：每次导出增量值（StatsD 模式）

**最佳实践**:
- ✅ 实现 `getAggregationTemporality()` 指定聚合模式
- ✅ 处理不同类型的 Metric（Counter、Histogram、Gauge）
- ✅ 正确转换 Attributes 到目标格式的 Tags
- ✅ 处理 Histogram 的桶数据
- ✅ 实现异步导出避免阻塞

---

### Logs 信号 SPI

#### ConfigurableLogRecordExporterProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.logs.ConfigurableLogRecordExporterProvider`

**稳定性**: ✅ 公共 SPI（稳定）

**目的**: 提供自定义的日志导出器，将日志数据发送到自定义后端。

**使用场景**:
- 导出到自定义日志系统
- 实现自定义日志格式
- 集成遗留日志系统

**相关环境变量**: `OTEL_LOGS_EXPORTER`（导出器名称）

**接口定义**:
```java
public interface ConfigurableLogRecordExporterProvider {
    /**
     * 创建日志导出器
     * @param config 配置属性
     * @return LogRecordExporter 实例
     */
    LogRecordExporter createExporter(ConfigProperties config);

    /**
     * 导出器名称（用于 OTEL_LOGS_EXPORTER）
     * @return 名称
     */
    String getName();
}
```

**完整实现示例**:
```java
package com.example;

import io.opentelemetry.sdk.autoconfigure.spi.ConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.logs.ConfigurableLogRecordExporterProvider;
import io.opentelemetry.sdk.logs.export.LogRecordExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.logs.data.LogRecordData;

import java.util.Collection;
import java.io.FileWriter;
import java.io.IOException;
import java.time.Instant;

/**
 * 文件日志导出器（JSON Lines 格式）
 */
public class FileLogExporterProvider implements ConfigurableLogRecordExporterProvider {

    @Override
    public LogRecordExporter createExporter(ConfigProperties config) {
        String filePath = config.getString("otel.exporter.filelog.path");
        if (filePath == null) {
            throw new IllegalArgumentException("otel.exporter.filelog.path is required");
        }

        boolean append = config.getBoolean("otel.exporter.filelog.append", true);

        return new FileLogExporter(filePath, append);
    }

    @Override
    public String getName() {
        return "filelog";
    }

    private static class FileLogExporter implements LogRecordExporter {
        private final String filePath;
        private final boolean append;
        private FileWriter writer;

        public FileLogExporter(String filePath, boolean append) {
            this.filePath = filePath;
            this.append = append;

            try {
                this.writer = new FileWriter(filePath, append);
            } catch (IOException e) {
                throw new RuntimeException("Failed to open log file: " + filePath, e);
            }
        }

        @Override
        public CompletableResultCode export(Collection<LogRecordData> logs) {
            CompletableResultCode result = new CompletableResultCode();

            try {
                for (LogRecordData log : logs) {
                    // 转换为 JSON
                    String json = convertToJson(log);

                    // 写入文件（JSON Lines 格式）
                    writer.write(json);
                    writer.write("\n");
                }

                writer.flush();
                result.succeed();
            } catch (IOException e) {
                System.err.println("Failed to export logs: " + e.getMessage());
                result.fail();
            }

            return result;
        }

        private String convertToJson(LogRecordData log) {
            // 简化的 JSON 转换（实际应使用 JSON 库）
            return String.format(
                "{\"timestamp\":\"%s\",\"severity\":\"%s\",\"body\":\"%s\",\"traceId\":\"%s\",\"spanId\":\"%s\"}",
                Instant.ofEpochMilli(log.getTimestampEpochNanos() / 1_000_000),
                log.getSeverity(),
                log.getBody().asString(),
                log.getSpanContext().getTraceId(),
                log.getSpanContext().getSpanId()
            );
        }

        @Override
        public CompletableResultCode flush() {
            try {
                writer.flush();
                return CompletableResultCode.ofSuccess();
            } catch (IOException e) {
                return CompletableResultCode.ofFailure();
            }
        }

        @Override
        public CompletableResultCode shutdown() {
            try {
                writer.close();
                System.out.println("Closed FileLogExporter");
                return CompletableResultCode.ofSuccess();
            } catch (IOException e) {
                return CompletableResultCode.ofFailure();
            }
        }
    }
}
```

**注册和配置**:

```
# SPI 注册文件
META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.logs.ConfigurableLogRecordExporterProvider
内容: com.example.FileLogExporterProvider

# 环境变量配置
export OTEL_LOGS_EXPORTER=filelog
export OTEL_EXPORTER_FILELOG_PATH=/var/log/otel/logs.jsonl
export OTEL_EXPORTER_FILELOG_APPEND=true
```

**最佳实践**:
- ✅ 正确处理 `traceId` 和 `spanId`（关联到 Trace）
- ✅ 实现文件轮转（避免文件过大）
- ✅ 处理 Severity 级别映射
- ✅ 正确实现 `flush()` 和 `shutdown()`
- ✅ 处理 Body 的不同类型（String、JSON）

---

### 配置基础设施

#### Ordered

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.Ordered`

**稳定性**: ✅ 公共接口（稳定）

**目的**: 提供控制 SPI 加载和执行顺序的标准接口。

**使用场景**:
- 控制多个 ResourceProvider 的执行顺序
- 确保后续提供者可以覆盖前面的配置
- 协调多个扩展之间的依赖关系

**接口定义**:
```java
public interface Ordered {
    /**
     * 返回执行顺序值
     * @return 顺序值，数值越大越晚执行（优先级越高）
     */
    default int order() {
        return 0;
    }
}
```

**顺序规则**:
- **数值越小，越早执行**：order() = -100 会在 order() = 0 之前执行
- **数值越大，越晚执行**：order() = 100 会在 order() = 0 之后执行
- **后执行的优先级更高**：可以覆盖前面的配置
- **默认值为 0**：不实现 order() 或返回 0 的组件使用默认顺序

**示例：Resource 属性覆盖**:
```java
// Provider 1: order() = 0 (默认)
public class DefaultResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("environment"), "development"
        ));
    }
    // order() = 0 (默认)
}

// Provider 2: order() = 100 (更高优先级)
public class ProductionResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("environment"), "production"
        ));
    }

    @Override
    public int order() {
        return 100;  // 在 DefaultResourceProvider 之后执行
    }
}

// 最终结果: environment=production (被 ProductionResourceProvider 覆盖)
```

**实际应用场景**:

1. **环境特定配置**:
```java
// 基础配置 (order = 0)
public class BaseResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("service.name"), "my-service",
            AttributeKey.stringKey("deployment.environment"), "unknown"
        ));
    }
}

// 环境检测 (order = 50)
public class EnvironmentDetectorProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        String env = detectEnvironment();
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("deployment.environment"), env
        ));
    }

    @Override
    public int order() {
        return 50;
    }
}

// 手动覆盖 (order = 100)
public class ManualOverrideProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        String manualEnv = config.getString("custom.environment");
        if (manualEnv != null) {
            return Resource.create(Attributes.of(
                AttributeKey.stringKey("deployment.environment"), manualEnv
            ));
        }
        return Resource.empty();
    }

    @Override
    public int order() {
        return 100;  // 最高优先级
    }
}
```

2. **AutoConfigurationCustomizerProvider 顺序**:
```java
// 基础定制器 (order = 0)
public class BaseCustomizerProvider implements AutoConfigurationCustomizerProvider {
    @Override
    public void customize(AutoConfigurationCustomizer customizer) {
        // 添加基础配置
        customizer.addPropertiesSupplier(() -> getBaseProperties());
    }
}

// 高级定制器 (order = 100)
public class AdvancedCustomizerProvider implements AutoConfigurationCustomizerProvider {
    @Override
    public void customize(AutoConfigurationCustomizer customizer) {
        // 覆盖基础配置
        customizer.addPropertiesCustomizer(props -> getAdvancedProperties());
    }

    @Override
    public int order() {
        return 100;
    }
}
```

**最佳实践**:
- ✅ 使用明确的顺序值（-100, 0, 50, 100）而不是随机数字
- ✅ 为后续覆盖保留空间（使用 100、200 而不是 1、2）
- ✅ 文档化顺序值的含义和依赖关系
- ✅ 避免使用极端值（Integer.MIN_VALUE、Integer.MAX_VALUE）

---

#### ConfigurationException

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.ConfigurationException`

**稳定性**: ✅ 公共类（稳定）

**目的**: 表示配置过程中发生的异常。

**使用场景**:
- 必需的配置属性缺失
- 配置属性值无效
- 无法创建组件（如连接失败）
- SPI 加载失败

**类定义**:
```java
public class ConfigurationException extends RuntimeException {
    public ConfigurationException(String message) {
        super(message);
    }

    public ConfigurationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

**使用示例**:

1. **必需配置缺失**:
```java
public class MySpanExporterProvider implements ConfigurableSpanExporterProvider {
    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        String endpoint = config.getString("otel.exporter.myexporter.endpoint");

        if (endpoint == null || endpoint.isEmpty()) {
            throw new ConfigurationException(
                "otel.exporter.myexporter.endpoint is required but not configured"
            );
        }

        return new MySpanExporter(endpoint);
    }

    @Override
    public String getName() {
        return "myexporter";
    }
}
```

2. **配置值无效**:
```java
public class MySamplerProvider implements ConfigurableSamplerProvider {
    @Override
    public Sampler createSampler(ConfigProperties config) {
        Double samplingRate = config.getDouble("otel.traces.sampler.arg");

        if (samplingRate == null) {
            throw new ConfigurationException(
                "otel.traces.sampler.arg is required for custom sampler"
            );
        }

        if (samplingRate < 0.0 || samplingRate > 1.0) {
            throw new ConfigurationException(
                "otel.traces.sampler.arg must be between 0.0 and 1.0, got: " + samplingRate
            );
        }

        return new MySampler(samplingRate);
    }

    @Override
    public String getName() {
        return "custom";
    }
}
```

3. **组件初始化失败**:
```java
public class DatabaseSpanExporterProvider implements ConfigurableSpanExporterProvider {
    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        String jdbcUrl = config.getString("otel.exporter.db.url");
        String username = config.getString("otel.exporter.db.username");
        String password = config.getString("otel.exporter.db.password");

        try {
            Connection connection = DriverManager.getConnection(jdbcUrl, username, password);
            return new DatabaseSpanExporter(connection);
        } catch (SQLException e) {
            throw new ConfigurationException(
                "Failed to connect to database: " + jdbcUrl,
                e
            );
        }
    }

    @Override
    public String getName() {
        return "database";
    }
}
```

4. **依赖组件不可用**:
```java
public class AdvancedResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        try {
            // 尝试加载可选依赖
            Class.forName("com.example.CloudMetadataProvider");

            CloudMetadataProvider provider = new CloudMetadataProvider();
            return provider.getResource();
        } catch (ClassNotFoundException e) {
            throw new ConfigurationException(
                "CloudMetadataProvider is required but not found on classpath. " +
                "Add dependency: com.example:cloud-metadata:1.0.0",
                e
            );
        }
    }

    @Override
    public int order() {
        return 50;
    }
}
```

**异常处理**:

```java
// 应用代码中捕获 ConfigurationException
import io.opentelemetry.sdk.autoconfigure.AutoConfiguredOpenTelemetrySdk;
import io.opentelemetry.sdk.autoconfigure.spi.ConfigurationException;

public class Application {
    public static void main(String[] args) {
        try {
            AutoConfiguredOpenTelemetrySdk sdk =
                AutoConfiguredOpenTelemetrySdk.initialize();

            // 正常启动
            runApplication(sdk);

        } catch (ConfigurationException e) {
            // 配置错误：提供清晰的错误信息
            System.err.println("OpenTelemetry configuration failed:");
            System.err.println("  " + e.getMessage());

            if (e.getCause() != null) {
                System.err.println("  Caused by: " + e.getCause().getMessage());
            }

            // 降级策略：使用 No-op 实现
            System.err.println("Using no-op OpenTelemetry implementation");
            OpenTelemetry noopOtel = OpenTelemetry.noop();
            runApplication(noopOtel);
        }
    }
}
```

**最佳实践**:
- ✅ 提供清晰的错误消息，说明问题和解决方案
- ✅ 包含相关的配置属性名称和值
- ✅ 使用 cause 参数保留原始异常信息
- ✅ 在无法恢复的配置错误时抛出异常
- ✅ 提供配置文档链接或示例

---

## 高级主题

### 多组件协调

当需要多个 SPI 实现协同工作时，使用 `Ordered` 接口和 `AutoConfigurationCustomizer` 进行协调。

**示例：分层 Resource 配置**

```java
// 层次 1: 默认值 (order = 0)
public class DefaultResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("service.name"), "unknown-service",
            AttributeKey.stringKey("service.version"), "0.0.0",
            AttributeKey.stringKey("deployment.environment"), "development"
        ));
    }
}

// 层次 2: 环境检测 (order = 50)
public class EnvironmentResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        Attributes.Builder builder = Attributes.builder();

        // 检测云环境
        if (isRunningOnAWS()) {
            builder.put("cloud.provider", "aws");
            builder.put("cloud.region", getAWSRegion());
        } else if (isRunningOnGCP()) {
            builder.put("cloud.provider", "gcp");
            builder.put("cloud.zone", getGCPZone());
        }

        // 检测容器环境
        if (isRunningInKubernetes()) {
            builder.put("k8s.pod.name", getPodName());
            builder.put("k8s.namespace.name", getNamespace());
        }

        return Resource.create(builder.build());
    }

    @Override
    public int order() {
        return 50;
    }
}

// 层次 3: 配置覆盖 (order = 100)
public class ConfigOverrideResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        Attributes.Builder builder = Attributes.builder();

        // 从配置读取覆盖值
        String serviceName = config.getString("custom.service.name");
        if (serviceName != null) {
            builder.put("service.name", serviceName);
        }

        String version = config.getString("custom.service.version");
        if (version != null) {
            builder.put("service.version", version);
        }

        String environment = config.getString("custom.deployment.environment");
        if (environment != null) {
            builder.put("deployment.environment", environment);
        }

        return Resource.create(builder.build());
    }

    @Override
    public int order() {
        return 100;  // 最高优先级
    }
}
```

**执行结果**:
```
最终 Resource 属性:
  service.name = "my-service"           # 来自 ConfigOverrideResourceProvider
  service.version = "1.2.3"             # 来自 ConfigOverrideResourceProvider
  deployment.environment = "production" # 来自 ConfigOverrideResourceProvider
  cloud.provider = "aws"                # 来自 EnvironmentResourceProvider
  cloud.region = "us-west-2"            # 来自 EnvironmentResourceProvider
  k8s.pod.name = "my-pod-xyz"           # 来自 EnvironmentResourceProvider
  k8s.namespace.name = "default"        # 来自 EnvironmentResourceProvider
```

### 条件组件加载

使用 `ConfigProperties` 实现条件加载：

```java
public class ConditionalSpanExporterProvider implements ConfigurableSpanExporterProvider {
    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        // 检查是否启用
        boolean enabled = config.getBoolean("otel.exporter.conditional.enabled", true);
        if (!enabled) {
            // 返回 No-op 导出器
            return SpanExporter.composite();
        }

        // 根据环境选择实现
        String environment = config.getString("otel.exporter.conditional.environment", "local");

        switch (environment) {
            case "production":
                return createProductionExporter(config);
            case "staging":
                return createStagingExporter(config);
            case "local":
                return createLocalExporter(config);
            default:
                throw new ConfigurationException(
                    "Unknown environment: " + environment
                );
        }
    }

    @Override
    public String getName() {
        return "conditional";
    }

    private SpanExporter createProductionExporter(ConfigProperties config) {
        // 生产环境：使用高性能 OTLP 导出器
        String endpoint = config.getString("otel.exporter.otlp.endpoint");
        return OtlpGrpcSpanExporter.builder()
            .setEndpoint(endpoint)
            .setCompression("gzip")
            .build();
    }

    private SpanExporter createStagingExporter(ConfigProperties config) {
        // 预发布环境：组合导出器（OTLP + 文件）
        SpanExporter otlpExporter = createProductionExporter(config);
        SpanExporter fileExporter = new FileSpanExporter("/tmp/spans.json");
        return SpanExporter.composite(otlpExporter, fileExporter);
    }

    private SpanExporter createLocalExporter(ConfigProperties config) {
        // 本地环境：控制台日志
        return LoggingSpanExporter.create();
    }
}
```

**配置示例**:
```bash
# 生产环境
export OTEL_TRACES_EXPORTER=conditional
export OTEL_EXPORTER_CONDITIONAL_ENVIRONMENT=production
export OTEL_EXPORTER_OTLP_ENDPOINT=https://api.example.com

# 本地开发
export OTEL_TRACES_EXPORTER=conditional
export OTEL_EXPORTER_CONDITIONAL_ENVIRONMENT=local

# 禁用导出器
export OTEL_EXPORTER_CONDITIONAL_ENABLED=false
```

---

## 内部 API

⚠️ **警告**: 以下接口标记为 `@Internal`，不保证 API 稳定性，可能随时变更。仅供 OpenTelemetry 内部使用和高级用户参考。

### ConditionalResourceProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.internal.ConditionalResourceProvider`

**稳定性**: ⚠️ 内部 API（不稳定）

**目的**: 扩展 `ResourceProvider`，增加条件判断，仅在特定条件下应用 Resource。

**接口定义**:
```java
public interface ConditionalResourceProvider extends ResourceProvider {
    /**
     * 判断是否应该应用此 ResourceProvider
     * @param config 配置属性
     * @param existing 当前已构建的 Resource
     * @return true 表示应用，false 表示跳过
     */
    boolean shouldApply(ConfigProperties config, Resource existing);
}
```

**使用场景**:
- 仅在特定环境下添加 Resource（如仅在 Kubernetes 中）
- 避免属性冲突（检查现有 Resource）
- 条件性覆盖（如仅在未设置时添加默认值）

**示例**:
```java
public class K8sConditionalResourceProvider implements ConditionalResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("k8s.pod.name"), System.getenv("POD_NAME"),
            AttributeKey.stringKey("k8s.namespace.name"), System.getenv("NAMESPACE")
        ));
    }

    @Override
    public boolean shouldApply(ConfigProperties config, Resource existing) {
        // 仅在 Kubernetes 环境中应用
        return System.getenv("KUBERNETES_SERVICE_HOST") != null;
    }

    @Override
    public int order() {
        return 50;
    }
}
```

---

### ConfigurableMetricReaderProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.internal.ConfigurableMetricReaderProvider`

**稳定性**: ⚠️ 内部 API（不稳定）

**目的**: 提供自定义的 MetricReader（拉取模式），与 MetricExporter（推送模式）不同。

**使用场景**:
- 实现 Pull-based Metric 导出（如 Prometheus）
- 自定义 Metric 收集间隔
- 集成拉取式监控系统

**接口定义**:
```java
public interface ConfigurableMetricReaderProvider {
    MetricReader createMetricReader(ConfigProperties config);
    String getName();
}
```

**与 ConfigurableMetricExporterProvider 的区别**:
- **MetricExporter**（推送模式）：SDK 主动推送 Metric 到后端
- **MetricReader**（拉取模式）：外部系统拉取 Metric（如 Prometheus scrape）

---

### ComponentProvider

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.internal.ComponentProvider`

**稳定性**: ⚠️ 内部 API（不稳定）

**目的**: 支持声明式配置（YAML 文件），为 YAML 配置中的组件提供工厂实现。

**使用场景**:
- 在 YAML 配置文件中定义自定义导出器、采样器、传播器等组件
- 扩展声明式配置系统以支持新的组件类型
- 替代环境变量配置，使用结构化的 YAML 配置

**接口定义**:
```java
public interface ComponentProvider<T> {
    /**
     * 从声明式配置创建组件实例
     * @param config 来自 YAML 文件的配置属性
     * @return 配置好的组件实例
     */
    T create(DeclarativeConfigProperties config);

    /**
     * 组件在 YAML 中的名称标识符
     * @return 名称（用于 YAML 文件中引用）
     */
    String getName();

    /**
     * 组件的类型
     * @return 组件类（如 SpanExporter.class）
     */
    Class<T> getType();
}
```

**与环境变量配置 SPI 的区别**:

| 特性 | ComponentProvider (声明式) | ConfigurableXxxProvider (环境变量) |
|------|---------------------------|----------------------------------|
| **配置方式** | YAML 文件 | 环境变量 |
| **配置复杂度** | 支持复杂嵌套结构 | 仅支持简单键值对 |
| **使用模块** | `sdk-extension-incubator` | `sdk-extension-autoconfigure` |
| **稳定性** | 实验性（@Internal） | 稳定公共 API |
| **示例** | `config.yaml` | `OTEL_TRACES_EXPORTER=otlp` |

**已注册的 ComponentProvider 实现**:

OpenTelemetry Java SDK 提供了以下内置 ComponentProvider:

```
OTLP 导出器 (exporters/otlp/all/):
  - OtlpGrpcSpanExporterComponentProvider (otlp_grpc)
  - OtlpGrpcMetricExporterComponentProvider
  - OtlpGrpcLogRecordExporterComponentProvider
  - OtlpHttpSpanExporterComponentProvider (otlp_http)
  - OtlpHttpMetricExporterComponentProvider
  - OtlpHttpLogRecordExporterComponentProvider

Prometheus 导出器 (exporters/prometheus/):
  - PrometheusComponentProvider (prometheus)

日志导出器 (exporters/logging/):
  - LoggingSpanExporterComponentProvider (logging)
  - LoggingMetricExporterComponentProvider
  - LoggingLogRecordExporterComponentProvider

传播器 (extensions/trace-propagators/):
  - B3PropagatorComponentProvider (b3, b3multi)
  - JaegerPropagatorComponentProvider (jaeger)
  - OtTracePropagatorComponentProvider (ottrace)

采样器 (sdk-extensions/jaeger-remote-sampler/):
  - JaegerRemoteSamplerComponentProvider (jaeger_remote)
```

**完整实现示例**:

```java
package com.example;

import io.opentelemetry.api.incubator.config.DeclarativeConfigProperties;
import io.opentelemetry.sdk.autoconfigure.spi.internal.ComponentProvider;
import io.opentelemetry.sdk.trace.export.SpanExporter;
import io.opentelemetry.sdk.common.CompletableResultCode;
import io.opentelemetry.sdk.trace.data.SpanData;

import java.util.Collection;

/**
 * 自定义 SpanExporter 的 ComponentProvider 实现
 * 用于 YAML 声明式配置
 */
public class CustomSpanExporterComponentProvider implements ComponentProvider<SpanExporter> {

    @Override
    public Class<SpanExporter> getType() {
        return SpanExporter.class;
    }

    @Override
    public String getName() {
        // 在 YAML 中使用的名称
        return "custom";
    }

    @Override
    public SpanExporter create(DeclarativeConfigProperties config) {
        // 从 YAML 配置读取属性
        String endpoint = config.getString("endpoint");
        if (endpoint == null) {
            throw new IllegalArgumentException("endpoint is required");
        }

        Integer timeout = config.getInt("timeout");
        Boolean compression = config.getBoolean("compression");

        // 读取嵌套配置
        DeclarativeConfigProperties retryConfig = config.getStructure("retry");
        int maxRetries = 3;
        if (retryConfig != null) {
            maxRetries = retryConfig.getInt("max_attempts", 3);
        }

        return new CustomSpanExporter(endpoint, timeout, compression, maxRetries);
    }

    private static class CustomSpanExporter implements SpanExporter {
        private final String endpoint;
        private final Integer timeout;
        private final Boolean compression;
        private final int maxRetries;

        CustomSpanExporter(String endpoint, Integer timeout, Boolean compression, int maxRetries) {
            this.endpoint = endpoint;
            this.timeout = timeout;
            this.compression = compression;
            this.maxRetries = maxRetries;
            System.out.println("CustomSpanExporter created with:");
            System.out.println("  endpoint: " + endpoint);
            System.out.println("  timeout: " + timeout);
            System.out.println("  compression: " + compression);
            System.out.println("  maxRetries: " + maxRetries);
        }

        @Override
        public CompletableResultCode export(Collection<SpanData> spans) {
            System.out.println("Exporting " + spans.size() + " spans to " + endpoint);
            // 实现导出逻辑
            return CompletableResultCode.ofSuccess();
        }

        @Override
        public CompletableResultCode flush() {
            return CompletableResultCode.ofSuccess();
        }

        @Override
        public CompletableResultCode shutdown() {
            System.out.println("Shutting down CustomSpanExporter");
            return CompletableResultCode.ofSuccess();
        }
    }
}
```

**注册 SPI**:

创建文件 `src/main/resources/META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.internal.ComponentProvider`:

```
com.example.CustomSpanExporterComponentProvider
```

**在 YAML 配置文件中使用**:

创建 `config.yaml`:

```yaml
file_format: "1.0"

# 定义自定义导出器
exporters:
  custom:  # ← 对应 getName() 返回的 "custom"
    endpoint: http://localhost:8080/traces
    timeout: 10000
    compression: true
    retry:
      max_attempts: 5
      initial_interval: 1000

# 在 TracerProvider 中使用
tracer_provider:
  processors:
    - batch:
        exporter: custom  # ← 引用上面定义的导出器

# 全局配置
resource:
  attributes:
    service.name: my-service
    deployment.environment: production
```

**Java 代码使用声明式配置**:

```java
import io.opentelemetry.sdk.extension.incubator.fileconfig.DeclarativeConfiguration;
import io.opentelemetry.sdk.extension.incubator.ExtendedOpenTelemetrySdk;

import java.io.FileInputStream;
import java.io.InputStream;

public class Application {
    public static void main(String[] args) throws Exception {
        // 方法 1: 从文件加载
        try (InputStream configStream = new FileInputStream("config.yaml")) {
            ExtendedOpenTelemetrySdk sdk =
                DeclarativeConfiguration.parseAndCreate(configStream);

            // 使用 SDK
            runApplication(sdk);

            // 关闭
            sdk.close();
        }

        // 方法 2: 通过环境变量指定配置文件
        // export OTEL_EXPERIMENTAL_CONFIG_FILE=/path/to/config.yaml
        // AutoConfiguredOpenTelemetrySdk.initialize() 会自动加载
    }
}
```

**YAML 配置的优势**:

1. **结构化配置**: 支持嵌套对象、数组等复杂结构
2. **可读性强**: 比环境变量更易于理解和维护
3. **集中管理**: 所有配置在一个文件中
4. **版本控制**: YAML 文件可以纳入 Git 等版本控制
5. **多环境支持**: 为不同环境维护不同的 YAML 文件

**YAML 配置示例（完整）**:

```yaml
file_format: "1.0"

# 资源属性
resource:
  attributes:
    service.name: my-microservice
    service.version: 1.2.3
    deployment.environment: production

# 传播器配置
propagator:
  composite:
    - tracecontext
    - baggage
    - b3

# Traces 配置
tracer_provider:
  # 采样器
  sampler:
    parent_based:
      root:
        trace_id_ratio_based:
          ratio: 0.1

  # Span 处理器
  processors:
    - batch:
        max_queue_size: 2048
        schedule_delay: 5000
        max_export_batch_size: 512
        exporter: otlp_grpc

# Metrics 配置
meter_provider:
  readers:
    - periodic:
        interval: 60000
        exporter: otlp_grpc
    - prometheus:
        host: localhost
        port: 9464

# 导出器配置
exporters:
  otlp_grpc:
    endpoint: http://otel-collector:4317
    headers:
      Authorization: Bearer secret-token
    timeout: 10000
    compression: gzip

  custom:
    endpoint: http://custom-backend:8080
    timeout: 15000
```

**最佳实践**:

1. ✅ **使用 DeclarativeConfigProperties 而不是直接解析 Map**
   - 提供类型安全的配置访问
   - 自动处理类型转换和默认值

2. ✅ **验证必需的配置参数**
   ```java
   String endpoint = config.getString("endpoint");
   if (endpoint == null) {
       throw new IllegalArgumentException("endpoint is required");
   }
   ```

3. ✅ **提供合理的默认值**
   ```java
   int timeout = config.getInt("timeout", 10000);
   boolean compression = config.getBoolean("compression", false);
   ```

4. ✅ **文档化 YAML 配置结构**
   - 在 Javadoc 中提供 YAML 示例
   - 说明所有支持的配置项

5. ✅ **处理嵌套配置**
   ```java
   DeclarativeConfigProperties retryConfig = config.getStructure("retry");
   if (retryConfig != null) {
       int maxRetries = retryConfig.getInt("max_attempts", 3);
   }
   ```

**注意事项**:

- ⚠️ **实验性功能**: ComponentProvider 是内部 API，可能随时变更
- ⚠️ **需要 incubator 依赖**: 必须添加 `opentelemetry-sdk-extension-incubator` 依赖
- ⚠️ **不适用于环境变量配置**: ComponentProvider 仅用于 YAML 文件，环境变量配置请使用 `ConfigurableXxxProvider`
- ⚠️ **向后兼容性**: 作为内部 API，不保证向后兼容

**典型用例**:

1. **复杂配置场景**: 当组件需要大量配置参数时
2. **嵌套结构**: 配置包含对象、数组等嵌套结构
3. **多环境部署**: 为不同环境维护不同的 YAML 文件
4. **团队协作**: YAML 文件更易于审查和维护

---

### AutoConfigureListener

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.internal.AutoConfigureListener`

**稳定性**: ⚠️ 内部 API（不稳定）

**目的**: 在自动配置完成后接收回调通知。

**接口定义**:
```java
public interface AutoConfigureListener {
    void afterAutoConfigure(OpenTelemetrySdk sdk);
}
```

**使用场景**:
- 配置完成后的验证
- 打印配置摘要
- 发送配置完成事件

---

### DefaultConfigProperties

**包路径**: `io.opentelemetry.sdk.autoconfigure.spi.internal.DefaultConfigProperties`

**稳定性**: ⚠️ 内部 API（不稳定）

**目的**: `ConfigProperties` 接口的默认实现。

**用途**: 供 `autoconfigure` 模块内部使用，不建议直接使用。

---

## 完整示例

以下示例展示如何结合多个 SPI 实现复杂的自定义需求。

### 示例 1：完整的生产环境配置

结合多个 SPI 实现生产就绪的配置：

```java
// 1. ResourceProvider: 添加部署信息
public class DeploymentResourceProvider implements ResourceProvider {
    @Override
    public Resource createResource(ConfigProperties config) {
        return Resource.create(Attributes.of(
            AttributeKey.stringKey("deployment.environment"), "production",
            AttributeKey.stringKey("deployment.region"), "us-west-2",
            AttributeKey.stringKey("service.version"), getBuildVersion()
        ));
    }

    @Override
    public int order() {
        return 100;
    }
}

// 2. ConfigurableSamplerProvider: 智能采样
public class SmartSamplerProvider implements ConfigurableSamplerProvider {
    @Override
    public Sampler createSampler(ConfigProperties config) {
        return new SmartSampler();
    }

    @Override
    public String getName() {
        return "smart";
    }
}

// 3. ConfigurableSpanExporterProvider: 多后端导出
public class MultiBackendExporterProvider implements ConfigurableSpanExporterProvider {
    @Override
    public SpanExporter createExporter(ConfigProperties config) {
        // 同时导出到 OTLP 和 S3
        SpanExporter otlp = OtlpGrpcSpanExporter.builder()
            .setEndpoint(config.getString("otel.exporter.otlp.endpoint"))
            .build();
        SpanExporter s3 = new S3SpanExporter(config);
        return SpanExporter.composite(otlp, s3);
    }

    @Override
    public String getName() {
        return "multi";
    }
}

// 4. AutoConfigurationCustomizerProvider: 全局定制
public class ProductionCustomizerProvider implements AutoConfigurationCustomizerProvider {
    @Override
    public void customize(AutoConfigurationCustomizer customizer) {
        customizer
            // 添加全局属性
            .addResourceCustomizer((resource, config) ->
                resource.toBuilder()
                    .put("service.namespace", "my-company")
                    .build())
            // 过滤内部 Span
            .addSpanProcessorCustomizer((processor, config) ->
                new FilteringSpanProcessor(processor));
    }

    @Override
    public int order() {
        return 100;
    }
}
```

**注册所有 SPI**:
```
META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.ResourceProvider
com.example.DeploymentResourceProvider

META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSamplerProvider
com.example.SmartSamplerProvider

META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.traces.ConfigurableSpanExporterProvider
com.example.MultiBackendExporterProvider

META-INF/services/io.opentelemetry.sdk.autoconfigure.spi.AutoConfigurationCustomizerProvider
com.example.ProductionCustomizerProvider
```

**配置**:
```bash
export OTEL_SERVICE_NAME=my-service
export OTEL_TRACES_SAMPLER=smart
export OTEL_TRACES_EXPORTER=multi
export OTEL_EXPORTER_OTLP_ENDPOINT=https://api.example.com:4317
```

---

## 最佳实践

### SPI 实现

1. **线程安全**: SPI 实现应该是无状态的或线程安全的
2. **快速初始化**: `create*()` 方法应该快速返回，避免耗时操作
3. **资源清理**: 导出器必须正确实现 `shutdown()` 方法
4. **错误处理**: 使用 `ConfigurationException` 报告配置错误
5. **文档化**: 清楚文档化所有配置属性和默认值

### 配置属性命名

遵循 OpenTelemetry 的命名约定：
- 使用小写字母和点分隔：`otel.exporter.myexporter.endpoint`
- 信号特定配置：`otel.traces.exporter`, `otel.metrics.exporter`
- 导出器特定配置：`otel.exporter.<name>.*`

### 顺序控制

- 使用有意义的顺序值：-100（最早）、0（默认）、50（中间）、100（最晚）
- 为未来扩展保留空间
- 文档化顺序依赖关系

---

## 故障排查

### SPI 未加载

**症状**: 自定义 SPI 实现未被发现

**检查清单**:
1. 确认 `META-INF/services/` 文件名正确（完整接口名）
2. 确认文件内容是实现类的完整类名
3. 确认文件使用 UTF-8 编码
4. 确认实现类在 classpath 中
5. 确认实现类有无参构造函数
6. 检查是否有类加载错误（查看日志）

**调试**:
```java
// 手动测试 SPI 加载
ServiceLoader<ConfigurableSpanExporterProvider> loader =
    ServiceLoader.load(ConfigurableSpanExporterProvider.class);

for (ConfigurableSpanExporterProvider provider : loader) {
    System.out.println("Loaded: " + provider.getName());
}
```

### 配置未生效

**症状**: 环境变量设置但未生效

**检查清单**:
1. 确认环境变量在应用启动前设置
2. 检查是否有系统属性覆盖（`-D` 优先级更高）
3. 确认属性名称正确（区分大小写）
4. 检查是否有 `AutoConfigurationCustomizer` 覆盖配置
5. 启用 debug 日志查看配置加载过程

### 顺序问题

**症状**: ResourceProvider 的属性被意外覆盖

**解决方案**:
- 检查所有 ResourceProvider 的 `order()` 值
- 确保后执行的 Provider 有更高的 `order()` 值
- 使用 `order()` 明确控制执行顺序

---

## 迁移指南

### 从 SdkTracerProviderConfigurer 迁移

`SdkTracerProviderConfigurer` 已废弃，迁移到 `AutoConfigurationCustomizerProvider`:

**旧代码**:
```java
public class MyConfigurer implements SdkTracerProviderConfigurer {
    @Override
    public void configure(SdkTracerProviderBuilder builder, ConfigProperties config) {
        builder.setSampler(Sampler.alwaysOn());
    }
}
```

**新代码**:
```java
public class MyCustomizerProvider implements AutoConfigurationCustomizerProvider {
    @Override
    public void customize(AutoConfigurationCustomizer customizer) {
        customizer.addTracerProviderCustomizer((builder, config) ->
            builder.setSampler(Sampler.alwaysOn()));
    }
}
```

---

## API 稳定性

### 稳定的公共 SPI

以下接口保证向后兼容：
- ✅ `AutoConfigurationCustomizer`
- ✅ `AutoConfigurationCustomizerProvider`
- ✅ `ConfigProperties`
- ✅ `ResourceProvider`
- ✅ `ConfigurablePropagatorProvider`
- ✅ `ConfigurableSpanExporterProvider`
- ✅ `ConfigurableSamplerProvider`
- ✅ `ConfigurableMetricExporterProvider`
- ✅ `ConfigurableLogRecordExporterProvider`
- ✅ `Ordered`
- ✅ `ConfigurationException`

### 内部 API

标记为 `@Internal` 的接口可能随时变更：
- ⚠️ `ConditionalResourceProvider`
- ⚠️ `ConfigurableMetricReaderProvider`
- ⚠️ `ComponentProvider`
- ⚠️ `AutoConfigureListener`
- ⚠️ `DefaultConfigProperties`

### 废弃 API

- ❌ `SdkTracerProviderConfigurer` - 使用 `AutoConfigurationCustomizerProvider` 替代

---

## 相关文档

- **OpenTelemetry 规范**: [https://opentelemetry.io/docs/specs/otel/](https://opentelemetry.io/docs/specs/otel/)
- **Java SDK 文档**: [https://opentelemetry.io/docs/instrumentation/java/](https://opentelemetry.io/docs/instrumentation/java/)
- **自动配置模块**: [../autoconfigure/README.zh-CN.md](../autoconfigure/README.zh-CN.md)
- **环境变量规范**: [https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/](https://opentelemetry.io/docs/specs/otel/configuration/sdk-environment-variables/)

---

## 许可证

本项目采用 Apache License 2.0 许可证。

---

**维护者**: OpenTelemetry Java SIG
**最后更新**: 2026-01-13
**文档版本**: 1.0.0

