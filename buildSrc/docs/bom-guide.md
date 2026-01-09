### 3.8 otel.bom-conventions.gradle.kts

**文件路径**: `src/main/kotlin/otel.bom-conventions.gradle.kts:1`

**核心功能**: BOM（Bill of Materials）项目配置，自动收集所有可发布的子项目，并生成依赖约束和复合构建替换文件。

#### 什么是 BOM（Bill of Materials）？

BOM（物料清单）是一种特殊的 Maven/Gradle 项目类型，用于集中管理多个相关库的版本。

**核心概念**:
```
┌──────────────────────────────────────────────────────┐
│ 外部项目                                              │
│                                                      │
│ dependencies {                                       │
│   // ✓ 导入 BOM，统一管理版本                         │
│   implementation(platform(                           │
│     "io.opentelemetry:opentelemetry-bom:1.35.0"))   │
│                                                      │
│   // ✓ 不需要指定版本（由 BOM 管理）                  │
│   implementation("io.opentelemetry:opentelemetry-api")│
│   implementation("io.opentelemetry:opentelemetry-sdk")│
│ }                                                    │
└──────────────────────────────────────────────────────┘
                         │
                         ↓ BOM 提供版本约束
┌──────────────────────────────────────────────────────┐
│ opentelemetry-bom (POM)                              │
│                                                      │
│ <dependencyManagement>                               │
│   <dependencies>                                     │
│     <dependency>                                     │
│       <groupId>io.opentelemetry</groupId>            │
│       <artifactId>opentelemetry-api</artifactId>     │
│       <version>1.35.0</version>                      │
│     </dependency>                                    │
│     <dependency>                                     │
│       <groupId>io.opentelemetry</groupId>            │
│       <artifactId>opentelemetry-sdk</artifactId>     │
│       <version>1.35.0</version>                      │
│     </dependency>                                    │
│     <!-- ... 50+ 个子项目 -->                         │
│   </dependencies>                                    │
│ </dependencyManagement>                              │
└──────────────────────────────────────────────────────┘
```

#### 为什么需要 BOM？

**问题场景（没有 BOM）**:
```kotlin
// ❌ 用户需要手动管理每个依赖的版本
dependencies {
    implementation("io.opentelemetry:opentelemetry-api:1.35.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.35.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.35.0")
    implementation("io.opentelemetry:opentelemetry-exporter-prometheus:1.35.0")
    // ... 繁琐且容易版本不一致
}

// ⚠️ 版本不一致导致的问题
dependencies {
    implementation("io.opentelemetry:opentelemetry-api:1.35.0")
    implementation("io.opentelemetry:opentelemetry-sdk:1.34.0")  // 版本不匹配！
    // 可能导致运行时错误、API 不兼容
}
```

**使用 BOM 的优势**:
```kotlin
// ✓ 只需指定一次 BOM 版本
dependencies {
    implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))

    // ✓ 不需要版本号，由 BOM 统一管理
    implementation("io.opentelemetry:opentelemetry-api")
    implementation("io.opentelemetry:opentelemetry-sdk")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
    implementation("io.opentelemetry:opentelemetry-exporter-prometheus")
}

// ✓ 保证所有依赖版本一致
// ✓ 升级只需更改 BOM 版本
// ✓ 简化依赖声明
```

#### Maven BOM vs Gradle Platform

| 特性 | Maven BOM | Gradle Platform |
|------|-----------|-----------------|
| **声明方式** | `<dependencyManagement>` | `dependencies { constraints { } }` |
| **使用方式** | `<dependency><type>pom</type><scope>import</scope>` | `implementation(platform(...))` |
| **发布格式** | POM 文件（无 JAR） | POM + JAR（可选）|
| **兼容性** | Maven、Gradle 通用 | 主要用于 Gradle |
| **约束类型** | 版本推荐（可覆盖）| 版本约束（可配置强制）|

**OpenTelemetry Java 策略**:
- 使用 Gradle `java-platform` 插件创建 BOM
- 发布标准的 Maven POM 格式
- 同时兼容 Maven 和 Gradle 用户

#### OpenTelemetry Java 的 BOM 架构

OpenTelemetry Java 项目维护两个 BOM：

```
┌─────────────────────────────────────────────────────┐
│ opentelemetry-bom (稳定版)                          │
│ - 包含所有稳定的 API 和实现                          │
│ - 适合生产环境使用                                   │
│ - groupId: io.opentelemetry                         │
│ - artifactId: opentelemetry-bom                     │
└─────────────────────────────────────────────────────┘
                      │
                      ↓ 继承（via api(platform(...))）
┌─────────────────────────────────────────────────────┐
│ opentelemetry-bom-alpha (孵化版)                    │
│ - 包含实验性、不稳定的 API                            │
│ - 供早期采用者测试新功能                              │
│ - 继承 stable BOM 的所有约束                         │
│ - groupId: io.opentelemetry                         │
│ - artifactId: opentelemetry-bom-alpha               │
└─────────────────────────────────────────────────────┘
```

**分层策略的优势**:
- ✅ 稳定 API 和实验性 API 分离
- ✅ 用户可以选择性地使用 alpha 功能
- ✅ alpha BOM 继承 stable BOM，保证版本一致性
- ✅ 便于管理 API 生命周期（alpha → stable → deprecated）

#### 完整配置详解

