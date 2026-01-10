# 依赖管理

本模块使用 Gradle 的 `java-platform` 插件为整个 OpenTelemetry Java 项目提供集中式的依赖版本管理。

## 用途

`dependencyManagement` 模块作为项目 45+ 个模块中所有外部依赖版本的单一数据源。它确保：

- **版本一致性**：所有模块使用相同版本的依赖
- **冲突预防**：消除传递性依赖之间的版本冲突
- **简化维护**：在一个地方更新版本，而不是跨多个构建文件
- **强制标准**：强制执行 BOM 以防止意外的版本覆盖

## 工作原理

本模块使用 Gradle 的 `java-platform` 插件创建一个依赖平台：

1. 为所有外部依赖定义版本约束
2. 通过约定插件 (`otel.java-conventions`) 自动应用到所有模块
3. 对 BOM 使用 `enforcedPlatform()` 严格控制传递性依赖版本
4. 向根项目暴露一个 `versions` 映射，供构建脚本引用

当模块声明依赖而不指定版本时：
```kotlin
dependencies {
  implementation("com.google.guava:guava")  // 版本从平台继承
}
```

版本会自动从该平台解析。

## 配置代码详解

本节深入解析 `build.gradle.kts` 配置文件的核心技术实现，帮助理解 BOM 构建机制的工作原理。

### 1. 核心插件：`java-platform`

```kotlin
plugins {
  `java-platform`
}
```

**作用：** 声明这个模块是一个"Java 平台"（Java Platform）模块。

**关键特性：**
- **不生成代码 JAR**：与普通的 Java 模块不同，这个模块不会生成包含 `.class` 文件的 JAR 包
- **生成 BOM POM**：生成的是一个包含依赖版本约束的 POM 文件（即 Bill of Materials）
- **被动消费**：其他模块通过 `platform(project(":dependencyManagement"))` 引入它，从而继承这里定义的所有版本约束

**与普通 Java 模块的区别：**

| 特性 | 普通 Java 模块 | java-platform 模块 |
|------|---------------|-------------------|
| 输出产物 | JAR（包含编译后的类） | POM（仅包含依赖元数据） |
| 用途 | 提供可执行代码 | 提供版本约束 |
| 依赖声明 | 会下载并打包依赖 | 仅定义版本约束 |

### 2. 版本变量设计

```kotlin
val autoValueVersion = "1.11.1"
val errorProneVersion = "2.45.0"
val jmhVersion = "1.37"
val mockitoVersion = "4.11.0"  // Mockito 5.x.x requires Java 11
val slf4jVersion = "2.0.17"
val opencensusVersion = "0.31.1"
val prometheusServerVersion = "1.3.10"
val armeriaVersion = "1.35.0"
val junitVersion = "5.14.2"
val okhttpVersion = "5.3.2"
```

**设计原则：**

1. **集中管理**：所有关键库的版本号集中定义在文件顶部，易于查找和维护
2. **语义化命名**：变量名采用 `<库名>Version` 的命名模式，清晰明确
3. **复用性**：当同一个库的多个模块都需要使用相同版本时，使用变量避免重复

**特殊版本约束案例：**

```kotlin
// Mockito 5.x.x requires Java 11 https://github.com/mockito/mockito/releases/tag/v5.0.0
val mockitoVersion = "4.11.0"
```
- **原因**：项目仍然支持 Java 8，而 Mockito 5.x 需要 Java 11+
- **决策**：将 Mockito 固定在 4.x 系列的最新版本

```kotlin
// jqf-fuzz version 1.8+ requires Java 11+
"edu.berkeley.cs.jqf:jqf-fuzz:1.7"
```
- **原因**：同样的 Java 版本兼容性考虑
- **实践**：在注释中明确说明版本限制原因，便于后续维护者理解

### 3. 全局版本导出机制

```kotlin
val dependencyVersions = hashMapOf<String, String>()
rootProject.extra["versions"] = dependencyVersions
```

