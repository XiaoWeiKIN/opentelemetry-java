### 3.7 otel.animalsniffer-conventions.gradle.kts

**文件路径**: `src/main/kotlin/otel.animalsniffer-conventions.gradle.kts:1`

**核心功能**: 检查代码是否使用了目标 Java 版本不支持的 API，特别是 Android 平台的 API 兼容性检查。

#### 什么是 Animal Sniffer？

Animal Sniffer 是一个 Gradle 插件，用于确保代码只使用指定 Java 版本或 Android API 级别支持的类和方法。它通过比对"签名文件"（signature file）来检查代码中的 API 调用是否合法。

**核心概念**:
```
┌─────────────────────────────────────────────────────┐
│ 你的源代码                                           │
│ - 使用 java.time.Duration (Java 8+)                │
│ - 使用 java.util.Optional (Java 8+)                │
│ - 使用 java.nio.file.Files (Java 7+)               │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓ AnimalSniffer 检查
┌─────────────────────────────────────────────────────┐
│ 签名文件 (android.signature)                        │
│ - Android API 23 (6.0) 支持的类和方法列表           │
│ - java.time.Duration ✗ (API 26+)                   │
│ - java.util.Optional ✗ (API 24+)                   │
│ - java.nio.file.Files ✗ (API 26+)                  │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓ 报告 API 不兼容
┌─────────────────────────────────────────────────────┐
│ 构建失败 / 警告                                      │
│ error: Undefined reference: java.time.Duration      │
│ error: Undefined reference: java.util.Optional      │
└─────────────────────────────────────────────────────┘
```

#### 为什么需要 Animal Sniffer？

**问题场景**:
```java
// 开发者使用 Java 21 编译，目标是 Android API 23
// ❌ 这段代码编译成功，但在 Android API 23 上运行时崩溃！
public class MyService {
    public void process() {
        Duration duration = Duration.ofSeconds(10);  // ⚠️ API 26+
        Optional<String> opt = Optional.of("foo");   // ⚠️ API 24+
    }
}
```

**编译器无法检测到这个问题的原因**:
- Java 编译器只检查语法和类型，不检查运行时 API 可用性
- `-release 8` 参数保证生成 Java 8 字节码，但无法保证 Android 兼容性
- Android API 和 Java API 不完全一致（即使是相同的 Java 版本）

**Animal Sniffer 的价值**:
```
✓ 编译时捕获 API 兼容性问题（而非运行时崩溃）
✓ 确保库可以在旧版本 Android 上运行
✓ 自动化检查，无需手动审查代码
✓ 支持自定义签名文件（适配特定平台需求）
```

#### 完整配置详解

**otel.animalsniffer-conventions.gradle.kts 源码**:
```kotlin
import ru.vyarus.gradle.plugin.animalsniffer.AnimalSniffer

plugins {
  `java-library`
  id("ru.vyarus.animalsniffer")
}

dependencies {
  signature(project(path = ":animal-sniffer-signature", configuration = "generatedSignature"))
}

animalsniffer {
  sourceSets = listOf(java.sourceSets.main.get())
}

tasks.withType<AnimalSniffer> {
  // always having declared output makes this task properly participate in tasks up-to-date checks
  reports.text.required.set(true)
}
```

##### 1. 依赖配置

```kotlin
dependencies {
  signature(project(path = ":animal-sniffer-signature", configuration = "generatedSignature"))
}
```

**解析**:
- `signature`: AnimalSniffer 插件提供的专用配置，用于指定签名文件
- `:animal-sniffer-signature`: 项目内的自定义子模块，负责生成签名文件
- `configuration = "generatedSignature"`: 使用该模块的 `generatedSignature` 配置（消费型配置）

##### 2. 源码集配置

```kotlin
animalsniffer {
  sourceSets = listOf(java.sourceSets.main.get())
}
```

**仅检查主源码集**:
- 主源码 (`src/main/java`) - ✓ 检查
- 测试代码 (`src/test/java`) - ✗ 不检查
- JMH 基准测试 - ✗ 不检查

**为什么不检查测试代码？**
- 测试代码不会打包到发布的库中
- 测试可以使用更新的 API 和工具
- 减少构建时间和复杂度

##### 3. 报告配置

```kotlin
tasks.withType<AnimalSniffer> {
  reports.text.required.set(true)
}
```