**otel.bom-conventions.gradle.kts 源码**（65 行）:
```kotlin
import io.opentelemetry.gradle.OtelBomExtension
import org.gradle.kotlin.dsl.create

plugins {
  id("otel.publish-conventions")
  id("java-platform")
}

if (!project.name.startsWith("bom")) {
  throw IllegalStateException("Name of BOM projects must start with 'bom'.")
}

rootProject.subprojects.forEach { subproject ->
  if (!subproject.name.startsWith("bom")) {
    evaluationDependsOn(subproject.path)
  }
}
val otelBom = extensions.create<OtelBomExtension>("otelBom")

val generateBuildSubstitutions by tasks.registering {
  group = "publishing"
  description = "Generate a code snippet that can be copy-pasted for use in composite builds."
}

tasks.named("publish") {
  dependsOn(generateBuildSubstitutions)
}

rootProject.tasks.named(generateBuildSubstitutions.name) {
  dependsOn(generateBuildSubstitutions)
}

afterEvaluate {
  otelBom.projectFilter.finalizeValue()
  val bomProjects = rootProject.subprojects
    .sortedBy { it.findProperty("archivesName") as String? }
    .filter { !it.name.startsWith("bom") }
    .filter(otelBom.projectFilter.get()::test)
    .filter { it.plugins.hasPlugin("maven-publish") }

  generateBuildSubstitutions {
    val outputFile = File(layout.buildDirectory.asFile.get(), "substitutions.gradle.kts")
    outputs.file(outputFile)
    val substitutionSnippet = bomProjects.joinToString(
      separator = "\n",
      prefix = "dependencySubstitution {\n",
      postfix = "\n}\n",
    ) { project ->
      val publication = project.publishing.publications.getByName("mavenPublication") as MavenPublication
      "  substitute(module(\"${publication.groupId}:${publication.artifactId}\")).using(project(\"${project.path}\"))"
    }
    inputs.property("projectPathsAndArtifactCoordinates", substitutionSnippet)
    doFirst {
      outputFile.writeText(substitutionSnippet)
    }
  }
  bomProjects.forEach { project ->
    dependencies {
      constraints {
        api(project)
      }
    }
  }
}
```

##### 逐段解析

**第 1-7 行：插件声明**
```kotlin
plugins {
  id("otel.publish-conventions")
  id("java-platform")
}
```

- `otel.publish-conventions`: 提供 Maven 发布配置（见 3.2 节）
- `java-platform`: Gradle 官方插件，用于创建平台（BOM）项目

**`java-platform` 插件的作用**:
- 不编译代码，不生成 JAR（仅生成 POM）
- 提供 `constraints { }` DSL 用于声明依赖约束
- 发布时生成 Maven `<dependencyManagement>` 元数据

**第 9-11 行：命名验证**
```kotlin
if (!project.name.startsWith("bom")) {
  throw IllegalStateException("Name of BOM projects must start with 'bom'.")
}
```

**强制命名约定**: 所有 BOM 项目必须以 `bom` 开头，避免误用。

**有效名称**:
- ✓ `bom`
- ✓ `bom-alpha`
- ✓ `bom-experimental`

**无效名称**:
- ✗ `bill-of-materials`
- ✗ `platform`
- ✗ `dependencies`

**第 13-17 行：确保项目评估顺序**
```kotlin
rootProject.subprojects.forEach { subproject ->
  if (!subproject.name.startsWith("bom")) {
    evaluationDependsOn(subproject.path)
  }
}
```

**为什么需要 `evaluationDependsOn`？**

**问题场景**:
```
Gradle 默认并行评估项目配置
↓
BOM 项目在其他项目之前评估完成
↓
afterEvaluate { } 块尝试访问子项目的 publishing.publications
↓
子项目的发布配置尚未初始化
↓
构建失败：NullPointerException
```

**解决方案**:
```kotlin
evaluationDependsOn(subproject.path)
// 强制等待子项目配置完成后，再执行 BOM 的 afterEvaluate 块
```

**评估顺序**:
```
1. 根项目配置
2. BOM 项目调用 evaluationDependsOn → 等待所有子项目
3. 所有非 BOM 子项目配置完成（包括 publishing 配置）
4. BOM 项目的 afterEvaluate 块执行 ← 此时可以安全访问发布配置
```

**第 18 行：创建 OtelBomExtension**
```kotlin
val otelBom = extensions.create<OtelBomExtension>("otelBom")
```

创建名为 `otelBom` 的扩展对象，供 BOM 项目配置。

**第 20-31 行：generateBuildSubstitutions 任务**
```kotlin
val generateBuildSubstitutions by tasks.registering {
  group = "publishing"
  description = "Generate a code snippet that can be copy-pasted for use in composite builds."
}

tasks.named("publish") {
  dependsOn(generateBuildSubstitutions)
}

rootProject.tasks.named(generateBuildSubstitutions.name) {
  dependsOn(generateBuildSubstitutions)
}
```

**任务用途**: 生成复合构建（composite build）的依赖替换代码片段。

**任务依赖关系**:
```
./gradlew publish
    ↓ dependsOn
generateBuildSubstitutions
    ↓ 生成
build/substitutions.gradle.kts（可复制到其他项目）
```

**为什么集成到 publish 任务？**
- 确保每次发布前生成最新的替换文件
- 便于贡献者使用（发布后即可获得替换代码）

**第 33-64 行：项目筛选和约束生成**
```kotlin
afterEvaluate {
  otelBom.projectFilter.finalizeValue()
  val bomProjects = rootProject.subprojects
    .sortedBy { it.findProperty("archivesName") as String? }
    .filter { !it.name.startsWith("bom") }
    .filter(otelBom.projectFilter.get()::test)
    .filter { it.plugins.hasPlugin("maven-publish") }

  generateBuildSubstitutions {
    // ... 生成替换文件
  }
  bomProjects.forEach { project ->
    dependencies {
      constraints {
        api(project)
      }
    }
  }
}
```