**技术实现：**

这是一个巧妙的设计，创建了一个 `HashMap` 并将其挂载到根项目的扩展属性中。

**工作流程：**

1. 创建空的 `dependencyVersions` Map
2. 在解析 BOM 和依赖时，将每个库的 `groupId` 和 `version` 存入 Map
3. 通过 `rootProject.extra["versions"]` 暴露给整个项目

**实际应用：**

```kotlin
// 在项目的其他 Gradle 脚本中访问版本信息
val versions = rootProject.extra["versions"] as Map<String, String>
val grpcVersion = versions["io.grpc"]  // 获取 gRPC 的版本号
```

**优势：**
- **单一数据源**：所有版本信息集中存储
- **编程访问**：可以在构建脚本中动态读取版本号
- **文档生成**：可以自动生成版本清单文档

### 4. 依赖分类策略

#### A. DEPENDENCY_BOMS（外部 BOM 导入）

```kotlin
val DEPENDENCY_BOMS = listOf(
  "com.fasterxml.jackson:jackson-bom:2.20.1",
  "io.grpc:grpc-bom:1.78.0",
  "io.netty:netty-bom:4.2.9.Final",
  // ...
)
```

**为什么引入外部 BOM？**

像 Jackson、Netty、gRPC 这样的库通常由多个模块组成。例如 Jackson 包含：
- `jackson-core`：核心功能
- `jackson-databind`：数据绑定
- `jackson-annotations`：注解支持
- `jackson-dataformat-xml`：XML 支持
- 还有 20+ 个其他模块...

**版本对齐的重要性：**

```kotlin
// ❌ 错误示例：版本不一致导致问题
implementation("com.fasterxml.jackson.core:jackson-core:2.14.0")
implementation("com.fasterxml.jackson.core:jackson-databind:2.15.0")
// 可能导致运行时错误：NoSuchMethodError, ClassNotFoundException 等

// ✅ 正确示例：通过 BOM 确保版本一致
implementation(platform("com.fasterxml.jackson:jackson-bom:2.20.1"))
implementation("com.fasterxml.jackson.core:jackson-core")      // 版本继承
implementation("com.fasterxml.jackson.core:jackson-databind")  // 版本继承
```

**许可证合规性考虑：**

```kotlin
// 注释说明为何不使用 JUnit 和 Armeria 的 BOM
// for some reason boms show up as runtime dependencies in license and vulnerability scans
// even if they are only used by test dependencies, so not using junit bom here
// (which is EPL licensed) or armeria bom (which is Apache licensed but is getting flagged
// by FOSSA for containing EPL-licensed)
```

这体现了企业级开源项目对**许可证合规**的重视：
- JUnit 采用 EPL（Eclipse Public License）许可证
- 即使仅在测试中使用，BOM 也会被扫描工具标记为运行时依赖
- 因此手动管理 JUnit 版本，避免许可证扫描工具误报

#### B. DEPENDENCIES（直接依赖管理）

```kotlin
val DEPENDENCIES = listOf(
  "org.junit.jupiter:junit-jupiter-api:${junitVersion}",
  "com.google.guava:guava-beta-checker:1.0",
  "io.opentelemetry.proto:opentelemetry-proto:1.9.0-alpha",
  // ...
)
```

这里列出的是：
1. **没有 BOM 的库**：如 `guava-beta-checker`
2. **不使用 BOM 的库**：如 JUnit（出于合规考虑）
3. **项目特定的库**：如 OpenTelemetry 相关库

### 5. 版本约束应用机制

```kotlin
javaPlatform {
  allowDependencies()  // 允许平台声明依赖
}

dependencies {
  // 处理外部 BOM
  for (bom in DEPENDENCY_BOMS) {
    api(enforcedPlatform(bom))
    val split = bom.split(':')
    dependencyVersions[split[0]] = split[2]
  }

  // 处理直接依赖
  constraints {
    for (dependency in DEPENDENCIES) {
      api(dependency)
      val split = dependency.split(':')
      dependencyVersions[split[0]] = split[2]
    }
  }
}
```