**报告选项**:
- `reports.text.required.set(true)`: 生成文本格式报告
- **报告路径**: `build/reports/animalsniffer/main.txt`

**报告示例**:
```
Animal Sniffer Report
=====================

Undefined reference: java.time.Duration
  at io.opentelemetry.sdk.MyClass:45

Undefined reference: java.util.Optional
  at io.opentelemetry.exporter.Exporter:67

Total violations: 2
```

#### 签名文件的生成（`:animal-sniffer-signature` 模块）

OpenTelemetry 项目使用自定义签名文件，专门针对 Android API 23（Android 6.0）进行兼容性检查。

**animal-sniffer-signature/build.gradle.kts**:
```kotlin
import ru.vyarus.gradle.plugin.animalsniffer.signature.BuildSignatureTask

plugins {
  id("otel.java-conventions")
  id("ru.vyarus.animalsniffer")
}

// 1. 定义配置
val signatureJar = configurations.create("signatureJar")
val signatureJarClasspath = configurations.create("signatureJarClasspath") {
  extendsFrom(signatureJar)
}
val generatedSignature = configurations.create("generatedSignature") {
  isCanBeConsumed = true  // 供其他模块消费
}

// 2. 声明基础签名和额外的 JAR
dependencies {
  // 基础签名: Android API 23
  signature("com.toasttab.android:gummy-bears-api-23:0.12.0@signature")

  // 扩展库: Android Desugar 库（提供部分 Java 8+ API 的向后移植）
  signatureJar("com.android.tools:desugar_jdk_libs")
}

// 3. 构建签名任务
val signatureSimpleName = "android.signature"
val signatureBuilderTask = tasks.register("buildSignature", BuildSignatureTask::class.java) {
  files(signatureJarClasspath)      // 添加额外的 JAR 到签名
  signatures(configurations.signature)  // 继承基础签名
  outputName = signatureSimpleName
}

// 4. 导出生成的签名文件
artifacts {
  add("generatedSignature", File(signatureBuilderTask.outputs.files.singleFile, signatureSimpleName)) {
    builtBy(signatureBuilderTask)
  }
}
```

##### 签名生成流程

```
┌─────────────────────────────────────────────────────┐
│ 1. 基础签名                                          │
│    gummy-bears-api-23:0.12.0@signature              │
│    - Android API 23 标准 API                        │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────────────┐
│ 2. 扩展库                                            │
│    desugar_jdk_libs (Android Desugar)               │
│    - java.time.* (向后移植到 API 21+)               │
│    - java.util.stream.* (向后移植)                  │
│    - java.util.Optional (向后移植)                  │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓ BuildSignatureTask 合并
┌─────────────────────────────────────────────────────┐
│ 3. 生成 android.signature                           │
│    包含:                                             │
│    - Android API 23 标准 API                        │
│    - Desugar 库向后移植的 API                       │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓ 导出为 generatedSignature 配置
┌─────────────────────────────────────────────────────┐
│ 4. 被其他模块消费                                    │
│    dependencies {                                   │
│      signature(project(":animal-sniffer-signature"))│
│    }                                                │
└─────────────────────────────────────────────────────┘
```

##### 什么是 Android Desugar？

**Android Desugar** 是 Google 提供的库，允许在旧版本 Android 上使用部分 Java 8+ API。

**支持的 API**（部分）:
```java
// ✓ 在 Android API 21+ 可用（通过 Desugar）
java.time.Duration
java.time.Instant
java.util.Optional
java.util.stream.Stream
java.util.function.Function

// ✗ 仍然不可用（需要 API 26+）
java.nio.file.Files
java.nio.file.Path
```

**Desugar 的工作原理**:
```
1. 编译时: 将 java.time.* 调用重写为 j$.time.* (向后移植的实现)
2. 运行时: j$.time.* 类打包在 desugar_jdk_libs.jar 中
3. 结果: 代码可以在 Android API 21+ 运行
```

#### 实际使用示例

##### 示例 1: 检测 API 不兼容

**源代码**:
```java
package io.opentelemetry.sdk.metrics;

import java.nio.file.Files;  // ⚠️ Android API 26+
import java.nio.file.Paths;

public class MetricsExporter {
    public void export() {
        // 使用了 Android API 23 不支持的 API
        String content = Files.readString(Paths.get("/tmp/data.txt"));
    }
}
```