**筛选流程**:
```
rootProject.subprojects（所有子项目）
    ↓ sortedBy archivesName
按归档名称排序（确保输出顺序稳定）
    ↓ filter: !name.startsWith("bom")
排除 BOM 项目自身
    ↓ filter: otelBom.projectFilter.get()::test
应用自定义过滤器（stable vs alpha）
    ↓ filter: plugins.hasPlugin("maven-publish")
仅包含可发布的项目
    ↓
bomProjects（最终包含在 BOM 中的项目列表）
```

**依赖约束生成**:
```kotlin
dependencies {
  constraints {
    api(project(":api"))
    api(project(":sdk"))
    api(project(":exporters:otlp"))
    // ... 所有筛选出的项目
  }
}
```

**约束类型**:
- `api`: 约束会传递给使用 BOM 的项目
- `constraints { }`: 不会添加实际依赖，仅提供版本约束

#### OtelBomExtension 扩展配置

**源码**（`io/opentelemetry/gradle/OtelBomExtension.kt`）:
```kotlin
abstract class OtelBomExtension {
  abstract val projectFilter: Property<Predicate<Project>>
}
```

**唯一配置项**: `projectFilter`

**类型**: `Property<Predicate<Project>>`（接受一个项目作为输入，返回 boolean 的函数）

**使用示例**:
```kotlin
// 示例 1: 排除特定模块
otelBom.projectFilter.set { project ->
    !project.name.contains("internal")
}

// 示例 2: 仅包含特定组
otelBom.projectFilter.set { project ->
    project.group == "io.opentelemetry"
}

// 示例 3: 基于属性过滤
otelBom.projectFilter.set { project ->
    project.hasProperty("includeInBom") &&
    project.property("includeInBom") == "true"
}
```

#### 两个 BOM 项目实例

##### 3.8.1 Stable BOM（稳定版）

**文件**: `bom/build.gradle.kts`
```kotlin
plugins {
  id("otel.bom-conventions")
}

description = "OpenTelemetry Bill of Materials"
group = "io.opentelemetry"
base.archivesName.set("opentelemetry-bom")

otelBom.projectFilter.set { !it.hasProperty("otel.release") }
```

**过滤逻辑**:
```kotlin
{ !it.hasProperty("otel.release") }
```

**含义**: 包含所有**没有** `otel.release` 属性的项目（即稳定项目）。

**包含的项目**:
```
✓ api:all (没有 otel.release 属性)
✓ sdk:all (没有 otel.release 属性)
✓ exporters:otlp (没有 otel.release 属性)
✗ api:incubator (有 otel.release = "alpha")
✗ sdk-extensions:incubator (有 otel.release = "alpha")
```

**发布坐标**:
```xml
<groupId>io.opentelemetry</groupId>
<artifactId>opentelemetry-bom</artifactId>
<version>1.35.0</version>
```

##### 3.8.2 Alpha BOM（孵化版）

**文件**: `bom-alpha/build.gradle.kts`
```kotlin
plugins {
  id("otel.bom-conventions")
}

description = "OpenTelemetry Bill of Materials (Alpha)"
group = "io.opentelemetry"
base.archivesName.set("opentelemetry-bom-alpha")

otelBom.projectFilter.set { it.findProperty("otel.release") == "alpha" }

// Required to place dependency on opentelemetry-bom
javaPlatform.allowDependencies()

dependencies {
  // Add dependency on opentelemetry-bom to ensure synchronization between alpha and stable artifacts
  api(platform(project(":bom")))
}
```

**过滤逻辑**:
```kotlin
{ it.findProperty("otel.release") == "alpha" }
```

**含义**: 仅包含 `otel.release` 属性值为 `"alpha"` 的项目。

**包含的项目**:
```
✓ api:incubator (otel.release = "alpha")
✓ sdk-extensions:incubator (otel.release = "alpha")
✗ api:all (没有 otel.release 属性)
✗ sdk:all (没有 otel.release 属性)
```

**继承 Stable BOM**:
```kotlin
javaPlatform.allowDependencies()

dependencies {
  api(platform(project(":bom")))
}
```

**`javaPlatform.allowDependencies()` 的作用**:
- 默认情况下，`java-platform` 项目不能有依赖
- 调用此方法允许 BOM 依赖其他 BOM
- 实现 BOM 继承（alpha BOM 继承 stable BOM）

**继承效果**:
```
opentelemetry-bom-alpha
├── 自身的 alpha 项目约束
│   ├── api:incubator → 1.35.0
│   └── sdk-extensions:incubator → 1.35.0
└── 继承 opentelemetry-bom 的稳定项目约束
    ├── api:all → 1.35.0
    ├── sdk:all → 1.35.0
    └── exporters:otlp → 1.35.0
```

**用户使用 alpha BOM**:
```kotlin
dependencies {
  // ✓ 使用 alpha BOM 会自动包含 stable BOM 的约束
  implementation(platform("io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))

  // ✓ 可以使用稳定 API（来自 stable BOM）
  implementation("io.opentelemetry:opentelemetry-api")

  // ✓ 也可以使用 alpha API
  implementation("io.opentelemetry:opentelemetry-api-incubator")
}
```

**发布坐标**:
```xml
<groupId>io.opentelemetry</groupId>
<artifactId>opentelemetry-bom-alpha</artifactId>
<version>1.35.0</version>
```