#### A. `javaPlatform { allowDependencies() }`

**含义：** 允许平台模块声明依赖（即可以引入其他 BOM）

**如果不设置：** 平台模块只能定义约束（constraints），不能声明实际依赖

#### B. `enforcedPlatform()` 详解

**`enforcedPlatform()` vs `platform()` 的关键区别：**

| 特性 | platform() | enforcedPlatform() |
|------|-----------|-------------------|
| 版本优先级 | 建议版本（可被覆盖） | **强制版本**（覆盖一切） |
| 传递依赖 | 可能被更高版本覆盖 | **强制使用指定版本** |
| 用途 | 温和的版本建议 | 严格的版本控制 |

**实际效果示例：**

```kotlin
// 场景：你的项目依赖 A 和 B
dependencies {
  implementation("library-a:1.0")  // A 传递依赖 netty:4.1.50
  implementation("library-b:2.0")  // B 传递依赖 netty:4.1.80
}

// 使用 platform()
api(platform("io.netty:netty-bom:4.2.9.Final"))
// 结果：Gradle 可能选择 4.1.80（更高版本优先）

// 使用 enforcedPlatform() ✅
api(enforcedPlatform("io.netty:netty-bom:4.2.9.Final"))
// 结果：强制所有 Netty 模块使用 4.2.9.Final，覆盖传递依赖
```

**为什么选择 `enforcedPlatform()`？**

在 OpenTelemetry 这样的大型项目中：
- 有 45+ 个子模块
- 数百个传递依赖
- 需要**绝对的版本一致性**来避免"依赖地狱"

#### C. `constraints` 块详解

```kotlin
constraints {
  for (dependency in DEPENDENCIES) {
    api(dependency)
  }
}
```

**constraints 的核心概念：**

| 特性 | 普通 dependencies | constraints |
|------|------------------|-------------|
| 是否下载 | 立即下载 | **不下载** |
| 作用时机 | 无条件依赖 | **按需生效** |
| 语义 | "我需要这个库" | "如果有人需要这个库，使用这个版本" |

**工作机制：**

```kotlin
// 在 dependencyManagement 中定义约束
constraints {
  api("org.junit.jupiter:junit-jupiter-api:5.14.2")
}

// 在某个子模块中声明依赖（不指定版本）
dependencies {
  testImplementation("org.junit.jupiter:junit-jupiter-api")  // 无版本号
}

// 结果：Gradle 自动使用 5.14.2 版本（从约束继承）
```

**如果没有人使用：** 约束不会触发，不会下载任何文件

**优势：**
- **懒加载**：只在需要时生效
- **零开销**：未使用的约束不占用资源
- **推荐而非强制**：子模块可以选择性采纳

### 6. 实践应用示例

#### 场景：为自定义 OpenTelemetry 发行版创建 dependencyManagement

**步骤 1：创建 dependencyManagement 模块**

```kotlin
// my-otel-distro/dependencyManagement/build.gradle.kts
plugins {
  `java-platform`
}

val otelVersion = "1.52.0"
val grpcVersion = "1.78.0"

val DEPENDENCY_BOMS = listOf(
  "io.grpc:grpc-bom:${grpcVersion}",
  "com.fasterxml.jackson:jackson-bom:2.20.1"
)

val DEPENDENCIES = listOf(
  "io.opentelemetry:opentelemetry-api:${otelVersion}",
  "io.opentelemetry:opentelemetry-sdk:${otelVersion}",
  "io.opentelemetry:opentelemetry-exporter-otlp:${otelVersion}"
)

javaPlatform {
  allowDependencies()
}

dependencies {
  for (bom in DEPENDENCY_BOMS) {
    api(enforcedPlatform(bom))
  }
  constraints {
    for (dep in DEPENDENCIES) {
      api(dep)
    }
  }
}
```