**运行检查**:
```bash
./gradlew :sdk:metrics:animalsnifferMain
```

**输出（构建失败）**:
```
> Task :sdk:metrics:animalsnifferMain FAILED

Animal Sniffer violations:
  Undefined reference: java.nio.file.Files
    at io.opentelemetry.sdk.metrics.MetricsExporter:8
  Undefined reference: java.nio.file.Paths
    at io.opentelemetry.sdk.metrics.MetricsExporter:8

BUILD FAILED
```

##### 示例 2: 使用兼容 API

**修复方案 1: 使用 Desugar 支持的 API**
```java
package io.opentelemetry.sdk.metrics;

import java.time.Duration;      // ✓ Desugar 支持（API 21+）
import java.util.Optional;      // ✓ Desugar 支持（API 21+）

public class MetricsExporter {
    private Duration timeout = Duration.ofSeconds(10);

    public Optional<String> getData() {
        return Optional.of("data");
    }
}
```

**修复方案 2: 使用 @IgnoreJRERequirement 注解抑制检查**
```java
import org.codehaus.mojo.animal_sniffer.IgnoreJRERequirement;
import java.nio.file.Files;

public class MetricsExporter {
    @IgnoreJRERequirement  // 显式标记：我知道这个 API 不兼容
    public void export() {
        if (isAndroid()) {
            // 在 Android 上使用替代实现
        } else {
            // 在 JVM 上使用 Files API
            String content = Files.readString(...);
        }
    }
}
```

**修复方案 3: 完全避免使用不兼容 API**
```java
import java.io.BufferedReader;
import java.io.FileReader;

public class MetricsExporter {
    public void export() {
        // 使用 Java 7 兼容的 API
        try (BufferedReader reader = new BufferedReader(new FileReader("/tmp/data.txt"))) {
            String content = reader.readLine();
        }
    }
}
```

##### 示例 3: 在项目中启用 AnimalSniffer

**子模块 build.gradle.kts**:
```kotlin
plugins {
    id("otel.java-conventions")
    id("otel.animalsniffer-conventions")  // 启用 AnimalSniffer 检查
}

dependencies {
    api(project(":api"))
    implementation("com.google.guava:guava")
}
```

**构建时自动检查**:
```bash
# 编译时自动运行 AnimalSniffer
./gradlew :exporters:otlp:build

# 手动运行 AnimalSniffer
./gradlew :exporters:otlp:animalsnifferMain

# 查看生成的报告
cat exporters/otlp/build/reports/animalsniffer/main.txt
```

#### AnimalSniffer 工作流程

```
┌─────────────────────────────────────────────────────┐
│ 1. 编译 Java 源码 → class 文件                      │
│    javac -release 8 Main.java                       │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────────────┐
│ 2. AnimalSniffer 扫描 class 文件                    │
│    - 提取所有类、方法、字段引用                       │
│    - 例: java/time/Duration.ofSeconds(J)           │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────────────┐
│ 3. 对比签名文件 (android.signature)                 │
│    - 检查 java.time.Duration 是否存在               │
│    - 检查 ofSeconds 方法是否存在                    │
└─────────────────┬───────────────────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────────────────┐
│ 4. 报告不兼容的 API 引用                             │
│    - Undefined reference: java.nio.file.Files       │
│    - 构建失败或警告                                  │
└─────────────────────────────────────────────────────┘
```

#### 调试和故障排查

##### 1. 查看签名文件内容

```bash
# 打印签名文件包含的 API
./gradlew :animal-sniffer-signature:printSignature

# 输出示例:
# java.lang.Object
# java.lang.String
# java.util.List
# java.time.Duration (from desugar)
# ...
```

##### 2. 跳过 AnimalSniffer 检查

```bash
# 跳过所有 AnimalSniffer 任务
./gradlew build -x animalsnifferMain

# 仅跳过特定模块
./gradlew :sdk:metrics:build -x animalsnifferMain
```

##### 3. 查看详细日志

```bash
./gradlew animalsnifferMain --info
```

##### 4. 常见错误和解决方案

**错误 1: Undefined reference to internal class**
```
Undefined reference: sun.misc.Unsafe
```

**解决**: 内部 API 通常不在签名文件中，使用 `@IgnoreJRERequirement` 注解抑制。