#### 如何标记项目为 Alpha？

**方法 1: 在项目的 build.gradle.kts 中设置属性**
```kotlin
// api/incubator/build.gradle.kts
plugins {
  id("otel.java-conventions")
  id("otel.publish-conventions")
}

// ✓ 标记为 alpha
extensions.extraProperties["otel.release"] = "alpha"

description = "OpenTelemetry API Incubator"
```

**方法 2: 在 gradle.properties 中设置**
```properties
# api/incubator/gradle.properties
otel.release=alpha
```

**方法 3: 通过命令行传递**
```bash
./gradlew build -Potel.release=alpha
```

**检查项目属性**:
```bash
# 查看所有项目的 otel.release 属性
./gradlew properties | grep "otel.release"
```

#### BOM 工作原理

##### 1. Gradle 约束解析流程

```
用户项目声明依赖
    ↓
implementation("io.opentelemetry:opentelemetry-api")  # 没有版本
    ↓
Gradle 查找版本约束
    ↓
找到 platform("io.opentelemetry:opentelemetry-bom:1.35.0")
    ↓
读取 BOM 的 <dependencyManagement> 元数据
    ↓
找到 opentelemetry-api 的版本约束: 1.35.0
    ↓
解析为 io.opentelemetry:opentelemetry-api:1.35.0
    ↓
下载 JAR 并添加到 classpath
```

##### 2. 生成的 POM 结构

**opentelemetry-bom.pom**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-bom</artifactId>
  <version>1.35.0</version>
  <packaging>pom</packaging>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
        <version>1.35.0</version>
      </dependency>
      <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-sdk</artifactId>
        <version>1.35.0</version>
      </dependency>
      <!-- ... 50+ 个子项目 -->
    </dependencies>
  </dependencyManagement>
</project>
```

**关键元素**:
- `<packaging>pom</packaging>`: 表明这是一个 POM 项目，不包含 JAR
- `<dependencyManagement>`: 声明依赖版本约束（不是实际依赖）

##### 3. BOM 继承结构

**opentelemetry-bom-alpha.pom**:
```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-bom-alpha</artifactId>
  <version>1.35.0</version>
  <packaging>pom</packaging>

  <dependencyManagement>
    <dependencies>
      <!-- 继承 stable BOM -->
      <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-bom</artifactId>
        <version>1.35.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>

      <!-- Alpha 项目约束 -->
      <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api-incubator</artifactId>
        <version>1.35.0</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

**继承机制**:
```xml
<dependency>
  <artifactId>opentelemetry-bom</artifactId>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

**`<scope>import</scope>` 的作用**:
- 将 `opentelemetry-bom` 的 `<dependencyManagement>` 导入到当前 BOM
- 相当于 Gradle 的 `api(platform(...))`

#### generateBuildSubstitutions 任务详解

##### 任务用途

**问题场景**: 开发者想在本地修改 OpenTelemetry Java SDK，并在自己的项目中测试修改。

**传统方式**（繁琐）:
```bash
# 1. 修改 OpenTelemetry Java 代码
cd /path/to/opentelemetry-java
vim api/all/src/main/java/io/opentelemetry/api/OpenTelemetry.java

# 2. 发布到本地 Maven 仓库
./gradlew publishToMavenLocal

# 3. 在自己的项目中使用 SNAPSHOT 版本
# build.gradle.kts
dependencies {
  implementation("io.opentelemetry:opentelemetry-api:1.36.0-SNAPSHOT")
}

# ⚠️ 问题：每次修改都要重新 publishToMavenLocal
```

**复合构建方式**（高效）:
```bash
# 1. 生成替换文件
cd /path/to/opentelemetry-java
./gradlew generateBuildSubstitutions

# 2. 复制生成的内容到自己的项目
cat bom/build/substitutions.gradle.kts

# 3. 在自己的项目中配置复合构建
# settings.gradle.kts
includeBuild("/path/to/opentelemetry-java") {
  dependencySubstitution {
    # 粘贴生成的替换代码
    substitute(module("io.opentelemetry:opentelemetry-api")).using(project(":api:all"))
    substitute(module("io.opentelemetry:opentelemetry-sdk")).using(project(":sdk:all"))
    # ...
  }
}

# ✓ 好处：修改 OpenTelemetry 代码后，自己的项目会自动使用最新代码
```

##### 生成的文件内容

**bom/build/substitutions.gradle.kts**:
```kotlin
dependencySubstitution {
  substitute(module("io.opentelemetry:opentelemetry-api")).using(project(":api:all"))
  substitute(module("io.opentelemetry:opentelemetry-context")).using(project(":context"))
  substitute(module("io.opentelemetry:opentelemetry-sdk")).using(project(":sdk:all"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-common")).using(project(":sdk:common"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure")).using(project(":sdk-extensions:autoconfigure"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-extension-autoconfigure-spi")).using(project(":sdk-extensions:autoconfigure-spi"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-logs")).using(project(":sdk:logs"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-metrics")).using(project(":sdk:metrics"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-testing")).using(project(":sdk:testing"))
  substitute(module("io.opentelemetry:opentelemetry-sdk-trace")).using(project(":sdk:trace"))
  # ... 50+ 个项目
}
```

**每行的含义**:
```kotlin
substitute(module("io.opentelemetry:opentelemetry-api"))  // Maven 坐标
  .using(project(":api:all"))                             // Gradle 项目路径