**步骤 2：在自定义 Agent 中使用**

```kotlin
// my-otel-distro/custom-agent/build.gradle.kts
dependencies {
  // 引入 dependencyManagement 平台
  implementation(platform(project(":dependencyManagement")))

  // 之后所有依赖都不需要写版本号
  implementation("io.opentelemetry:opentelemetry-api")           // 版本自动继承
  implementation("io.opentelemetry:opentelemetry-sdk")           // 版本自动继承
  implementation("io.grpc:grpc-netty-shaded")                    // 从 grpc-bom 继承
  implementation("com.fasterxml.jackson.core:jackson-databind")  // 从 jackson-bom 继承
}
```

**优势：**
- ✅ **版本一致性**：所有模块使用相同的 OpenTelemetry 和 gRPC 版本
- ✅ **简化维护**：升级版本只需修改 dependencyManagement 一处
- ✅ **避免冲突**：`enforcedPlatform()` 防止传递依赖引入不兼容版本
- ✅ **清晰明了**：子模块的 `build.gradle.kts` 不再充斥版本号

## 受管理的依赖

### 物料清单 (BOMs)

这些 BOM 通过 `enforcedPlatform()` 导入，以控制传递性依赖版本：

| Group ID | Artifact ID | Version |
|----------|-------------|---------|
| com.fasterxml.jackson | jackson-bom | 2.20.1 |
| com.google.guava | guava-bom | 33.5.0-jre |
| com.google.protobuf | protobuf-bom | 4.33.2 |
| com.squareup.okhttp3 | okhttp-bom | 5.3.2 |
| com.squareup.okio | okio-bom | 3.16.4 |
| io.grpc | grpc-bom | 1.78.0 |
| io.netty | netty-bom | 4.2.9.Final |
| io.zipkin.brave | brave-bom | 6.3.0 |
| io.zipkin.reporter2 | zipkin-reporter-bom | 3.5.1 |
| org.assertj | assertj-bom | 3.27.6 |
| org.testcontainers | testcontainers-bom | 2.0.3 |
| org.snakeyaml | snakeyaml-engine | 2.10 |

> **注意**：故意不使用 JUnit BOM，因为即使仅被测试依赖使用，BOM 也会在许可证/漏洞扫描中显示为运行时依赖。JUnit 采用 EPL 许可证，需要特殊处理。

### 直接依赖

#### 测试框架
| 依赖 | 版本 |
|------|------|
| org.junit.jupiter:junit-jupiter-api | 5.14.2 |
| org.junit.jupiter:junit-jupiter-params | 5.14.2 |
| org.mockito:mockito-core | 4.11.0 |
| org.mockito:mockito-junit-jupiter | 4.11.0 |
| org.junit-pioneer:junit-pioneer | 1.9.1 |
| junit:junit | 4.13.2 |

> **注意**：Mockito 固定在 4.x 版本，因为 5.x 需要 Java 11+

#### 构建工具和代码质量
| 依赖 | 版本 |
|------|------|
| com.google.auto.value:auto-value | 1.11.1 |
| com.google.auto.value:auto-value-annotations | 1.11.1 |
| com.google.errorprone:error_prone_annotations | 2.45.0 |
| com.google.errorprone:error_prone_core | 2.45.0 |
| com.google.errorprone:error_prone_test_helpers | 2.45.0 |
| com.uber.nullaway:nullaway | 0.12.15 |
| com.google.guava:guava-beta-checker | 1.0 |
| org.codehaus.mojo:animal-sniffer-annotations | 1.26 |

#### 网络和 gRPC
| 依赖 | 版本 |
|------|------|
| com.linecorp.armeria:armeria | 1.35.0 |
| com.linecorp.armeria:armeria-grpc | 1.35.0 |
| com.linecorp.armeria:armeria-grpc-protocol | 1.35.0 |
| com.linecorp.armeria:armeria-junit5 | 1.35.0 |
| com.squareup.okhttp3:okhttp | 5.3.2 |
| com.google.api.grpc:proto-google-common-protos | 2.63.2 |