**错误 2: 签名文件找不到**
```
Could not resolve project :animal-sniffer-signature
```

**解决**: 确保根项目 `settings.gradle.kts` 中包含：
```kotlin
include(":animal-sniffer-signature")
```

**错误 3: 检查失败但 API 实际上兼容**
```
Undefined reference: java.util.concurrent.CompletableFuture
```

**解决**: 如果该 API 通过 Desugar 支持，需要在 `:animal-sniffer-signature` 模块中添加相应的依赖。

#### 与其他工具的对比

| 工具 | 用途 | 检查对象 | 运行时机 |
|------|------|----------|----------|
| **AnimalSniffer** | API 兼容性 | class 文件的 API 引用 | 编译后 |
| **ErrorProne** | 静态分析 | 源码的潜在 bug | 编译时 |
| **Checkstyle** | 代码风格 | 源码格式 | 构建时 |
| **JApiCmp** | API 向后兼容 | JAR 之间的 API 差异 | 发布前 |

**协同工作**:
```
编译源码 → ErrorProne 检查 → 生成 class 文件 → AnimalSniffer 检查 → 构建成功
    ↓                                                          ↓
Checkstyle 检查                                            JApiCmp 检查
```

#### 性能和缓存

**构建缓存**:
```kotlin
tasks.withType<AnimalSniffer> {
  // 启用增量构建和缓存
  reports.text.required.set(true)  // 声明输出，支持 up-to-date 检查
}
```

**跳过条件**:
- ✓ 源码未改变
- ✓ 签名文件未改变
- ✓ 插件版本未改变

**性能优化**:
```bash
# 仅检查变更的模块
./gradlew :sdk:metrics:animalsnifferMain --parallel

# 使用构建缓存
./gradlew build --build-cache
```

#### 配置选项参考

**完整配置示例**:
```kotlin
animalsniffer {
  // 要检查的源码集
  sourceSets = listOf(sourceSets.main.get(), sourceSets.api.get())

  // 忽略缺失的类（默认 false）
  // ignoreMissingClasses = true

  // 签名版本
  // toolVersion = "1.23"
}

tasks.withType<AnimalSniffer> {
  // 报告配置
  reports {
    text.required.set(true)
    // html.required.set(true)
  }

  // 自定义输出目录
  // reports.text.outputLocation.set(file("${buildDir}/reports/animalsniffer-custom.txt"))
}
```

#### 最佳实践

**1. 明确目标平台**
```kotlin
// 为不同平台使用不同的签名
dependencies {
  if (project.hasProperty("targetAndroid")) {
    signature(project(":animal-sniffer-signature"))  // Android API 23
  } else {
    signature("net.sf.androidscents.signature:android-api-level-21:5.0.1_r2@signature")
  }
}
```

**2. 在 CI 中强制检查**
```yaml
# .github/workflows/build.yml
- name: Build with AnimalSniffer
  run: ./gradlew build  # AnimalSniffer 自动运行
```

**3. 文档化不兼容的 API**
```java
/**
 * Uses java.nio.file API which requires Android API 26+.
 * On older Android versions, this method will throw UnsupportedOperationException.
 *
 * @throws UnsupportedOperationException on Android API < 26
 */
@IgnoreJRERequirement
public void processFile(Path path) {
    // ...
}
```

**4. 定期更新签名文件**
```bash
# 升级到 Android API 24
dependencies {
  signature("com.toasttab.android:gummy-bears-api-24:0.12.0@signature")
}
```

#### 总结

**AnimalSniffer 的核心价值**:
- ✅ **编译时保证**: 在开发阶段发现 API 兼容性问题
- ✅ **自动化**: 无需手动审查每个 API 调用
- ✅ **灵活**: 支持自定义签名文件和抑制规则
- ✅ **轻量**: 检查快速，对构建性能影响小

**适用场景**:
- 开发 Android 库（需要兼容旧版本）
- 开发跨平台 Java 库（JVM + Android）
- 确保代码在特定 Java 版本上运行

**OpenTelemetry 使用 AnimalSniffer 的原因**:
- 确保 SDK 可以在 Android API 23+ 上运行
- 利用 Desugar 库向后移植 Java 8+ API
- 在编译时捕获 API 兼容性问题，而非用户运行时崩溃

---