```

**替换机制**:
```
用户项目请求 io.opentelemetry:opentelemetry-api:1.35.0
    ↓ Gradle 检测到 dependencySubstitution 规则
替换为 project(":api:all") 的输出
    ↓
直接使用 OpenTelemetry 项目中的源码编译结果
    ↓
无需 publishToMavenLocal，修改即生效
```

##### 使用工作流

**步骤 1: 克隆 OpenTelemetry Java 仓库**
```bash
git clone https://github.com/open-telemetry/opentelemetry-java.git
cd opentelemetry-java
```

**步骤 2: 生成替换文件**
```bash
./gradlew generateBuildSubstitutions

# 输出文件位置:
# - bom/build/substitutions.gradle.kts (stable)
# - bom-alpha/build/substitutions.gradle.kts (alpha)
```

**步骤 3: 在你的项目中配置复合构建**
```kotlin
// settings.gradle.kts
includeBuild("/absolute/path/to/opentelemetry-java") {
  dependencySubstitution {
    // 粘贴 bom/build/substitutions.gradle.kts 的内容
    substitute(module("io.opentelemetry:opentelemetry-api")).using(project(":api:all"))
    substitute(module("io.opentelemetry:opentelemetry-sdk")).using(project(":sdk:all"))
    // ... 更多替换
  }
}
```

**步骤 4: 构建你的项目**
```bash
cd /your/project
./gradlew build

# ✓ Gradle 会自动编译 OpenTelemetry Java 的源码
# ✓ 使用编译结果替代 Maven Central 的 JAR
```

**步骤 5: 修改 OpenTelemetry 代码并测试**
```bash
# 修改 OpenTelemetry 代码
vim /path/to/opentelemetry-java/api/all/src/main/java/...

# 重新构建你的项目（会自动使用新代码）
./gradlew build

# ✓ 无需 publishToMavenLocal
# ✓ 修改即生效
```

#### 使用示例

##### 示例 1: 在外部项目中使用 Stable BOM

**build.gradle.kts**:
```kotlin
plugins {
  id("java")
}

dependencies {
  // ✓ 导入 BOM，统一管理 OpenTelemetry 版本
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))

  // ✓ 不需要指定版本（由 BOM 管理）
  implementation("io.opentelemetry:opentelemetry-api")
  implementation("io.opentelemetry:opentelemetry-sdk")
  implementation("io.opentelemetry:opentelemetry-exporter-otlp")

  // ✓ 也可以显式指定版本（覆盖 BOM）
  implementation("io.opentelemetry:opentelemetry-exporter-prometheus:1.34.0")
}
```

**Maven 等效配置**:
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
    <!-- 版本由 BOM 管理 -->
  </dependency>
</dependencies>
```

##### 示例 2: 在外部项目中使用 Alpha BOM

**build.gradle.kts**:
```kotlin
dependencies {
  // ✓ 使用 alpha BOM（自动包含 stable BOM）
  implementation(platform("io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))

  // ✓ 使用稳定 API
  implementation("io.opentelemetry:opentelemetry-api")
  implementation("io.opentelemetry:opentelemetry-sdk")

  // ✓ 使用 alpha API
  implementation("io.opentelemetry:opentelemetry-api-incubator")
  implementation("io.opentelemetry:opentelemetry-sdk-extension-incubator")
}
```

##### 示例 3: 同时使用多个 BOM

**build.gradle.kts**:
```kotlin
dependencies {
  // ✓ Gradle 支持多个 BOM
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
  implementation(platform("io.micrometer:micrometer-bom:1.12.0"))
  implementation(platform("com.google.cloud:libraries-bom:26.32.0"))

  // ✓ 所有依赖的版本由对应的 BOM 管理
  implementation("io.opentelemetry:opentelemetry-api")
  implementation("io.micrometer:micrometer-core")
  implementation("com.google.cloud:google-cloud-storage")
}
```

**版本冲突处理**:
```kotlin
dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
  implementation(platform("other-bom-with-otel:other-bom:1.0.0"))

  // ⚠️ 如果两个 BOM 都约束了 opentelemetry-api，Gradle 使用第一个
  implementation("io.opentelemetry:opentelemetry-api")
  // → 版本来自 opentelemetry-bom（1.35.0）

  // ✓ 可以显式指定版本覆盖
  implementation("io.opentelemetry:opentelemetry-api:1.36.0")
}
```

##### 示例 4: 创建新的 BOM 项目

**场景**: 为特定用例创建自定义 BOM（如仅包含 exporters）。

**exporters-bom/build.gradle.kts**:
```kotlin
plugins {
  id("otel.bom-conventions")
}

description = "OpenTelemetry Exporters Bill of Materials"
group = "io.opentelemetry"
base.archivesName.set("opentelemetry-exporters-bom")

// ✓ 自定义过滤器：仅包含 exporters 模块
otelBom.projectFilter.set { project ->
  project.path.startsWith(":exporters") &&
  !project.hasProperty("otel.release")
}
```

**生成的 BOM 包含**:
```
✓ exporters:otlp
✓ exporters:prometheus
✓ exporters:zipkin
✓ exporters:jaeger
✗ api:all (不在 exporters 目录下)
✗ sdk:all (不在 exporters 目录下)
```

#### 项目筛选逻辑详解