#### 可观测性库
| 依赖 | 版本 |
|------|------|
| io.opencensus:opencensus-api | 0.31.1 |
| io.opencensus:opencensus-impl-core | 0.31.1 |
| io.opencensus:opencensus-impl | 0.31.1 |
| io.opencensus:opencensus-exporter-metrics-util | 0.31.1 |
| io.opencensus:opencensus-contrib-exemplar-util | 0.31.1 |
| io.prometheus:prometheus-metrics-exporter-httpserver | 1.3.10 |
| io.prometheus:prometheus-metrics-exposition-formats-no-protobuf | 1.3.10 |
| io.jaegertracing:jaeger-client | 1.8.1 |
| io.opentracing:opentracing-api | 0.33.0 |
| io.opentracing:opentracing-noop | 0.33.0 |

#### OpenTelemetry 扩展
| 依赖 | 版本 |
|------|------|
| io.opentelemetry.contrib:opentelemetry-aws-xray-propagator | 1.52.0-alpha |
| io.opentelemetry.semconv:opentelemetry-semconv-incubating | 1.37.0-alpha |
| io.opentelemetry.proto:opentelemetry-proto | 1.9.0-alpha |

#### 测试基础设施
| 依赖 | 版本 |
|------|------|
| org.awaitility:awaitility | 4.3.0 |
| nl.jqno.equalsverifier:equalsverifier | 3.19.4 |
| org.skyscreamer:jsonassert | 1.5.3 |
| com.tngtech.archunit:archunit-junit5 | 1.4.1 |
| org.mock-server:mockserver-netty | 5.15.0:shaded |
| eu.rekawek.toxiproxy:toxiproxy-java | 2.1.11 |
| io.github.netmikey.logunit:logunit-jul | 2.0.0 |
| com.github.stefanbirkner:system-rules | 1.19.0 |

#### 基准测试
| 依赖 | 版本 |
|------|------|
| org.openjdk.jmh:jmh-core | 1.37 |
| org.openjdk.jmh:jmh-generator-bytecode | 1.37 |
| org.openjdk.jmh:jmh-generator-annprocess | 1.37 |

#### 其他依赖
| 依赖 | 版本 |
|------|------|
| org.slf4j:slf4j-simple | 2.0.17 |
| org.slf4j:jul-to-slf4j | 2.0.17 |
| javax.annotation:javax.annotation-api | 1.3.2 |
| com.google.code.findbugs:jsr305 | 3.0.2 |
| com.sun.net.httpserver:http | 20070405 |
| org.jctools:jctools-core | 4.0.5 |
| org.bouncycastle:bcpkix-jdk15on | 1.70 |
| com.android.tools:desugar_jdk_libs | 2.1.5 |
| edu.berkeley.cs.jqf:jqf-fuzz | 1.7 |

> **注意**：jqf-fuzz 固定在 1.7 版本，因为 1.8+ 需要 Java 11+

## 版本变量

模块级别定义的关键版本常量：

| 变量 | 版本 | 说明 |
|------|------|------|
| autoValueVersion | 1.11.1 | Google Auto Value 代码生成器 |
| errorProneVersion | 2.45.0 | Error Prone 静态分析器 |
| jmhVersion | 1.37 | Java 微基准测试工具 |
| mockitoVersion | 4.11.0 | 固定在 4.x（5.x 需要 Java 11+） |
| slf4jVersion | 2.0.17 | SLF4J 日志门面 |
| opencensusVersion | 0.31.1 | OpenCensus 可观测性库 |
| prometheusServerVersion | 1.3.10 | Prometheus 指标导出器 |
| armeriaVersion | 1.35.0 | Armeria HTTP/gRPC 框架 |
| junitVersion | 5.14.2 | JUnit 5 测试框架 |
| okhttpVersion | 5.3.2 | OkHttp HTTP 客户端 |

