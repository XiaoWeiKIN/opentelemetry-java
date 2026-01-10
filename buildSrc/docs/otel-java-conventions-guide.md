# otel.java-conventions.gradle.kts 逐行详解

**文件信息**:
- **源文件路径**: `buildSrc/src/main/kotlin/otel.java-conventions.gradle.kts`
- **文件行数**: 299 行
- **作用**: 为所有 Java 模块提供统一的构建配置
- **文档版本**: 1.0.0
- **最后更新**: 2026-01-10

## 概述

本文档详细解析 OpenTelemetry Java 项目中最核心的约定插件 `otel.java-conventions.gradle.kts`。这个插件为项目的 50+ 个 Java 模块提供了统一的构建基础，包括：

- Java 工具链配置（Java 21 编译，Java 8+ 运行）
- 代码质量工具集成（Checkstyle、ErrorProne、Spotless）
- 测试框架配置（JUnit 5、Mockito）
- 依赖管理策略
- 安全漏洞扫描
- 可重现构建
- CI/CD 优化

---

## 目录

- [1. 导入声明（第 1-3 行）](#1-导入声明第-1-3-行)
- [2. 插件声明（第 5-16 行）](#2-插件声明第-5-16-行)
- [3. 扩展配置（第 18 行）](#3-扩展配置第-18-行)
- [4. 归档命名规范（第 20-26 行）](#4-归档命名规范第-20-26-行)
- [5. 可重现构建配置（第 28-33 行）](#5-可重现构建配置第-28-33-行)
- [6. Java 工具链配置（第 35-42 行）](#6-java-工具链配置第-35-42-行)
- [7. Checkstyle 配置（第 44-49 行）](#7-checkstyle-配置第-44-49-行)
- [8. OWASP 依赖安全检查（第 51-71 行）](#8-owasp-依赖安全检查第-51-71-行)
- [9. 测试 Java 版本配置（第 73 行）](#9-测试-java-版本配置第-73-行)
- [10. Java 编译配置（第 75-107 行）](#10-java-编译配置第-75-107-行)
- [11. 测试配置（第 109-129 行）](#11-测试配置第-109-129-行)
- [12. Javadoc 配置（第 131-167 行）](#12-javadoc-配置第-131-167-行)
- [13. JAR 清单配置（第 145-157 行）](#13-jar-清单配置第-145-157-行)
- [14. 多 Java 版本测试（第 170-181 行）](#14-多-java-版本测试第-170-181-行)
- [15. 版本资源生成（第 184-205 行）](#15-版本资源生成第-184-205-行)
- [16. 依赖管理配置（第 207-251 行）](#16-依赖管理配置第-207-251-行)
- [17. 测试套件配置（第 253-298 行）](#17-测试套件配置第-253-298-行)
- [18. 总结](#18-总结)

---

## 1. 导入声明（第 1-3 行）

```kotlin
import io.opentelemetry.gradle.OtelJavaExtension
import org.gradle.api.JavaVersion
import org.gradle.api.tasks.testing.logging.TestExceptionFormat
```

**解析**:
- `OtelJavaExtension`: 自定义扩展类，提供 `moduleName` 和 `minJavaVersionSupported` 配置
- `JavaVersion`: Gradle 标准类，用于 Java 版本比较和配置
- `TestExceptionFormat`: 测试日志格式枚举，控制异常输出格式

---

## 2. 插件声明（第 5-16 行）

```kotlin
plugins {
  `java-library`
  checkstyle
  eclipse
  idea
  id("otel.errorprone-conventions")
  id("otel.jacoco-conventions")
  id("otel.spotless-conventions")
  id("org.owasp.dependencycheck")
}
```

### 核心插件

**`java-library`**:
- 提供 Java 库项目的基础功能
- 支持 `api` 和 `implementation` 依赖配置
- 与 `java` 插件的区别：
```kotlin
// java-library 提供
dependencies {
    api("com.google.guava:guava")           // 传递给消费者
    implementation("org.slf4j:slf4j-api")   // 不传递
}
```

**`checkstyle`**: Google Java Style 代码风格检查（版本 13.0.0）

**`eclipse` 和 `idea`**: 生成 IDE 项目文件

### 自定义约定插件

- `otel.errorprone-conventions`: ErrorProne 静态分析 + NullAway 空指针检查
- `otel.jacoco-conventions`: JaCoCo 代码覆盖率
- `otel.spotless-conventions`: 代码格式化 + Apache 2.0 许可证头

### 安全插件

**`org.owasp.dependencycheck`**:
- 扫描依赖的已知安全漏洞（基于 NVD 数据库）
- CVSS >= 7.0 的漏洞会导致构建失败

---

## 3. 扩展配置（第 18 行）

```kotlin
val otelJava = extensions.create<OtelJavaExtension>("otelJava")
```

创建名为 `otelJava` 的扩展对象，子项目可通过 DSL 配置：

```kotlin
otelJava {
    moduleName.set("io.opentelemetry.api")
    minJavaVersionSupported.set(JavaVersion.VERSION_1_8)
}
```

---

## 4. 归档命名规范（第 20-26 行）

```kotlin
base {
  if (!archivesName.get().startsWith("opentelemetry-")) {
    archivesName.set("opentelemetry-$name")
  }
}
```

**解析**:
- 统一归档命名：`opentelemetry-<模块名>`
- 避免重复添加前缀（父项目可能已设置）
- 生成的 JAR 文件名示例：`opentelemetry-api-1.35.0.jar`

---

## 5. 可重现构建配置（第 28-33 行）

```kotlin
// normalize timestamps and file ordering in jars, making the outputs reproducible
// see open-telemetry/opentelemetry-java#4488
tasks.withType<AbstractArchiveTask>().configureEach {
  isPreserveFileTimestamps = false
  isReproducibleFileOrder = true
}
```

### `isPreserveFileTimestamps = false`
**作用**: 禁用 ZIP/JAR 中的文件时间戳，所有文件时间戳设为固定值

**为什么重要**:
```bash
# 没有此配置（每次构建哈希不同）
$ sha256sum build/libs/api-1.0.0.jar
a1b2c3d4... api-1.0.0.jar  # 第一次构建

$ sha256sum build/libs/api-1.0.0.jar
e5f6g7h8... api-1.0.0.jar  # 第二次构建（内容相同但哈希不同！）

# 有此配置（可重现构建）
$ sha256sum build/libs/api-1.0.0.jar
a1b2c3d4... api-1.0.0.jar  # 每次构建的哈希都相同
```

### `isReproducibleFileOrder = true`
**作用**: 按字典序排序 ZIP/JAR 中的文件条目，避免文件系统遍历顺序的随机性

**收益**:
- ✅ 构建缓存更有效（字节级相同）
- ✅ 安全性更好（可验证构建完整性）
- ✅ 支持分布式缓存（Gradle Build Cache、Develocity）

---

## 6. Java 工具链配置（第 35-42 行）

```kotlin
java {
  toolchain {
    languageVersion.set(JavaLanguageVersion.of(21))
  }
  withJavadocJar()
  withSourcesJar()
}
```

### Java 工具链

**关键概念**: 工具链 vs 目标版本
```
┌─────────────────────────────────────────┐
│ Java 21 工具链 (编译器)                  │
│   ↓ 编译                                 │
│ Java 8 字节码 (release = 8)             │
│   ↓ 运行                                 │
│ Java 8+ 运行时环境                      │
└─────────────────────────────────────────┘
```

**优势**:
- ✅ 使用现代编译器（更快、更好的优化）
- ✅ 向后兼容旧版本 Java（Java 8+）
- ✅ 避免 bootclasspath 配置的复杂性

### 附加 JAR 生成

```kotlin
withJavadocJar()   // 生成 *-javadoc.jar
withSourcesJar()   // 生成 *-sources.jar
```

**生成的文件**:
```
build/libs/
├── opentelemetry-api-1.35.0.jar         # 主 JAR
├── opentelemetry-api-1.35.0-javadoc.jar # API 文档
└── opentelemetry-api-1.35.0-sources.jar # 源码
```

---

## 7. Checkstyle 配置（第 44-49 行）

```kotlin
checkstyle {
  configDirectory.set(file("$rootDir/buildscripts/"))
  toolVersion = "13.0.0"
  isIgnoreFailures = false
  configProperties["rootDir"] = rootDir
}
```

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `configDirectory` | `buildscripts/` | 配置文件目录 |
| `toolVersion` | `13.0.0` | Checkstyle 版本 |
| `isIgnoreFailures` | `false` | 风格违规**立即失败**构建 |
| `configProperties["rootDir"]` | 项目根目录 | 传递给 checkstyle.xml 使用 |

**报告位置**: `build/reports/checkstyle/main.html`

---

## 8. OWASP 依赖安全检查（第 51-71 行）

```kotlin
dependencyCheck {
  skipConfigurations = mutableListOf(
    "errorprone", "checkstyle", "annotationProcessor",
    "java9AnnotationProcessor", "moduleAnnotationProcessor",
    "testAnnotationProcessor", "testJpmsAnnotationProcessor",
    "animalsniffer", "spotless996155815", "js2p",
    "jmhAnnotationProcessor", "jmhBasedTestAnnotationProcessor",
    "jmhCompileClasspath", "jmhRuntimeClasspath", "jmhRuntimeOnly"
  )
  failBuildOnCVSS = 7.0f
  analyzers.assemblyEnabled = false
  nvd.apiKey = System.getenv("NVD_API_KEY")
}
```

### 跳过的配置

| 配置类型 | 示例 | 跳过原因 |
|---------|------|----------|
| 构建工具依赖 | `errorprone`, `checkstyle` | 仅编译时使用，不打包到产物 |
| 注解处理器 | `annotationProcessor` | 仅编译时处理注解 |
| 测试依赖 | `jmh*` | 不影响生产代码 |

**性能优化**: 减少 50%+ 的扫描时间

### CVSS 阈值

```kotlin
failBuildOnCVSS = 7.0f  // fail on high or critical CVE
```

**CVSS 评分体系**:
```
0.0 - 3.9  ➜ Low      (低危)      ✓ 构建继续
4.0 - 6.9  ➜ Medium   (中危)      ✓ 构建继续
7.0 - 8.9  ➜ High     (高危)      ✗ 构建失败 ⚠️
9.0 - 10.0 ➜ Critical (严重)      ✗ 构建失败 ⚠️
```

### NVD API 密钥

```kotlin
nvd.apiKey = System.getenv("NVD_API_KEY")
```

**获取密钥**: [https://nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key)

**配置方式**:
```bash
# 环境变量
export NVD_API_KEY=your-api-key-here

# CI 环境变量（推荐）
# GitHub Actions: Settings → Secrets → NVD_API_KEY
```

---

## 9. 测试 Java 版本配置（第 73 行）

```kotlin
val testJavaVersion = gradle.startParameter.projectProperties.get("testJavaVersion")?.let(JavaVersion::toVersion)
```

从 Gradle 启动参数读取 `testJavaVersion` 属性，用于动态测试版本切换。

**使用示例**:
```bash
# 使用 Java 17 运行测试
./gradlew test -PtestJavaVersion=17

# 使用 Java 21 运行测试
./gradlew test -PtestJavaVersion=21
```

---

## 10. Java 编译配置（第 75-107 行）

```kotlin
tasks {
  withType<JavaCompile>().configureEach {
    with(options) {
      release.set(otelJava.minJavaVersionSupported.map { it.majorVersion.toInt() })

      if (name != "jmhCompileGeneratedClasses") {
        compilerArgs.addAll(
          listOf(
            "-Xlint:all",
            "-Xlint:-try",
            "-Xlint:-processing",
            "-Xlint:-options",
            "-Xlint:-serial",
            "-Xlint:-this-escape",
            "-Werror",
          ),
        )
      }

      encoding = "UTF-8"

      if (name.contains("Test")) {
        compilerArgs.add("-Xlint:-serial")
      }
    }
  }
}
```

### Release 参数

```kotlin
release.set(otelJava.minJavaVersionSupported.map { it.majorVersion.toInt() })
```

**默认值**: `8`（Java 8）

**为什么使用 `release` 而非 `sourceCompatibility`**:
```kotlin
// ❌ 不推荐（可能使用 Java 21 API）
sourceCompatibility = "8"
targetCompatibility = "8"

// ✅ 推荐（严格限制只能使用 Java 8 API）
release = 8
```

### 编译器警告配置

| 警告类型 | 含义 | 为何禁用 |
|---------|------|----------|
| `try` | try-with-resources 未引用资源 | 允许 `try (resource) {}` 用于确保关闭 |
| `processing` | 注解处理器相关 | Bazel 项目建议（避免误报）|
| `options` | 编译选项不兼容 | 现代 JDK 编译旧版本目标时触发 |
| `serial` | 缺少 serialVersionUID | 大多数类不需要序列化 |
| `this-escape` | 构造函数中 this 逃逸 | Java 21 新增，过于严格 |

**`-Werror`**: 将所有警告视为错误，确保代码质量

---

## 11. 测试配置（第 109-129 行）

```kotlin
withType<Test>().configureEach {
  useJUnitPlatform()

  val defaultMaxRetries = if (System.getenv().containsKey("CI")) 2 else 0
  val maxTestRetries = gradle.startParameter.projectProperties["maxTestRetries"]?.toInt() ?: defaultMaxRetries

  develocity.testRetry {
    maxRetries.set(maxTestRetries);
  }

  testLogging {
    exceptionFormat = TestExceptionFormat.FULL
    showExceptions = true
    showCauses = true
    showStackTraces = true
    showStandardStreams = true
  }
  maxHeapSize = "1500m"
}
```

### JUnit Platform

```kotlin
useJUnitPlatform()
```

启用 JUnit 5（Jupiter）测试引擎，支持 `@Test`、`@ParameterizedTest`、`@RepeatedTest` 等。

### 测试重试机制

```kotlin
val defaultMaxRetries = if (System.getenv().containsKey("CI")) 2 else 0
```

**逻辑**:
- CI 环境：自动重试 2 次（处理不稳定的测试）
- 本地开发：不重试（快速失败）

**使用示例**:
```bash
# CI 环境（自动）
./gradlew test  # 自动重试 2 次

# 本地强制重试
./gradlew test -PmaxTestRetries=3

# 禁用重试（即使在 CI）
./gradlew test -PmaxTestRetries=0
```

### 测试日志配置

| 配置项 | 值 | 效果 |
|--------|-----|------|
| `exceptionFormat` | `FULL` | 显示完整异常信息（包括抑制的异常） |
| `showExceptions` | `true` | 显示异常消息 |
| `showCauses` | `true` | 显示异常链（Caused by）|
| `showStackTraces` | `true` | 显示完整堆栈跟踪 |
| `showStandardStreams` | `true` | 显示 `System.out` 和 `System.err` |

### 测试内存配置

```kotlin
maxHeapSize = "1500m"
```

每个测试 JVM 最大堆内存：1.5 GB，防止测试 OOM。

---

## 12. Javadoc 配置（第 131-167 行）

```kotlin
withType<Javadoc>().configureEach {
  exclude("io/opentelemetry/**/internal/**")

  with(options as StandardJavadocDocletOptions) {
    source = "8"
    encoding = "UTF-8"
    docEncoding = "UTF-8"
    breakIterator(true)
    addBooleanOption("html5", true)
    addBooleanOption("Xdoclint:all,-missing", true)
  }
}

afterEvaluate {
  withType<Javadoc>().configureEach {
    with(options as StandardJavadocDocletOptions) {
      val title = "${project.description}"
      docTitle = title
      windowTitle = title
    }
  }
}
```

### 排除内部包

```kotlin
exclude("io/opentelemetry/**/internal/**")
```

排除所有 `internal` 包及其子包（`**` 匹配任意级别的目录）。

**匹配示例**:
```
✓ 排除: io/opentelemetry/api/internal/Utils.java
✓ 排除: io/opentelemetry/sdk/internal/metrics/Helper.java
✗ 保留: io/opentelemetry/api/trace/Span.java
```

### 标准 Javadoc 选项

| 选项 | 值 | 说明 |
|------|-----|------|
| `source` | `8` | Java 8 语法（即使用 Java 21 编译）|
| `encoding` | `UTF-8` | 读取源文件的编码 |
| `docEncoding` | `UTF-8` | 生成 HTML 的编码 |
| `breakIterator` | `true` | 使用 `java.text.BreakIterator` 检测句子边界 |
| `html5` | `true` | 生成 HTML5（而非 HTML4）|
| `Xdoclint:all,-missing` | `true` | 检查所有文档问题，但忽略缺失的文档 |

**为什么忽略 `missing`**: 不是所有方法都需要文档（如简单的 getter），避免过于严格的要求。

---

## 13. JAR 清单配置（第 145-157 行）

```kotlin
withType<Jar>().configureEach {
  inputs.property("moduleName", otelJava.moduleName)

  manifest {
    attributes(
      "Automatic-Module-Name" to otelJava.moduleName,
      "Built-By" to System.getProperty("user.name"),
      "Built-JDK" to System.getProperty("java.version"),
      "Implementation-Title" to project.name,
      "Implementation-Version" to project.version,
    )
  }
}
```

### 生成的 MANIFEST.MF 示例

```manifest
Manifest-Version: 1.0
Automatic-Module-Name: io.opentelemetry.api
Built-By: jdoe
Built-JDK: 21.0.1
Implementation-Title: opentelemetry-api
Implementation-Version: 1.35.0
```

### Automatic-Module-Name 详解

**作用**:
- 指定未模块化的 JAR 在 Java 9+ 模块系统中的模块名
- 避免自动生成的名称（基于 JAR 文件名，可能不稳定）

**示例**:
```java
// 模块声明中引用
module my.app {
    requires io.opentelemetry.api;  // 使用 Automatic-Module-Name
}
```

**最佳实践**:
- 使用反向域名（如 `io.opentelemetry.api`）
- 与包名一致
- 为未来的模块化做准备

---

## 14. 多 Java 版本测试（第 170-181 行）

```kotlin
afterEvaluate {
  tasks.withType<Test>().configureEach {
    if (testJavaVersion != null) {
      javaLauncher.set(
        javaToolchains.launcherFor {
          languageVersion.set(JavaLanguageVersion.of(testJavaVersion.majorVersion))
        }
      )
      isEnabled = isEnabled && testJavaVersion >= otelJava.minJavaVersionSupported.get()
    }
  }
}
```

### 动态 Java 版本切换

如果指定了 `-PtestJavaVersion=17`，使用 Java 17 运行测试。Gradle 自动下载或发现指定版本的 JDK。

**工作流程**:
```
1. 用 Java 21 编译 ➜ class 文件（target = 8）
2. 用 Java 17 运行测试 ➜ 验证 Java 17 兼容性
```

### 最低版本检查

```kotlin
isEnabled = isEnabled && testJavaVersion >= otelJava.minJavaVersionSupported.get()
```

如果 `testJavaVersion < minJavaVersionSupported`，禁用测试任务（跳过）。

### CI 矩阵测试示例

**GitHub Actions**:
```yaml
strategy:
  matrix:
    java: [8, 11, 17, 21]

steps:
  - name: Test with Java ${{ matrix.java }}
    run: ./gradlew test -PtestJavaVersion=${{ matrix.java }}
```

---

## 15. 版本资源生成（第 184-205 行）

```kotlin
plugins.withId("otel.publish-conventions") {
  tasks {
    register("generateVersionResource") {
      val moduleName = otelJava.moduleName
      val propertiesDir = moduleName.map {
        File(layout.buildDirectory.asFile.get(),
             "generated/properties/${it.replace('.', '/')}")
      }
      val versionProperty = project.version.toString()

      inputs.property("project.version", versionProperty)
      outputs.dir(propertiesDir)

      doLast {
        File(propertiesDir.get(), "version.properties").writeText("sdk.version=${versionProperty}")
      }
    }
  }

  sourceSets {
    main {
      output.dir("${layout.buildDirectory.asFile.get()}/generated/properties",
                 "builtBy" to "generateVersionResource")
    }
  }
}
```

### 条件触发

```kotlin
plugins.withId("otel.publish-conventions") {
    // 仅当应用了 otel.publish-conventions 插件时执行
}
```

**原因**: 只有发布的模块需要版本资源文件。

### 生成路径计算

```
moduleName = "io.opentelemetry.api"
↓
propertiesDir = build/generated/properties/io/opentelemetry/api/
```

### 生成的文件

**路径**: `build/generated/properties/io/opentelemetry/api/version.properties`
```properties
sdk.version=1.35.0
```

### 使用示例

**Java 代码中读取版本**:
```java
import java.io.InputStream;
import java.util.Properties;

public class VersionUtils {
    public static String getSdkVersion() {
        try (InputStream in = VersionUtils.class
                .getResourceAsStream("/io/opentelemetry/api/version.properties")) {
            Properties props = new Properties();
            props.load(in);
            return props.getProperty("sdk.version");
        } catch (Exception e) {
            return "unknown";
        }
    }
}
```

---

## 16. 依赖管理配置（第 207-251 行）

### 依赖冲突策略

```kotlin
configurations.configureEach {
  resolutionStrategy {
    failOnVersionConflict()
    preferProjectModules()
  }
}
```

**`failOnVersionConflict()`**:
- 依赖版本冲突时**立即失败**构建
- 强制显式解决冲突（而非静默选择版本）

**示例**:
```
项目依赖:
  ├─ guava:30.0
  └─ dep-A
       └─ guava:29.0  ❌ 冲突！

构建失败: "Conflict found for guava"
```

**`preferProjectModules()`**:
- 优先使用项目模块（而非外部依赖）
- 用于复合构建（composite builds）

### 依赖管理平台

```kotlin
val dependencyManagement by configurations.creating {
  isCanBeConsumed = false
  isCanBeResolved = false
}

dependencies {
  dependencyManagement(platform(project(":dependencyManagement")))

  afterEvaluate {
    configurations.configureEach {
      if (isCanBeResolved && !isCanBeConsumed) {
        extendsFrom(dependencyManagement)
      }
    }
  }
}
```

#### 核心功能概述

这段代码位于约定插件中，它的核心作用是**自动化地将项目的 BOM（版本清单）应用到所有的依赖配置中**。

简单来说：**"只要应用了这个插件，项目中的所有依赖（无论是编译、运行还是测试）都会自动遵循 `:dependencyManagement` 模块中定义的版本号，无需手动一个个添加 `platform(...)`。"**

#### 工作原理逐步解析

**步骤 1: 创建专用配置容器**

```kotlin
val dependencyManagement by configurations.creating {
  isCanBeConsumed = false
  isCanBeResolved = false
}
```

- **`configurations.creating`**: 创建名为 `dependencyManagement` 的新配置项
- **`isCanBeConsumed = false`**: 这个配置**不能**被其他项目引用（不是用来生成 jar 包给别人用的）
- **`isCanBeResolved = false`**: 这个配置**不能**被直接解析（不能直接下载里面的 jar 包）
- **总结**：创建了一个"虚拟的篮子"，专门用来存放版本约束规则，而不是具体的 jar 包

**步骤 2: 将 BOM 放入篮子**

```kotlin
dependencies {
  dependencyManagement(platform(project(":dependencyManagement")))
}
```

- 将 `:dependencyManagement` 项目（定义了所有版本的模块）作为 `platform` 依赖
- 添加到刚才创建的 `dependencyManagement` "篮子"里
- 此时，这个"篮子"里装满了版本约束规则

**步骤 3: 自动注入到所有配置（魔法所在）**

```kotlin
afterEvaluate {
  configurations.configureEach {
    if (isCanBeResolved && !isCanBeConsumed) {
      extendsFrom(dependencyManagement)
    }
  }
}
```

- **`afterEvaluate`**: 等当前子项目的 `build.gradle.kts` 脚本全部执行完，配置都加载好了，再执行这段代码（确保能捕获到用户在脚本中定义的所有配置）
- **`configurations.configureEach`**: 遍历当前子项目中的**每一个**配置项（如 `implementation`, `api`, `runtimeOnly`, `testImplementation` 等）
- **`if (isCanBeResolved && !isCanBeConsumed)`**: 过滤器，只关心那些**需要解析依赖**的配置（编译时、运行时需要找 jar 包的配置）
- **`extendsFrom(dependencyManagement)`**: **最关键的一句**
  - 让当前配置（如 `implementation`）继承 `dependencyManagement` 这个"篮子"
  - **结果**：`implementation` 自动获得了 BOM 中的所有版本约束

#### 为什么要这么设计？

**没有这段配置时（传统方式）**：

你必须在每个子模块的 `build.gradle.kts` 里重复写：

```kotlin
dependencies {
    implementation(platform(project(":dependencyManagement")))
    testImplementation(platform(project(":dependencyManagement")))
    // 甚至自定义的配置也要写...

    implementation("io.grpc:grpc-api:1.78.0")
    implementation("com.google.guava:guava:33.5.0-jre")
}
```

**有了这段配置后（自动化）**：

你只需要：

```kotlin
plugins {
    id("otel.java-conventions") // 应用这个插件
}

dependencies {
    // ✅ 不需要写版本号，也不需要显式引入 platform
    implementation("io.grpc:grpc-api")
    implementation("com.google.guava:guava")
}
```

Gradle 会自动通过 `extendsFrom` 机制，把 BOM 里的版本号应用到 `grpc-api` 和 `guava` 上。

**优势**：
- ✅ **减少样板代码**：无需在每个模块重复 `platform(...)` 声明
- ✅ **自动应用**：所有配置（包括自定义配置）自动继承版本约束
- ✅ **版本一致性**：确保整个项目使用相同的依赖版本
- ✅ **维护简单**：只需在一个地方（`:dependencyManagement` 模块）更新版本

#### 实践建议

如果你打算构建自己的 OpenTelemetry 发行版，**强烈建议复制这种模式**：

1. 建立 `:dependencyManagement` 模块定义版本
2. 在 `buildSrc` 中建立约定插件，包含上述自动注入代码
3. 在你的 `:agent` 和 `:custom-extension` 模块中应用该插件

这样可以极大减少样板代码，并保证你的 Agent 和扩展使用的依赖版本绝对一致，避免运行时出现 `NoSuchMethodError` 等版本冲突问题。

#### Gradle Configuration 的两个核心属性

在理解上述代码前，需要了解 Gradle Configuration 的两个关键属性：

**1. `isCanBeResolved` - 能否解析依赖**

**含义**：
- `true`：可以解析依赖（下载 JAR 文件，实际使用）
- `false`：不能解析依赖（仅用于声明，不实际使用）

**示例**：
```kotlin
// ❌ 不可解析配置
val api by configurations.creating {
    isCanBeResolved = false
}
dependencies {
    api("com.google.guava:guava:32.1.3-jre")
}
api.files  // 抛出异常：Configuration 'api' is not resolvable

// ✅ 可解析配置
val compileClasspath by configurations.creating {
    isCanBeResolved = true
    extendsFrom(api)
}
compileClasspath.files  // 返回 [guava-32.1.3-jre.jar, ...]
```

**2. `isCanBeConsumed` - 能否被其他项目消费**

**含义**：
- `true`：其他项目可以依赖这个配置
- `false`：仅供内部使用，不对外发布

**示例**：
```kotlin
// ❌ 不可消费配置（内部使用）
val compileClasspath by configurations.creating {
    isCanBeConsumed = false  // 其他项目无法依赖
}

// ✅ 可消费配置（发布给其他项目）
val apiElements by configurations.creating {
    isCanBeConsumed = true  // 其他项目可以依赖
}
```

**四种配置组合模式**：

| 类型 | `isCanBeResolved` | `isCanBeConsumed` | 作用 | 示例 |
|------|-------------------|-------------------|------|------|
| **声明型** | `false` | `false` | 仅声明依赖 | `api`, `implementation` |
| **可解析** | `true` | `false` | 实际使用依赖 | `compileClasspath`, `runtimeClasspath` |
| **可消费** | `false` | `true` | 发布给其他项目 | `apiElements`, `runtimeElements` |
| **遗留型** | `true` | `true` | 不推荐（角色混淆）| `compile`（已废弃）|

**`dependencyManagement` 配置为什么这样设置**：

```kotlin
val dependencyManagement by configurations.creating {
    isCanBeResolved = false  // 只存储版本信息，不实际下载 JAR
    isCanBeConsumed = false  // 仅供项目内部使用，不对外发布
}
```

**原因**：
- `dependencyManagement` 是一个"版本目录"，不需要实际下载依赖
- 它仅用于版本管理，其他配置继承它的版本信息后再解析
- 它是项目内部机制，不需要对外发布

#### 实际打包和依赖传递示例

**场景 1: 单模块项目的完整配置链**

```kotlin
// 完整示例：从声明到打包
plugins {
    `java-library`
}

// 1. 声明型配置（仅记录依赖）
val api by configurations.creating {
    isCanBeResolved = false  // 不能 .files
    isCanBeConsumed = false  // 不对外发布
}

// 2. 可解析配置（实际使用）
val compileClasspath by configurations.creating {
    isCanBeResolved = true   // 可以 .files，下载 JAR
    isCanBeConsumed = false
    extendsFrom(api)
}

// 3. 可消费配置（对外发布）
val apiElements by configurations.creating {
    isCanBeResolved = false
    isCanBeConsumed = true   // 其他项目可以依赖
    extendsFrom(api)
}

dependencies {
    api("com.google.guava:guava:32.1.3-jre")
}

// 验证任务
tasks.register("showClasspath") {
    doLast {
        // api.files  // ❌ 抛出异常：Configuration 'api' is not resolvable

        println("compileClasspath 文件:")
        compileClasspath.files.forEach {
            println("  - ${it.name}")
        }
        // 输出: guava-32.1.3-jre.jar, failureaccess-1.0.1.jar, ...
    }
}

// Fat JAR 打包（包含所有依赖）
tasks.jar {
    from(compileClasspath.map {
        if (it.isDirectory) it else zipTree(it)
    })
}
```

**场景 2: 多模块项目的依赖传递**

```kotlin
// 项目结构:
// ├── module-a (库)
// └── module-b (依赖 module-a)

// ========== module-a/build.gradle.kts ==========
plugins {
    `java-library`
    `maven-publish`
}

val apiElements by configurations.getting {
    // java-library 插件已创建此配置
    // isCanBeResolved = false
    // isCanBeConsumed = true
}

dependencies {
    api("com.google.guava:guava:32.1.3-jre")      // API 依赖（传递）
    implementation("org.slf4j:slf4j-api:2.0.9")   // 实现依赖（不传递）
}

// ========== module-b/build.gradle.kts ==========
dependencies {
    implementation(project(":module-a"))
}

tasks.register("showDependencies") {
    doLast {
        println("module-b 的编译类路径:")
        configurations.compileClasspath.get().files.forEach {
            println("  ${it.name}")
        }
        // 输出:
        // module-a.jar
        // guava-32.1.3-jre.jar  ← 传递依赖（来自 module-a 的 api）
        // （不包含 slf4j-api，因为是 implementation）
    }
}
```

**场景 3: Variant 属性匹配机制**

```kotlin
// module-a: 发布多个 Variant
val apiElements by configurations.creating {
    isCanBeConsumed = true
    attributes {
        attribute(Usage.USAGE_ATTRIBUTE, objects.named(Usage.JAVA_API))
        attribute(Category.CATEGORY_ATTRIBUTE, objects.named(Category.LIBRARY))
        attribute(LibraryElements.LIBRARY_ELEMENTS_ATTRIBUTE,
                  objects.named(LibraryElements.JAR))
    }
}

val runtimeElements by configurations.creating {
    isCanBeConsumed = true
    attributes {
        attribute(Usage.USAGE_ATTRIBUTE, objects.named(Usage.JAVA_RUNTIME))
        attribute(Category.CATEGORY_ATTRIBUTE, objects.named(Category.LIBRARY))
        attribute(LibraryElements.LIBRARY_ELEMENTS_ATTRIBUTE,
                  objects.named(LibraryElements.JAR))
    }
}

// module-b: Gradle 自动选择匹配的 Variant
dependencies {
    implementation(project(":module-a"))
    // 编译时 → 解析 apiElements（Usage.JAVA_API）
    // 运行时 → 解析 runtimeElements（Usage.JAVA_RUNTIME）
}
```

#### 调试命令和诊断方法

**查看项目配置**:
```bash
# 列出所有配置
./gradlew :api:configurations

# 查看特定配置的依赖树
./gradlew :api:dependencies --configuration compileClasspath

# 查看可消费的配置（outgoing variants）
./gradlew :api:outgoingVariants

# 查看可解析的配置
./gradlew :api:resolvableConfigurations

# 查看配置详细信息
./gradlew :api:dependencyInsight --dependency guava --configuration compileClasspath
```

**在代码中检查配置属性**:
```kotlin
tasks.register("inspectConfigurations") {
    doLast {
        configurations.forEach { config ->
            println("Configuration: ${config.name}")
            println("  isCanBeResolved: ${config.isCanBeResolved}")
            println("  isCanBeConsumed: ${config.isCanBeConsumed}")
            if (config.isCanBeResolved) {
                println("  files count: ${config.files.size}")
            }
            println()
        }
    }
}

// 输出示例:
// Configuration: api
//   isCanBeResolved: false
//   isCanBeConsumed: false
//
// Configuration: compileClasspath
//   isCanBeResolved: true
//   isCanBeConsumed: false
//   files count: 15
//
// Configuration: apiElements
//   isCanBeResolved: false
//   isCanBeConsumed: true
```

#### 常见问题和解决方案

**问题 1: Configuration 'xxx' is not resolvable**

```kotlin
// ❌ 错误代码
val api by configurations.creating {
    isCanBeResolved = false
}
dependencies {
    api("com.google.guava:guava:32.1.3-jre")
}
api.files  // 抛出异常：Resolving dependency configuration 'api' is not allowed

// ✅ 解决方案
val compileClasspath by configurations.creating {
    isCanBeResolved = true
    extendsFrom(api)  // 继承 api 的依赖
}
compileClasspath.files  // 正常工作，返回 guava JAR
```

**问题 2: 其他模块看不到我的依赖**

```kotlin
// ❌ 错误配置（module-a）
val api by configurations.creating {
    isCanBeConsumed = false  // 不对外发布
}
dependencies {
    api("com.google.guava:guava:32.1.3-jre")
}

// module-b 无法获取 guava 依赖

// ✅ 解决方案：创建可消费的配置
val apiElements by configurations.creating {
    isCanBeConsumed = true   // 其他项目可以消费
    extendsFrom(api)
    attributes {
        attribute(Usage.USAGE_ATTRIBUTE, objects.named(Usage.JAVA_API))
    }
}
```

**问题 3: 依赖传递不生效**

```kotlin
// ❌ module-a: 使用 implementation（不传递）
dependencies {
    implementation("com.google.guava:guava:32.1.3-jre")
}

// module-b: 无法看到 guava
dependencies {
    implementation(project(":module-a"))
    // compileClasspath 中没有 guava
}

// ✅ 解决方案：module-a 使用 api（传递）
dependencies {
    api("com.google.guava:guava:32.1.3-jre")
}

// 现在 module-b 可以看到 guava
```

**问题 4: Fat JAR 构建失败**

```kotlin
// ❌ 错误：使用不可解析的配置
tasks.jar {
    from(api.files)  // 抛出异常
}

// ✅ 解决方案：使用可解析的配置
tasks.jar {
    from(configurations.compileClasspath.get().map {
        if (it.isDirectory) it else zipTree(it)
    })
}
```

#### 依赖解析流程详解

```
1. module-b 声明依赖 project(":module-a")
   ↓
2. Gradle 查找 module-a 的可消费配置
   - 筛选条件: isCanBeConsumed = true
   ↓
3. 找到多个可消费配置（apiElements, runtimeElements）
   ↓
4. 根据 Variant Attributes 选择匹配的配置
   - 编译时: 选择 Usage.JAVA_API → apiElements
   - 运行时: 选择 Usage.JAVA_RUNTIME → runtimeElements
   ↓
5. 解析选中配置的依赖
   - extendsFrom 链: apiElements → api → dependencies
   ↓
6. 将依赖添加到 module-b 的 compileClasspath/runtimeClasspath
   - api 依赖传递，implementation 不传递
   ↓
7. 递归解析传递依赖
```

#### Configuration 属性快速参考

| 场景 | isCanBeResolved | isCanBeConsumed | 典型用途 |
|------|----------------|-----------------|----------|
| 声明依赖 | `false` | `false` | `api`, `implementation` - 记录依赖 |
| 编译/运行 | `true` | `false` | `compileClasspath` - 实际使用 JAR |
| 发布 API | `false` | `true` | `apiElements` - 供其他项目消费 |
| 版本管理 | `false` | `false` | `dependencyManagement` - 仅存储版本 |

#### 依赖管理工作流程

```
┌─────────────────────────────────────────────────────────┐
│ :dependencyManagement 项目                              │
│ constraints { guava:32.1.3-jre, mockito:5.8.0 }        │
└─────────────────────┬───────────────────────────────────┘
                      │ platform(...)
                      ↓
┌─────────────────────────────────────────────────────────┐
│ dependencyManagement 配置（版本目录）                    │
│ - isCanBeResolved: false (不下载 JAR)                  │
│ - isCanBeConsumed: false (不对外发布)                  │
└─────────────────────┬───────────────────────────────────┘
                      │ extendsFrom(...)
                      ↓
┌──────────────────────────────────────────────────────────┐
│ compileClasspath 配置（实际使用）                        │
│ - isCanBeResolved: true (下载 JAR)                     │
│ - isCanBeConsumed: false                                │
│ - 继承版本 → 解析依赖 → 下载 JAR                        │
└──────────────────────────────────────────────────────────┘
                      │
                      ↓
┌──────────────────────────────────────────────────────────┐
│ 子项目依赖声明（无需版本）                               │
│ dependencies { implementation("guava") }                 │
│ 自动使用 32.1.3-jre                                     │
└──────────────────────────────────────────────────────────┘
```

**效果**:
```kotlin
// 子项目不需要指定版本
dependencies {
    implementation("com.google.guava:guava")  // 自动使用管理的版本
}
```

### Mockito Agent 配置

```kotlin
val mockitoAgent by configurations.creating {
  extendsFrom(dependencyManagement)
}

dependencies {
  mockitoAgent("org.mockito:mockito-core")
}
```

**用途**: 用于测试套件配置中的 Mockito Agent 预加载（解决 Java 21+ 动态代理加载警告）。

### 通用依赖

```kotlin
dependencies {
  compileOnly("com.google.auto.value:auto-value-annotations")
  compileOnly("com.google.code.findbugs:jsr305")
  annotationProcessor("com.google.guava:guava-beta-checker")
  compileOnly("javax.annotation:javax.annotation-api")

  modules {
    module("com.google.collections:google-collections") {
      replacedBy("com.google.guava:guava", "google-collections is now part of Guava")
    }
  }
}
```

| 依赖 | 配置 | 用途 |
|------|------|------|
| `auto-value-annotations` | `compileOnly` | AutoValue 注解（编译时） |
| `jsr305` | `compileOnly` | `@Nullable`、`@NonNull` 注解 |
| `guava-beta-checker` | `annotationProcessor` | 检查 Guava Beta API 使用 |
| `javax.annotation-api` | `compileOnly` | `@Generated` 注解（gRPC 兼容）|

**模块替换**: 自动将旧依赖 `google-collections` 替换为 `guava`，解决 Java 9+ 模块冲突。

---

## 17. 测试套件配置（第 253-298 行）

```kotlin
testing {
  suites.withType(JvmTestSuite::class).configureEach {
    useJUnitJupiter()

    dependencies {
      implementation(project(project.path))
      implementation(project(":testing-internal"))

      compileOnly("com.google.auto.value:auto-value-annotations")
      compileOnly("com.google.errorprone:error_prone_annotations")
      compileOnly("com.google.code.findbugs:jsr305")

      implementation("nl.jqno.equalsverifier:equalsverifier")
      implementation("org.mockito:mockito-core")
      implementation("org.mockito:mockito-junit-jupiter")
      implementation("org.assertj:assertj-core")
      implementation("org.awaitility:awaitility")
      implementation("org.junit-pioneer:junit-pioneer")
      implementation("io.github.netmikey.logunit:logunit-jul")

      runtimeOnly("org.slf4j:slf4j-simple")
    }

    targets {
      all {
        testTask.configure {
          systemProperty("java.util.logging.config.class",
                         "io.opentelemetry.internal.testing.slf4j.JulBridgeInitializer")

          val mockitoAgent: FileCollection = mockitoAgent
          doFirst {
            val mockitoAgentJar = mockitoAgent.files.single {
              it.name.contains("byte-buddy-agent")
            }
            jvmArgs("-javaagent:${mockitoAgentJar}")
          }
        }
      }
    }
  }
}
```

### JUnit Jupiter 配置

```kotlin
useJUnitJupiter()
```

所有测试套件使用 JUnit 5。

### 测试工具库

| 库 | 用途 |
|-----|------|
| `equalsverifier` | 自动验证 `equals()` 和 `hashCode()` 实现 |
| `mockito-core` | Mock 框架 |
| `mockito-junit-jupiter` | Mockito + JUnit 5 集成 |
| `assertj-core` | 流式断言库 |
| `awaitility` | 异步测试工具 |
| `junit-pioneer` | JUnit 5 扩展 |
| `logunit-jul` | 日志断言工具 |
| `slf4j-simple` | 简单日志实现 |

### 使用示例

```java
// equalsverifier
@Test
void testEquals() {
    EqualsVerifier.forClass(MyClass.class).verify();
}

// assertj
@Test
void testAssertJ() {
    assertThat(list)
        .hasSize(3)
        .contains("foo", "bar");
}

// awaitility
@Test
void testAsync() {
    await().atMost(5, SECONDS)
           .until(() -> service.isReady());
}
```

### JUL 桥接配置

```kotlin
systemProperty("java.util.logging.config.class",
               "io.opentelemetry.internal.testing.slf4j.JulBridgeInitializer")
```

**作用**: 将 `java.util.logging`（JUL）日志桥接到 SLF4J，统一测试日志输出。

**工作流程**:
```
JUL 日志 ➜ JulBridgeInitializer ➜ SLF4J ➜ slf4j-simple ➜ 控制台
```

### Mockito Agent 预加载

```kotlin
val mockitoAgent: FileCollection = mockitoAgent
doFirst {
  val mockitoAgentJar = mockitoAgent.files.single {
    it.name.contains("byte-buddy-agent")
  }
  jvmArgs("-javaagent:${mockitoAgentJar}")
}
```

**问题背景**: Java 21+ 引入了动态代理加载警告：
```
WARNING: A Java agent has been loaded dynamically
WARNING: Dynamic loading of agents will be disallowed by default in a future release
```

**原因**:
- Mockito 使用 Byte Buddy 动态生成 Mock 类
- Byte Buddy 需要附加 Java Agent（`byte-buddy-agent.jar`）
- Java 21+ 不允许运行时动态附加 Agent（会触发警告）

**解决方案**:
- 通过 `-javaagent` JVM 参数**预先加载** Agent
- 从 `mockitoAgent` 配置中提取 `byte-buddy-agent-*.jar`
- 在测试 JVM 启动时附加

**效果**:
```bash
# 没有预加载（产生警告）
java -cp ... org.junit.runner.JUnit5 MyTest
WARNING: A Java agent has been loaded dynamically...

# 有预加载（无警告）
java -javaagent:byte-buddy-agent-1.14.9.jar -cp ... org.junit.runner.JUnit5 MyTest
✓ 测试运行，无警告
```

**最佳实践**:
- ✅ Java 21+: **必须**预加载 Agent
- ✅ Java 17: 可选（有警告但不影响功能）
- ✅ Java 8-11: 不需要（无警告）

---

## 18. 总结

### 核心特性

| 特性 | 实现位置 | 关键价值 |
|------|----------|----------|
| **向后兼容** | 第 78 行（release 参数）| Java 21 编译，Java 8+ 运行 |
| **可重现构建** | 第 30-33 行 | 字节级相同的构建产物 |
| **安全扫描** | 第 51-71 行 | CVSS 7.0+ 漏洞自动拦截 |
| **测试重试** | 第 112-118 行 | CI 环境自动重试不稳定测试 |
| **多版本测试** | 第 170-181 行 | 一次提交测试多个 Java 版本 |
| **严格质量门禁** | 第 80-98 行 | -Werror 将警告视为错误 |
| **统一依赖版本** | 第 214-231 行 | BOM 平台管理 |
| **Mockito Agent 优化** | 第 283-293 行 | 解决 Java 21+ 警告 |

### 配置层次结构

```
otel.java-conventions
├── 继承的插件
│   ├── otel.errorprone-conventions
│   ├── otel.jacoco-conventions
│   └── otel.spotless-conventions
├── 配置的工具
│   ├── Checkstyle 13.0.0
│   └── OWASP Dependency Check
└── 集成的功能
    ├── Java 工具链（Java 21）
    ├── 测试框架（JUnit 5）
    ├── 依赖管理平台
    └── 版本资源生成
```

这 299 行代码，为整个 OpenTelemetry Java 项目的 **50+ 模块** 提供了统一、高质量的构建基础！

---

**相关文档**:
- [buildSrc 主文档](../README.md)
- [AnimalSniffer 完整指南](animalsniffer-guide.md)
- [BOM 约定完整指南](bom-guide.md)

**问题反馈**: [GitHub Issues](https://github.com/open-telemetry/opentelemetry-java/issues)

**贡献指南**: [CONTRIBUTING.md](../../CONTRIBUTING.md)

---

**最后更新**: 2026-01-10
**文档版本**: 1.0.0