**筛选流程可视化**:
```
rootProject.subprojects（80+ 个项目）
    │
    ↓ sortedBy { it.findProperty("archivesName") as String? }
    │ 按归档名称排序
    │ - opentelemetry-api
    │ - opentelemetry-context
    │ - opentelemetry-exporter-otlp
    │ - ...
    │
    ↓ filter { !it.name.startsWith("bom") }
    │ 排除 BOM 项目
    │ ✗ bom
    │ ✗ bom-alpha
    │
    ↓ filter(otelBom.projectFilter.get()::test)
    │ 应用自定义过滤器
    │
    │ Stable BOM: !it.hasProperty("otel.release")
    │ ✓ api:all (无 otel.release 属性)
    │ ✗ api:incubator (有 otel.release = "alpha")
    │
    │ Alpha BOM: it.findProperty("otel.release") == "alpha"
    │ ✗ api:all (无 otel.release 属性)
    │ ✓ api:incubator (有 otel.release = "alpha")
    │
    ↓ filter { it.plugins.hasPlugin("maven-publish") }
    │ 仅包含可发布的项目
    │ ✓ api:all (有 maven-publish 插件)
    │ ✗ testing-internal (无 maven-publish 插件)
    │
    ↓
bomProjects（最终列表，~50 个项目）
```

**为什么需要排序？**
```kotlin
.sortedBy { it.findProperty("archivesName") as String? }
```

**原因**:
- 确保生成的 BOM 和替换文件顺序稳定
- 避免 Git diff 中出现无意义的顺序变化
- 便于人工审查（按字母顺序）

**示例**:
```
排序前（项目评估顺序随机）:
- sdk:all
- api:all
- exporters:otlp
- context

排序后（按 archivesName）:
- opentelemetry-api (api:all)
- opentelemetry-context (context)
- opentelemetry-exporter-otlp (exporters:otlp)
- opentelemetry-sdk (sdk:all)
```

#### 发布流程

##### 1. 与 otel.publish-conventions 的集成

**插件继承链**:
```
otel.bom-conventions
    ↓ applies
otel.publish-conventions
    ↓ applies
otel.japicmp-conventions
```

**otel.publish-conventions 提供**:
- Maven 发布配置
- POM 元数据（许可证、开发者、SCM）
- GPG 签名（CI 环境）
- Nexus Publish 插件集成

##### 2. 版本映射策略

**otel.publish-conventions.gradle.kts** (相关部分):
```kotlin
publishing {
  publications {
    register<MavenPublication>("mavenPublication") {
      plugins.withId("java-platform") {
        from(components["javaPlatform"])
      }

      versionMapping {
        allVariants {
          fromResolutionResult()
        }
      }
    }
  }
}
```

**`fromResolutionResult()` 的作用**:
- 使用实际解析的版本（而非声明的版本）
- 处理版本冲突和约束
- 确保 BOM 中的版本与实际构建一致

**示例**:
```
项目声明: api(project(":api:all"))
    ↓ Gradle 解析
实际版本: io.opentelemetry:opentelemetry-api:1.35.0
    ↓ versionMapping
BOM 中的约束: <version>1.35.0</version>
```

##### 3. 发布到 Maven Central

**发布命令**:
```bash
# 发布到 Sonatype staging 仓库
./gradlew publish

# 发布并自动 close/release 到 Maven Central
./gradlew publishToSonatype closeAndReleaseSonatypeStagingRepository
```

**发布的文件**:
```
Maven Central 上的文件:
- opentelemetry-bom-1.35.0.pom (BOM 元数据)
- opentelemetry-bom-1.35.0.pom.asc (GPG 签名)
- opentelemetry-bom-1.35.0.module (Gradle 模块元数据)
```

**没有 JAR 文件**: BOM 是纯 POM 项目，不包含代码。

##### 4. 生成的 POM 元数据

**完整的 opentelemetry-bom.pom**（简化版）:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-bom</artifactId>
  <version>1.35.0</version>
  <packaging>pom</packaging>
  <name>OpenTelemetry Java</name>
  <description>OpenTelemetry Bill of Materials</description>
  <url>https://github.com/open-telemetry/opentelemetry-java</url>

  <licenses>
    <license>
      <name>The Apache License, Version 2.0</name>
      <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
    </license>
  </licenses>

  <developers>
    <developer>
      <id>opentelemetry</id>
      <name>OpenTelemetry</name>
      <url>https://github.com/open-telemetry/community</url>
    </developer>
  </developers>

  <scm>
    <connection>scm:git:git@github.com:open-telemetry/opentelemetry-java.git</connection>
    <developerConnection>scm:git:git@github.com:open-telemetry/opentelemetry-java.git</developerConnection>
    <url>git@github.com:open-telemetry/opentelemetry-java.git</url>
  </scm>

  <dependencyManagement>
    <dependencies>
      <!-- 50+ 个子项目 -->
    </dependencies>
  </dependencyManagement>
</project>
```

#### 调试和故障排查

##### 1. 查看 BOM 包含的所有项目

**方法 1: 查看依赖树**
```bash
./gradlew :bom:dependencies --configuration apiElements

# 输出示例:
# apiElements - API elements for the 'main' feature.
# \--- io.opentelemetry:opentelemetry-api:1.35.0
# \--- io.opentelemetry:opentelemetry-sdk:1.35.0
# \--- ...
```

**方法 2: 查看约束**
```bash
./gradlew :bom:dependencies --configuration apiElements | grep "^+---"
```

**方法 3: 检查发布的 POM**
```bash
# 下载发布的 POM
curl -o bom.pom https://repo1.maven.org/maven2/io/opentelemetry/opentelemetry-bom/1.35.0/opentelemetry-bom-1.35.0.pom