`dependencyVersions` 映射通过 `rootProject.extra["versions"]` 暴露给根项目，供需要以编程方式引用版本的构建脚本使用。

## 集成方式

### 通过约定插件自动应用

平台通过 `otel.java-conventions` 插件自动应用到所有模块：

```kotlin
// buildSrc/src/main/kotlin/otel.java-conventions.gradle.kts
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

这意味着任何应用 `otel.java-conventions` 的模块都会自动继承所有版本约束。

### 专用插件集成

其他约定插件也与该平台集成：

**Protobuf 编译：**
```kotlin
// otel.protobuf-conventions.gradle.kts
dependencies {
  add("compileProtoPath", platform(project(":dependencyManagement")))
  add("testCompileProtoPath", platform(project(":dependencyManagement")))
}
```

**JMH 基准测试：**
```kotlin
// otel.jmh-conventions.gradle.kts
dependencies {
  jmh(platform(project(":dependencyManagement")))
  jmh("org.openjdk.jmh:jmh-core")
  jmh("org.openjdk.jmh:jmh-generator-bytecode")
}
```

## 维护指南

### 添加新依赖

1. **添加到 `build.gradle.kts` 中的相应列表**：
   - `DEPENDENCY_BOMS` 用于 BOM
   - `DEPENDENCIES` 用于直接依赖

2. **如果依赖在多个地方使用，使用版本变量**：
   ```kotlin
   val myLibVersion = "1.2.3"

   val DEPENDENCIES = listOf(
     // ...
     "com.example:my-lib:${myLibVersion}",
     "com.example:my-lib-test:${myLibVersion}",
   )
   ```

3. **如果需要固定版本，在注释中记录版本约束**：
   ```kotlin
   // MyLib 2.x 需要 Java 11+，所以我们保持在 1.x
   val myLibVersion = "1.9.9"
   ```

### 更新版本

1. **更新版本变量**或内联版本字符串
2. **跨模块测试**：
   ```bash
   ./gradlew build
   ```
3. **检查依赖变更日志中的破坏性更改**
4. **如果是协调发布的一部分，更新相关依赖**

### 添加 BOM

1. **添加到 DEPENDENCY_BOMS 列表**：
   ```kotlin
   val DEPENDENCY_BOMS = listOf(
     // ...
     "com.example:example-bom:1.2.3",
   )
   ```

2. **重要**：考虑许可证影响。即使仅被测试依赖使用，BOM 也会在许可证扫描中显示为运行时依赖。

### 最佳实践

- **版本对齐**：更新相关依赖（例如所有 Armeria 构件）时，应一起更新
- **Java 版本约束**：记录由于 Java 版本要求而固定版本的情况
- **BOM vs 直接依赖**：对于相关依赖系列（Jackson、gRPC 等）优先使用 BOM
- **许可证意识**：避免添加 EPL 许可的 BOM，以防许可证扫描复杂化
- **测试**：版本更新后始终运行完整构建，以捕获兼容性问题
- **变更日志审查**：主版本更新前审查上游变更日志，查找破坏性更改

## 架构上下文

本模块是构建基础设施的一部分：

```
opentelemetry-java/
├── dependencyManagement/          ← 版本定义（本模块）
├── buildSrc/                      ← 使用此平台的约定插件
│   └── src/main/kotlin/
│       ├── otel.java-conventions.gradle.kts
│       ├── otel.protobuf-conventions.gradle.kts
│       └── otel.jmh-conventions.gradle.kts
├── bom/                          ← 供外部用户使用的公共 BOM
├── bom-alpha/                    ← Alpha API BOM
├── api/                          ← API 模块（使用版本）
├── sdk/                          ← SDK 模块（使用版本）
└── [45+ 其他模块...]             ← 所有模块都从此平台使用版本
```

该平台使整个项目能够保持版本一致，无需在各个模块的构建文件中声明版本。