# 查看依赖管理部分
xmllint --format bom.pom | grep -A 5 "<dependencyManagement>"
```

##### 2. 生成替换文件

**命令**:
```bash
# 生成所有 BOM 的替换文件
./gradlew generateBuildSubstitutions

# 查看生成的文件
cat bom/build/substitutions.gradle.kts
cat bom-alpha/build/substitutions.gradle.kts
```

**验证替换文件**:
```bash
# 统计替换条目数量
wc -l bom/build/substitutions.gradle.kts

# 检查特定项目是否包含
grep "opentelemetry-api" bom/build/substitutions.gradle.kts
```

##### 3. 测试 BOM 约束

**创建测试项目**:
```kotlin
// test-bom/build.gradle.kts
plugins {
  id("java")
}

dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
  implementation("io.opentelemetry:opentelemetry-api")
}

tasks.register("printApiVersion") {
  doLast {
    configurations.runtimeClasspath.get().resolvedConfiguration
      .firstLevelModuleDependencies
      .forEach { dep ->
        if (dep.moduleGroup == "io.opentelemetry") {
          println("${dep.moduleName} → ${dep.moduleVersion}")
        }
      }
  }
}
```

**验证版本**:
```bash
./gradlew printApiVersion

# 输出:
# opentelemetry-api → 1.35.0
# opentelemetry-context → 1.35.0
```

##### 4. 检查项目是否被包含在 BOM 中

**问题**: 为什么我的项目没有包含在 BOM 中？

**诊断步骤**:

**1. 检查项目名称**
```bash
# 项目名称不能以 "bom" 开头
./gradlew projects | grep "^+--- "
```

**2. 检查 maven-publish 插件**
```bash
# 项目必须应用 maven-publish 插件
./gradlew :your-project:plugins | grep "maven-publish"
```

**3. 检查 otel.release 属性**
```bash
# Stable BOM: 项目不应有 otel.release 属性
# Alpha BOM: 项目必须有 otel.release = "alpha"
./gradlew :your-project:properties | grep "otel.release"
```

**4. 检查过滤器逻辑**
```kotlin
// 在 bom/build.gradle.kts 中添加调试输出
otelBom.projectFilter.set { project ->
  val included = !project.hasProperty("otel.release")
  if (!included) {
    println("Excluding ${project.path} from stable BOM")
  }
  included
}
```

##### 5. 常见错误

**错误 1: BOM 项目名称不符合规范**
```
FAILURE: Build failed with an exception.

* What went wrong:
Name of BOM projects must start with 'bom'.
```

**解决**: 重命名项目或修改 `settings.gradle.kts` 中的项目名称。

**错误 2: 循环依赖**
```
Circular dependency between the following tasks:
:bom:generateBuildSubstitutions
\--- :bom:publish
     \--- :bom:generateBuildSubstitutions (*)
```

**解决**: 这是设计行为，`publish` 依赖 `generateBuildSubstitutions`，确保发布前生成最新的替换文件。

**错误 3: 复合构建中的版本冲突**
```
Could not resolve io.opentelemetry:opentelemetry-api:1.35.0.
  Required by:
      project :my-app
  Substituted by:
      project :api:all
```

**解决**: 确保 OpenTelemetry 项目的版本与你项目声明的版本一致。

#### 最佳实践

##### 1. 何时创建新的 BOM

**应该创建新 BOM**:
- ✓ 为不同成熟度级别（stable、alpha、beta）
- ✓ 为不同用例（exporters-only、instrumentation-only）
- ✓ 为跨项目的版本对齐（如 OpenTelemetry + Micrometer）

**不应该创建新 BOM**:
- ✗ 为单个模块或小组件
- ✗ 为内部工具或测试项目
- ✗ 为临时或实验性功能

##### 2. 标记 Alpha 项目的策略

**推荐方式**: 在项目的 `build.gradle.kts` 中设置
```kotlin
// api/incubator/build.gradle.kts
plugins {
  id("otel.java-conventions")
  id("otel.publish-conventions")
}

// ✓ 明确标记
extensions.extraProperties["otel.release"] = "alpha"

description = "OpenTelemetry API Incubator"
otelJava.moduleName.set("io.opentelemetry.api.incubator")
```

**优势**:
- 版本控制跟踪
- 与代码关联
- 易于审查

##### 3. 本地测试 BOM 变更

**步骤 1: 发布到本地 Maven 仓库**
```bash
./gradlew :bom:publishToMavenLocal

# BOM 发布到: ~/.m2/repository/io/opentelemetry/opentelemetry-bom/
```

**步骤 2: 在测试项目中使用**
```kotlin
// test-project/settings.gradle.kts
repositories {
  mavenLocal()  // ✓ 优先使用本地仓库
  mavenCentral()
}

// test-project/build.gradle.kts
dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.36.0-SNAPSHOT"))
  implementation("io.opentelemetry:opentelemetry-api")
}
```

**步骤 3: 验证**
```bash
cd test-project
./gradlew dependencies --configuration runtimeClasspath | grep "opentelemetry"
```

##### 4. 版本管理策略

**推荐**: 使用 BOM 统一管理所有 OpenTelemetry 依赖
```kotlin
// ✓ 推荐：使用 BOM
dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
  implementation("io.opentelemetry:opentelemetry-api")
  implementation("io.opentelemetry:opentelemetry-sdk")
}

// ✗ 不推荐：手动管理版本
dependencies {
  implementation("io.opentelemetry:opentelemetry-api:1.35.0")
  implementation("io.opentelemetry:opentelemetry-sdk:1.35.0")
}
```

**原因**:
- ✅ 保证版本一致性
- ✅ 简化升级（只需更改 BOM 版本）
- ✅ 避免版本冲突
- ✅ 遵循 OpenTelemetry 官方推荐

##### 5. Alpha API 使用指南

**场景**: 想使用 alpha API，但不想引入所有 alpha 依赖。

**方案 1: 使用 alpha BOM（推荐）**
```kotlin
dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))

  // ✓ 使用 alpha API
  implementation("io.opentelemetry:opentelemetry-api-incubator")

  // ✓ 稳定 API 也可用（继承自 stable BOM）
  implementation("io.opentelemetry:opentelemetry-api")
}
```

**方案 2: 混合使用两个 BOM**
```kotlin
dependencies {
  implementation(platform("io.opentelemetry:opentelemetry-bom:1.35.0"))
  implementation(platform("io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))

  // ✓ 两个 BOM 的依赖都可用
  implementation("io.opentelemetry:opentelemetry-api")
  implementation("io.opentelemetry:opentelemetry-api-incubator")
}
```

**注意**: Alpha API 可能在未来版本中发生破坏性变更，谨慎用于生产环境。

#### 架构图

##### BOM 依赖层次结构

```
┌─────────────────────────────────────────────────────┐
│ User Project (外部项目)                              │
│                                                     │
│ dependencies {                                      │
│   implementation(platform(                          │
│     "io.opentelemetry:opentelemetry-bom-alpha:1.35.0"))│
│ }                                                   │
└─────────────────────┬───────────────────────────────┘
                      │ 导入 BOM
                      ↓
┌─────────────────────────────────────────────────────┐
│ opentelemetry-bom-alpha (Alpha BOM)                 │
│                                                     │
│ - API Incubator: 1.35.0                            │
│ - SDK Extensions Incubator: 1.35.0                 │
│                                                     │
│ dependencies {                                      │
│   api(platform(project(":bom")))  ← 继承           │
│ }                                                   │
└─────────────────────┬───────────────────────────────┘
                      │ 继承约束
                      ↓
┌─────────────────────────────────────────────────────┐
│ opentelemetry-bom (Stable BOM)                      │
│                                                     │
│ - API: 1.35.0                                       │
│ - SDK: 1.35.0                                       │
│ - Context: 1.35.0                                   │
│ - Exporters OTLP: 1.35.0                            │
│ - ... (50+ 个稳定模块)                               │
└─────────────────────────────────────────────────────┘
```

##### 约束解析流程

```
用户声明依赖
    │
    ↓
implementation("io.opentelemetry:opentelemetry-api")
    │ (没有版本)
    ↓
Gradle 查找版本约束
    │
    ├─ 检查 BOM 约束
    │  └─ opentelemetry-bom-alpha
    │     └─ 继承自 opentelemetry-bom
    │        └─ 找到: opentelemetry-api → 1.35.0
    │
    ├─ 检查显式版本声明 (无)
    │
    ├─ 检查依赖版本冲突 (无)
    │
    ↓
解析为: io.opentelemetry:opentelemetry-api:1.35.0
    │
    ↓
下载 JAR 并添加到 classpath
```

##### 复合构建架构

```
┌─────────────────────────────────────────────────────┐
│ User Project (你的项目)                              │
│                                                     │
│ settings.gradle.kts:                                │
│   includeBuild("/path/to/opentelemetry-java") {    │
│     dependencySubstitution { ... }                  │
│   }                                                 │
│                                                     │
│ build.gradle.kts:                                   │
│   dependencies {                                    │
│     implementation(                                 │
│       "io.opentelemetry:opentelemetry-api:1.35.0")│
│   }                                                 │
└─────────────────────┬───────────────────────────────┘
                      │
                      ↓ Gradle 应用 dependencySubstitution
┌─────────────────────────────────────────────────────┐
│ Dependency Substitution 规则                         │
│                                                     │
│ substitute(module(                                  │
│   "io.opentelemetry:opentelemetry-api"))           │
│   .using(project(":api:all"))                       │
└─────────────────────┬───────────────────────────────┘
                      │
                      ↓ 使用本地项目
┌─────────────────────────────────────────────────────┐
│ OpenTelemetry Java (包含的构建)                      │
│                                                     │
│ :api:all                                            │
│   └─ 编译源码                                        │
│      └─ 生成 JAR                                    │
│         └─ 提供给 User Project                       │
└─────────────────────────────────────────────────────┘
```

#### 总结

**BOM 约定插件的核心价值**:
- ✅ **自动化版本管理**: 自动收集所有可发布项目并生成约束
- ✅ **分层架构**: 支持 stable/alpha 等多级 BOM
- ✅ **开发者友好**: 生成复合构建替换文件，简化本地开发
- ✅ **灵活过滤**: 通过 `projectFilter` 支持自定义项目筛选逻辑
- ✅ **发布集成**: 与 Maven Central 发布流程无缝集成
- ✅ **标准兼容**: 生成标准的 Maven BOM，兼容 Maven 和 Gradle

**适用场景**:
- 管理多模块项目的版本一致性
- 简化用户的依赖声明
- 支持 alpha/beta 等不同成熟度 API
- 本地开发和测试（通过复合构建）

**OpenTelemetry Java 使用 BOM 的优势**:
- 用户只需指定一次 BOM 版本
- 保证 50+ 个模块的版本一致性
- 支持稳定 API 和实验性 API 的分离
- 简化项目升级（只需更改 BOM 版本）
- 贡献者可以通过复合构建快速测试变更

---

