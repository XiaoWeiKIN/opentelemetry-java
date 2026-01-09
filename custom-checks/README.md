# OpenTelemetry Java Custom Checks 技术文档

## 📑 目录

- [概述](#概述)
- [🚀 快速开始](#-快速开始)
- [1. 目录结构](#1-目录结构)
- [2. 核心概念](#2-核心概念)
  - [2.1 ErrorProne 插件机制](#21-errorprone-插件机制)
  - [2.2 SPI (Service Provider Interface)](#22-spi-service-provider-interface)
  - [2.3 BugChecker 接口](#23-bugchecker-接口)
  - [2.4 Java Compiler API](#24-java-compiler-api)
- [3. 自定义检查详解](#3-自定义检查详解)
  - [3.1 OtelInternalJavadoc](#31-otelinternaljavadoc)
  - [3.2 OtelPrivateConstructorForUtilityClass](#32-otelprivateconstructorforutilityclass)
- [4. 构建配置详解](#4-构建配置详解)
- [5. 测试](#5-测试)
- [6. 集成机制](#6-集成机制)
- [7. 使用指南](#7-使用指南)
- [8. 故障排查](#8-故障排查)
- [9. 开发指南](#9-开发指南)
- [10. 架构和设计模式](#10-架构和设计模式)
- [11. 最佳实践](#11-最佳实践)
- [12. 扩展阅读](#12-扩展阅读)

---

## 概述

**custom-checks** 是 OpenTelemetry Java 项目的 ErrorProne 自定义检查插件模块,提供项目特定的静态代码分析规则。

**核心价值**:
- ✅ **编译时保证**: 在开发阶段捕获 API 文档缺失和代码风格问题
- ✅ **自动化检查**: 无需人工审查,自动应用到所有子项目
- ✅ **轻量集成**: 通过 SPI 机制与 ErrorProne 无缝集成
- ✅ **可扩展**: 易于添加新的自定义检查规则

**项目统计**:
- **检查器数量**: 2 个自定义检查
- **源文件数量**: 5 个（2 个检查器 + 1 个测试 + 1 个 SPI 配置 + 1 个构建脚本）
- **代码量**: ~200 行 Java 代码
- **Java 版本要求**: Java 21（编译和测试）

---

## 🚀 快速开始

### 常用命令

```bash
# 编译 custom-checks 模块
./gradlew :custom-checks:build

# 运行测试
./gradlew :custom-checks:test

# 编译任意子项目（自动应用检查）
./gradlew :api:compileJava

# 查看检查报告
cat build/reports/errorprone/main.txt

# 跳过 custom-checks（临时禁用）
./gradlew build -x :custom-checks:build
```

### 快速参考卡片

| 检查器 | 用途 | 严重级别 | 目标 |
|--------|------|----------|------|
| `OtelInternalJavadoc` | 强制内部 API 文档免责声明 | WARNING | `internal` 包中的公共类 |
| `OtelPrivateConstructorForUtilityClass` | 确保工具类有私有构造函数 | WARNING | 静态工具类 |

### 如何抑制警告

```java
// 方法 1: 使用 @SuppressWarnings 注解
@SuppressWarnings("OtelInternalJavadoc")
public class MyInternalClass {
  // ...
}

// 方法 2: 在 build.gradle.kts 中禁用
tasks.withType<JavaCompile>().configureEach {
  options.errorprone.disable("OtelInternalJavadoc")
}
```

---

## 1. 目录结构

```
custom-checks/
├── build.gradle.kts                    # 构建配置（88 行）
├── build/                              # 编译输出目录
│   ├── classes/java/main/              # 编译后的 class 文件
│   ├── libs/custom-checks.jar          # 打包后的 JAR（包含 META-INF/services）
│   └── reports/                        # 测试报告
├── src/main/java/io/opentelemetry/gradle/customchecks/
│   ├── OtelInternalJavadoc.java       # 内部 API 文档检查（81 行）
│   └── OtelPrivateConstructorForUtilityClass.java  # 工具类构造函数检查（38 行）
├── src/main/resources/META-INF/services/
│   └── com.google.errorprone.bugpatterns.BugChecker  # SPI 注册（2 行）
└── src/test/java/io/opentelemetry/gradle/customchecks/
    └── OtelInternalJavadocTest.java    # 测试（57 行）
```

**关键文件说明**:

| 文件 | 作用 | 行数 |
|------|------|------|
| `OtelInternalJavadoc.java` | 核心检查器：验证内部类的 javadoc 免责声明 | 81 |
| `OtelPrivateConstructorForUtilityClass.java` | 包装检查器：委托给 ErrorProne 内置检查 | 38 |
| `META-INF/services/...BugChecker` | SPI 描述符：注册检查器到 ErrorProne | 2 |
| `OtelInternalJavadocTest.java` | 测试：正面和负面测试用例 | 57 |
| `build.gradle.kts` | 构建配置：Java 21、--add-exports 等 | 88 |

---

## 2. 核心概念

### 2.1 ErrorProne 插件机制

**ErrorProne** 是 Google 开发的 Java 编译器插件,在编译时执行静态代码分析。

**工作原理**:
```
源代码编译
    ↓
javac 生成抽象语法树（AST）
    ↓
ErrorProne 插件介入
    ↓
加载所有 BugChecker 实现（通过 SPI）
    ↓
遍历 AST 节点
    ↓
每个 BugChecker 检查匹配的节点
    ↓
收集发现的问题（Description）
    ↓
报告警告/错误
    ↓
编译继续或失败（取决于严重级别）
```

**优势**:
- ✅ 编译时检查,无额外构建步骤
- ✅ 集成到 IDE（实时反馈）
- ✅ 可配置严重级别（ERROR、WARNING、SUGGESTION）
- ✅ 支持自动修复（SuggestedFix）

**ErrorProne 在 OpenTelemetry 中的角色**:
- buildSrc 通过 `otel.errorprone-conventions` 插件统一配置
- custom-checks 作为额外的 errorprone 依赖被引入
- 编译每个子项目时自动应用所有检查

### 2.2 SPI (Service Provider Interface)

**SPI** 是 Java 标准的插件机制,允许在运行时发现和加载接口实现。

**工作流程**:
```
1. 定义服务接口
   com.google.errorprone.bugpatterns.BugChecker

2. 创建服务实现
   io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
   io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass

3. 注册服务实现（META-INF/services）
   META-INF/services/com.google.errorprone.bugpatterns.BugChecker
   ↓ 文件内容：
   io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
   io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass

4. 服务加载器发现实现
   ServiceLoader.load(BugChecker.class)
   ↓ 读取 JAR 中的 META-INF/services 文件
   ↓ 加载并实例化所有实现类

5. 调用服务实现
   ErrorProne 调用每个 BugChecker 的 matchXxx() 方法
```

**SPI 的优势**:
- ✅ 松耦合：检查器和 ErrorProne 核心解耦
- ✅ 可扩展：添加新检查器无需修改 ErrorProne 代码
- ✅ 标准化：遵循 Java 标准插件机制

**custom-checks 的 SPI 配置**:

**文件**: `src/main/resources/META-INF/services/com.google.errorprone.bugpatterns.BugChecker`
```
io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass
```

**打包后的结构**:
```
custom-checks.jar
├── io/opentelemetry/gradle/customchecks/
│   ├── OtelInternalJavadoc.class
│   └── OtelPrivateConstructorForUtilityClass.class
└── META-INF/services/
    └── com.google.errorprone.bugpatterns.BugChecker
```

### 2.3 BugChecker 接口

**BugChecker** 是 ErrorProne 的核心接口,所有自定义检查器必须继承此类并实现匹配器接口。

**基本结构**:
```java
@BugPattern(
    name = "MyCheck",           // 检查器名称（用于 @SuppressWarnings）
    summary = "检查描述",        // 简短描述
    severity = WARNING          // 严重级别（ERROR/WARNING/SUGGESTION）
)
public class MyCheck extends BugChecker
    implements BugChecker.ClassTreeMatcher {  // 匹配器接口

    @Override
    public Description matchClass(ClassTree tree, VisitorState state) {
        // 检查逻辑
        if (发现问题) {
            return describeMatch(tree);  // 报告问题
        }
        return Description.NO_MATCH;     // 没问题
    }
}
```

**常用匹配器接口**:

| 接口 | 匹配目标 | 用途 |
|------|----------|------|
| `ClassTreeMatcher` | 类声明 | 检查类结构、修饰符、javadoc |
| `MethodTreeMatcher` | 方法声明 | 检查方法签名、返回类型 |
| `VariableTreeMatcher` | 变量声明 | 检查变量命名、初始化 |
| `ImportTreeMatcher` | import 语句 | 检查导入规范 |
| `AnnotationTreeMatcher` | 注解使用 | 检查注解配置 |

**Description 对象**:
- `Description.NO_MATCH`: 没有发现问题
- `describeMatch(tree)`: 报告问题（使用 @BugPattern 的 summary）
- `describeMatch(tree, suggestedFix)`: 报告问题并提供自动修复

**VisitorState 对象**:
- 提供当前编译上下文
- 访问 AST 路径：`state.getPath()`
- 获取源码文本：`state.getSourceForNode(node)`
- 访问 javac 内部 API：`state.context`

### 2.4 Java Compiler API

**Java Compiler API** (`com.sun.tools.javac`) 是 JDK 内部 API,提供编译器功能的编程访问。

**custom-checks 使用的 API**:

#### JavacTrees API
用于访问 javadoc 注释:

```java
DocCommentTree docCommentTree =
    JavacTrees.instance(state.context).getDocCommentTree(state.getPath());
```

**为什么需要 JavacTrees**:
- AST 不包含注释（包括 javadoc）
- javadoc 存储在单独的数据结构中
- 必须通过 `JavacTrees` API 访问

#### 访问的内部包

**custom-checks 需要访问的 javac 包**:
```
com.sun.tools.javac.api       # JavacTrees、JavacScope 等
com.sun.tools.javac.code      # 符号表、类型系统
com.sun.tools.javac.comp      # 编译器组件
com.sun.tools.javac.tree      # AST 节点定义
com.sun.tools.javac.util      # 工具类
```

**为什么需要 `--add-exports`**:
- Java 9+ 引入模块系统（JPMS）
- `com.sun.tools.javac` 包属于 `jdk.compiler` 模块
- 默认情况下这些包是封装的（不可访问）
- 必须使用 `--add-exports` 显式导出

**编译时配置**:
```kotlin
tasks.withType<JavaCompile>().configureEach {
  options.compilerArgs.addAll(listOf(
    "--add-exports", "jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED",
    "--add-exports", "jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED",
    "--add-exports", "jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED",
    "--add-exports", "jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED",
    "--add-exports", "jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED"
  ))
}
```

**测试时配置**:
```kotlin
tasks.withType<Test>().configureEach {
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED")
  // ... 更多包
}
```

**`--add-exports` vs `--add-opens`**:

| 参数 | 作用 | 权限 | 用途 |
|------|------|------|------|
| `--add-exports` | 导出包 | 允许访问公共类型 | 编译时访问 |
| `--add-opens` | 打开包 | 允许反射访问私有成员 | 运行时反射 |

---

## 3. 自定义检查详解

### 3.1 OtelInternalJavadoc

**文件路径**: `src/main/java/io/opentelemetry/gradle/customchecks/OtelInternalJavadoc.java:1`

#### 目的和动机

**问题**: OpenTelemetry Java SDK 包含大量内部 API（位于 `*.internal.*` 包中）,这些 API 不保证向后兼容,但由于技术限制必须声明为 `public`（跨包访问需要）。

**风险**:
- 用户可能误用内部 API
- 内部 API 变更导致用户代码损坏
- 缺乏明确的"内部 API"警告

**解决方案**: 强制要求所有内部包中的公共类必须包含 javadoc 免责声明,明确告知使用者 API 的不稳定性。

**检查目标**:
- ✓ 公共类（`public class`）
- ✓ 包名包含 `"internal"`
- ✗ 测试类（类名以 `"Test"` 结尾）

#### 允许的免责声明

**版本 1** (标准免责声明):
```java
/**
 * This class is internal and is hence not for public use.
 * Its APIs are unstable and can change at any time.
 */
```

**版本 2** (实验性免责声明):
```java
/**
 * This class is internal and experimental. Its APIs are unstable and can change at any time.
 * Its APIs (or a version of them) may be promoted to the public stable API in the future,
 * but no guarantees are made.
 */
```

#### 完整源码分析

**文件**: `OtelInternalJavadoc.java` (81 行)

```java
package io.opentelemetry.gradle.customchecks;

import static com.google.errorprone.BugPattern.SeverityLevel.WARNING;

import com.google.errorprone.BugPattern;
import com.google.errorprone.VisitorState;
import com.google.errorprone.bugpatterns.BugChecker;
import com.google.errorprone.matchers.Description;
import com.sun.source.doctree.DocCommentTree;
import com.sun.source.tree.ClassTree;
import com.sun.source.tree.PackageTree;
import com.sun.tools.javac.api.JavacTrees;
import java.util.regex.Pattern;
import javax.annotation.Nullable;
import javax.lang.model.element.Modifier;
```

**导入说明**:
- `BugPattern`, `BugChecker`: ErrorProne 核心类
- `ClassTree`, `PackageTree`: AST 节点类型
- `DocCommentTree`: javadoc 注释树
- `JavacTrees`: 访问 javadoc 的 API
- `Pattern`: 正则表达式匹配

**@BugPattern 注解**:
```java
@BugPattern(
    summary =
        "This public internal class doesn't end with any of the applicable javadoc disclaimers: \""
            + OtelInternalJavadoc.EXPECTED_INTERNAL_COMMENT_V1
            + "\", or \""
            + OtelInternalJavadoc.EXPECTED_INTERNAL_COMMENT_V2
            + "\"",
    severity = WARNING)
```

**解析**:
- `summary`: 问题描述（显示在编译输出和 IDE 中）
- `severity = WARNING`: 警告级别（不会导致编译失败）

**类定义**:
```java
public class OtelInternalJavadoc extends BugChecker implements BugChecker.ClassTreeMatcher {

  private static final long serialVersionUID = 1L;

  private static final Pattern INTERNAL_PACKAGE_PATTERN = Pattern.compile("\\binternal\\b");
```

**解析**:
- 继承 `BugChecker` 并实现 `ClassTreeMatcher`
- `INTERNAL_PACKAGE_PATTERN`: 匹配包含 "internal" 单词边界的包名
- `\b` 确保 "internal" 是完整的单词（不匹配 "internationalization"）

**免责声明常量**:
```java
  static final String EXPECTED_INTERNAL_COMMENT_V1 =
      "This class is internal and is hence not for public use."
          + " Its APIs are unstable and can change at any time.";

  static final String EXPECTED_INTERNAL_COMMENT_V2 =
      "This class is internal and experimental. Its APIs are unstable and can change at any time."
          + " Its APIs (or a version of them) may be promoted to the public stable API in the"
          + " future, but no guarantees are made.";
```

**核心检查逻辑 - matchClass()**:
```java
  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 第 1 步：快速过滤
    if (!isPublic(tree) || !isInternal(state) || tree.getSimpleName().toString().endsWith("Test")) {
      return Description.NO_MATCH;
    }

    // 第 2 步：获取 javadoc
    String javadoc = getJavadoc(state);

    // 第 3 步：验证免责声明
    if (javadoc != null
        && (javadoc.contains(EXPECTED_INTERNAL_COMMENT_V1)
            || javadoc.contains(EXPECTED_INTERNAL_COMMENT_V2))) {
      return Description.NO_MATCH;
    }

    // 第 4 步：报告违规
    return describeMatch(tree);
  }
```

**逻辑流程**:
```
遍历类节点
    ↓
是否为公共类? No → 跳过
    ↓ Yes
包名是否包含 "internal"? No → 跳过
    ↓ Yes
类名是否以 "Test" 结尾? Yes → 跳过
    ↓ No
获取 javadoc（使用 JavacTrees API）
    ↓
javadoc 是否存在? No → 报告违规
    ↓ Yes
规范化 javadoc 文本（去除格式）
    ↓
是否包含允许的免责声明? No → 报告违规
    ↓ Yes
通过检查
```

**辅助方法 - isPublic()**:
```java
  private static boolean isPublic(ClassTree tree) {
    return tree.getModifiers().getFlags().contains(Modifier.PUBLIC);
  }
```

**作用**: 检查类是否有 `public` 修饰符。

**辅助方法 - isInternal()**:
```java
  private static boolean isInternal(VisitorState state) {
    PackageTree packageTree = state.getPath().getCompilationUnit().getPackage();
    if (packageTree == null) {
      return false;
    }
    String packageName = state.getSourceForNode(packageTree.getPackageName());
    return packageName != null && INTERNAL_PACKAGE_PATTERN.matcher(packageName).find();
  }
```

**解析**:
1. 获取当前编译单元的包声明
2. 提取包名的源码文本
3. 使用正则表达式检查是否包含 "internal"

**匹配示例**:
```
✓ io.opentelemetry.api.internal
✓ io.opentelemetry.sdk.trace.internal.data
✓ io.opentelemetry.internal.utils
✗ io.opentelemetry.api
✗ io.opentelemetry.api.internationalization (international != internal)
```

**辅助方法 - getJavadoc()**:
```java
  @Nullable
  private static String getJavadoc(VisitorState state) {
    DocCommentTree docCommentTree =
        JavacTrees.instance(state.context).getDocCommentTree(state.getPath());
    if (docCommentTree == null) {
      return null;
    }
    return docCommentTree.toString().replace("\n", " ").replace(" * ", " ").replaceAll("\\s+", " ");
  }
```

**解析**:
1. 使用 `JavacTrees.instance()` 获取 javadoc 树
2. 如果没有 javadoc,返回 `null`
3. 规范化文本：
   - 将换行符替换为空格
   - 移除 javadoc 注释的 ` * ` 前缀
   - 压缩多个空格为单个空格

**规范化示例**:
```java
// 原始 javadoc:
/**
 * This class is internal and is hence not for public use.
 * Its APIs are unstable and can change at any time.
 */

// 规范化后:
"This class is internal and is hence not for public use. Its APIs are unstable and can change at any time."
```

**为什么需要规范化**:
- javadoc 可能有不同的换行和缩进风格
- 简化字符串匹配（不需要考虑格式）
- 提高检查的鲁棒性

#### 使用示例

**示例 1: 正确的内部类**
```java
package io.opentelemetry.sdk.internal;

/**
 * This class is internal and is hence not for public use.
 * Its APIs are unstable and can change at any time.
 */
public class InternalHelper {
  // ✓ 通过检查
}
```

**示例 2: 缺少免责声明（错误）**
```java
package io.opentelemetry.sdk.internal;

// ✗ 编译警告：doesn't end with any of the applicable javadoc disclaimers
public class InternalHelper {
  // 问题：公共类在 internal 包中,但没有 javadoc
}
```

**示例 3: javadoc 不完整（错误）**
```java
package io.opentelemetry.sdk.internal;

/**
 * Helper class for internal use.
 */
// ✗ 编译警告：doesn't end with any of the applicable javadoc disclaimers
public class InternalHelper {
  // 问题：有 javadoc,但缺少必需的免责声明
}
```

**示例 4: 非公共类（跳过）**
```java
package io.opentelemetry.sdk.internal;

// ✓ 跳过检查（不是公共类）
class InternalHelper {
}
```

**示例 5: 测试类（跳过）**
```java
package io.opentelemetry.sdk.internal;

// ✓ 跳过检查（测试类）
public class InternalHelperTest {
}
```

**示例 6: 使用实验性免责声明**
```java
package io.opentelemetry.api.incubator.internal;

/**
 * This class is internal and experimental. Its APIs are unstable and can change at any time.
 * Its APIs (or a version of them) may be promoted to the public stable API in the future,
 * but no guarantees are made.
 */
public class ExperimentalFeature {
  // ✓ 通过检查（使用版本 2 免责声明）
}
```

#### 抑制警告

**方法 1: 类级别抑制**
```java
@SuppressWarnings("OtelInternalJavadoc")
public class LegacyInternalClass {
  // 抑制整个类的检查
}
```

**方法 2: 项目级别禁用**
```kotlin
// build.gradle.kts
tasks.withType<JavaCompile>().configureEach {
  options.errorprone.disable("OtelInternalJavadoc")
}
```

**方法 3: 模块级别禁用**
```kotlin
// 在特定模块的 build.gradle.kts 中
tasks.named<JavaCompile>("compileJava") {
  options.errorprone.disable("OtelInternalJavadoc")
}
```

### 3.2 OtelPrivateConstructorForUtilityClass

**文件路径**: `src/main/java/io/opentelemetry/gradle/customchecks/OtelPrivateConstructorForUtilityClass.java:1`

#### 目的和动机

**问题**: 工具类（utility class）通常只包含静态方法,不应该被实例化。如果没有私有构造函数,用户可能会错误地实例化这些类。

**Java 最佳实践**: 工具类应该有一个私有构造函数以防止实例化。

**为什么需要自定义检查器**:
- ErrorProne 已经有内置的 `PrivateConstructorForUtilityClass` 检查
- 但 OpenTelemetry 需要自定义行为（例如特定的错误消息或配置）
- 通过包装内置检查器,可以在未来添加项目特定的逻辑

#### 完整源码分析

**文件**: `OtelPrivateConstructorForUtilityClass.java` (38 行)

```java
package io.opentelemetry.gradle.customchecks;

import static com.google.errorprone.BugPattern.SeverityLevel.WARNING;
import static com.google.errorprone.matchers.Description.NO_MATCH;

import com.google.errorprone.BugPattern;
import com.google.errorprone.VisitorState;
import com.google.errorprone.bugpatterns.BugChecker;
import com.google.errorprone.bugpatterns.PrivateConstructorForUtilityClass;
import com.google.errorprone.matchers.Description;
import com.sun.source.tree.ClassTree;
```

**@BugPattern 注解**:
```java
@BugPattern(
    summary =
        "Classes which are not intended to be instantiated should be made non-instantiable with a private constructor. This includes utility classes (classes with only static members), and the main class.",
    severity = WARNING)
```

**类定义**:
```java
public class OtelPrivateConstructorForUtilityClass extends BugChecker
    implements BugChecker.ClassTreeMatcher {

  private static final long serialVersionUID = 1L;

  private final PrivateConstructorForUtilityClass delegate =
      new PrivateConstructorForUtilityClass();
```

**解析**:
- 继承 `BugChecker` 并实现 `ClassTreeMatcher`
- 持有 ErrorProne 内置检查器 `PrivateConstructorForUtilityClass` 的实例
- 使用委托模式（Delegation Pattern）

**检查逻辑**:
```java
  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    Description description = delegate.matchClass(tree, state);
    if (description == NO_MATCH) {
      return description;
    }
    return describeMatch(tree);
  }
}
```

**逻辑流程**:
```
调用内置检查器的 matchClass()
    ↓
返回 NO_MATCH? Yes → 返回 NO_MATCH（没有问题）
    ↓ No
重新包装为 OpenTelemetry 的 Description
    ↓
报告问题（使用自定义的 @BugPattern summary）
```

**为什么重新包装**:
- 使用 OpenTelemetry 特定的错误消息
- 使用自定义的检查器名称（用于 `@SuppressWarnings`）
- 未来可以添加额外的检查逻辑

#### 与内置检查的关系

**ErrorProne 内置检查**: `com.google.errorprone.bugpatterns.PrivateConstructorForUtilityClass`

**内置检查的逻辑**:
1. 检查类是否只有静态成员
2. 检查类是否是 `main` 类
3. 检查类是否有公共或默认构造函数
4. 报告应该使用私有构造函数

**OpenTelemetry 包装的价值**:
- ✅ 统一的检查器命名约定（`Otel*`）
- ✅ 可以在未来添加 OpenTelemetry 特定的逻辑
- ✅ 独立控制严重级别和错误消息

#### 使用示例

**示例 1: 工具类缺少私有构造函数（错误）**
```java
// ✗ 编译警告：should be made non-instantiable with a private constructor
public class StringUtils {
  public static String capitalize(String str) {
    // ...
  }

  public static String toLowerCase(String str) {
    // ...
  }
}
```

**修复方案**:
```java
// ✓ 通过检查
public class StringUtils {

  private StringUtils() {
    // 私有构造函数,防止实例化
  }

  public static String capitalize(String str) {
    // ...
  }

  public static String toLowerCase(String str) {
    // ...
  }
}
```

**示例 2: main 类缺少私有构造函数（错误）**
```java
// ✗ 编译警告：should be made non-instantiable with a private constructor
public class Main {
  public static void main(String[] args) {
    // ...
  }
}
```

**修复方案**:
```java
// ✓ 通过检查
public class Main {

  private Main() {
    // 私有构造函数
  }

  public static void main(String[] args) {
    // ...
  }
}
```

**示例 3: 混合类（跳过）**
```java
// ✓ 跳过检查（有实例方法,不是工具类）
public class Calculator {

  private int value;

  public Calculator(int value) {
    this.value = value;
  }

  public int add(int x) {
    return value + x;
  }

  public static int multiply(int a, int b) {
    return a * b;
  }
}
```

#### 抑制警告

```java
@SuppressWarnings("OtelPrivateConstructorForUtilityClass")
public class LegacyUtils {
  // 抑制检查（例如遗留代码）

  public static void doSomething() {
    // ...
  }
}
```

---

## 4. 构建配置详解

**文件路径**: `build.gradle.kts:1`

**文件长度**: 88 行

### 完整源码

```kotlin
plugins {
  id("otel.java-conventions")
}

dependencies {
  compileOnly("com.google.errorprone:error_prone_core")

  testImplementation("com.google.errorprone:error_prone_test_helpers")
}

otelJava.moduleName.set("io.opentelemetry.javaagent.customchecks")

// We cannot use "--release" javac option here because that will forbid exporting com.sun.tools package.
// We also can't seem to use the toolchain without the "--release" option. So disable everything.

java {
  sourceCompatibility = JavaVersion.VERSION_21
  targetCompatibility = JavaVersion.VERSION_21
  toolchain {
    languageVersion.set(null as JavaLanguageVersion?)
  }
}

tasks {
  withType<JavaCompile>().configureEach {
    with(options) {
      release.set(null as Int?)

      compilerArgs.addAll(
        listOf(
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED",
        ),
      )
    }
  }

  // only test on java 21+
  val testJavaVersion: String? by project
  if (testJavaVersion != null && Integer.valueOf(testJavaVersion) < 21) {
    test {
      enabled = false
    }
  }
}

tasks.withType<Test>().configureEach {
  // required when accessing javac internals
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED")
  jvmArgs("-XX:+IgnoreUnrecognizedVMOptions")
}

tasks.withType<Javadoc>().configureEach {
  // using com.sun.tools.javac.api.JavacTrees breaks javadoc generation
  enabled = false
}

// Our conventions apply this project as a dependency in the errorprone configuration, which would cause
// a circular dependency if trying to compile this project with that still there. So we filter this
// project out.
configurations {
  named("errorprone") {
    dependencies.removeIf {
      it is ProjectDependency && it.name == project.name
    }
  }
}

// Skip OWASP dependencyCheck task on test module
dependencyCheck {
  skip = true
}
```

### 逐段解析

#### 第 1-3 行：插件声明

```kotlin
plugins {
  id("otel.java-conventions")
}
```

**解析**:
- 应用 `otel.java-conventions` 插件
- 继承基本的 Java 配置（Checkstyle、Spotless、ErrorProne、JaCoCo 等）
- 参考 buildSrc 文档的 [3.1 otel.java-conventions](../buildSrc/README.md#31-oteljava-conventionsgradlekts)

#### 第 5-9 行：依赖配置

```kotlin
dependencies {
  compileOnly("com.google.errorprone:error_prone_core")

  testImplementation("com.google.errorprone:error_prone_test_helpers")
}
```

**解析**:

| 依赖 | 配置 | 用途 |
|------|------|------|
| `error_prone_core` | `compileOnly` | 提供 BugChecker、BugPattern 等 API |
| `error_prone_test_helpers` | `testImplementation` | 提供 CompilationTestHelper 测试框架 |

**为什么使用 `compileOnly`**:
- ErrorProne 核心库已经在运行时类路径中（由 buildSrc 引入）
- 避免重复依赖
- 减小 JAR 体积

#### 第 11 行：模块名称

```kotlin
otelJava.moduleName.set("io.opentelemetry.javaagent.customchecks")
```

**作用**: 设置 Java 模块名称（用于 JAR 清单的 `Automatic-Module-Name`）

**生成的 MANIFEST.MF**:
```manifest
Automatic-Module-Name: io.opentelemetry.javaagent.customchecks
```

#### 第 13-22 行：Java 版本配置

```kotlin
// We cannot use "--release" javac option here because that will forbid exporting com.sun.tools package.
// We also can't seem to use the toolchain without the "--release" option. So disable everything.

java {
  sourceCompatibility = JavaVersion.VERSION_21
  targetCompatibility = JavaVersion.VERSION_21
  toolchain {
    languageVersion.set(null as JavaLanguageVersion?)
  }
}
```

**解析**:

**为什么需要 Java 21**:
- 访问 `com.sun.tools.javac` API 需要较新的 JDK
- ErrorProne 最新版本要求 Java 11+
- 使用 Java 21 的新特性和 API

**为什么不使用 `--release` 参数**:
- `--release 8` 会限制只能使用 Java 8 API
- 但 `com.sun.tools.javac` 是 Java 内部 API,不在标准 API 中
- 使用 `--release` 会导致编译失败

**为什么禁用工具链**:
- Gradle 的 Java 工具链通常与 `--release` 参数配合使用
- 由于不能使用 `--release`,工具链也无法使用
- 直接设置 `sourceCompatibility` 和 `targetCompatibility`

#### 第 24-43 行：编译参数配置

```kotlin
tasks {
  withType<JavaCompile>().configureEach {
    with(options) {
      release.set(null as Int?)

      compilerArgs.addAll(
        listOf(
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED",
          "--add-exports",
          "jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED",
        ),
      )
    }
  }
```

**`--add-exports` 详解**:

| 导出的包 | 用途 |
|---------|------|
| `com.sun.tools.javac.api` | JavacTrees API（访问 javadoc） |
| `com.sun.tools.javac.code` | 符号表、类型系统 |
| `com.sun.tools.javac.comp` | 编译器组件 |
| `com.sun.tools.javac.tree` | AST 节点定义 |
| `com.sun.tools.javac.util` | 工具类 |

**格式**: `--add-exports <module>/<package>=<target-module>`
- `<module>`: `jdk.compiler`（编译器模块）
- `<package>`: 要导出的包（如 `com.sun.tools.javac.api`）
- `<target-module>`: `ALL-UNNAMED`（所有未命名模块,即类路径上的 JAR）

**为什么需要**:
- Java 9+ 引入模块系统（JPMS）
- `jdk.compiler` 模块默认不导出内部包
- `--add-exports` 显式导出这些包,允许编译时访问

#### 第 45-52 行：测试版本限制

```kotlin
  // only test on java 21+
  val testJavaVersion: String? by project
  if (testJavaVersion != null && Integer.valueOf(testJavaVersion) < 21) {
    test {
      enabled = false
    }
  }
}
```

**解析**:
- 读取 Gradle 属性 `testJavaVersion`
- 如果指定的 Java 版本 < 21,禁用测试
- 确保测试只在 Java 21+ 上运行

**使用场景**:
```bash
# 使用 Java 17 测试其他模块,跳过 custom-checks 测试
./gradlew test -PtestJavaVersion=17

# custom-checks 的测试被自动禁用
```

#### 第 54-66 行：测试运行时配置

```kotlin
tasks.withType<Test>().configureEach {
  // required when accessing javac internals
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED")
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED")
  jvmArgs("-XX:+IgnoreUnrecognizedVMOptions")
}
```

**`--add-opens` vs `--add-exports`**:

| 参数 | 编译时/运行时 | 权限 |
|------|--------------|------|
| `--add-exports` | 编译时 | 允许访问公共类型 |
| `--add-opens` | 运行时 | 允许反射访问私有成员 |

**为什么测试需要 `--add-opens`**:
- CompilationTestHelper 使用反射访问编译器内部
- 需要更深的访问权限（包括私有成员）
- 额外导出 `file`, `main`, `parser` 包（测试框架需要）

**`-XX:+IgnoreUnrecognizedVMOptions`**:
- 忽略不识别的 JVM 选项
- 确保在不同 JDK 版本上的兼容性

#### 第 68-71 行：禁用 Javadoc

```kotlin
tasks.withType<Javadoc>().configureEach {
  // using com.sun.tools.javac.api.JavacTrees breaks javadoc generation
  enabled = false
}
```

**为什么禁用 Javadoc**:
- 使用 `JavacTrees` API 会破坏 Javadoc 生成过程
- Javadoc 工具也使用 javac 内部 API,可能产生冲突
- custom-checks 是内部工具模块,不需要发布 Javadoc

#### 第 73-82 行：避免循环依赖

```kotlin
// Our conventions apply this project as a dependency in the errorprone configuration, which would cause
// a circular dependency if trying to compile this project with that still there. So we filter this
// project out.
configurations {
  named("errorprone") {
    dependencies.removeIf {
      it is ProjectDependency && it.name == project.name
    }
  }
}
```

**问题场景**:
```
otel.java-conventions 插件
    ↓ 应用到所有项目（包括 custom-checks）
配置 errorprone 依赖
    ↓ 包含 project(":custom-checks")
custom-checks 编译
    ↓ 应用 otel.java-conventions
    ↓ 尝试添加自己为 errorprone 依赖
    ↓ 循环依赖！
```

**解决方案**:
- 从 custom-checks 的 `errorprone` 配置中移除自己
- 使用 `removeIf` 过滤掉项目自身的依赖

**逻辑**:
```kotlin
dependencies.removeIf { dependency ->
  dependency is ProjectDependency  // 是项目依赖
    && dependency.name == project.name  // 名称是当前项目
}
```

#### 第 84-87 行：跳过安全扫描

```kotlin
// Skip OWASP dependencyCheck task on test module
dependencyCheck {
  skip = true
}
```

**为什么跳过**:
- custom-checks 是构建工具,不是运行时库
- 不会打包到最终产物中
- 节省构建时间

---

## 5. 测试

**文件路径**: `src/test/java/io/opentelemetry/gradle/customchecks/OtelInternalJavadocTest.java:1`

### CompilationTestHelper 框架

**CompilationTestHelper** 是 ErrorProne 提供的测试框架,用于验证自定义检查器的行为。

**工作原理**:
```
创建测试助手
    ↓
添加测试源码（内联字符串）
    ↓
标记预期的错误位置（// BUG: Diagnostic contains: ...）
    ↓
调用 doTest()
    ↓
CompilationTestHelper 编译源码
    ↓
运行自定义检查器
    ↓
验证实际错误与预期错误匹配
    ↓
测试通过/失败
```

### 测试用例结构

```java
@Test
void positiveCases() {
  CompilationTestHelper.newInstance(OtelInternalJavadoc.class, OtelInternalJavadocTest.class)
      .addSourceLines(
          "internal/InternalJavadocPositiveCases.java",  // 虚拟文件名
          "/*",
          " * Copyright The OpenTelemetry Authors",
          " * SPDX-License-Identifier: Apache-2.0",
          " */",
          "package io.opentelemetry.gradle.customchecks.internal;",
          "// BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers",
          "public class InternalJavadocPositiveCases {",
          "  // BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers",
          "  public static class One {}",
          "  /** Doesn't have the disclaimer. */",
          "  // BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers",
          "  public static class Two {}",
          "}")
      .doTest();
}
```

**关键元素**:

| 元素 | 说明 |
|------|------|
| `newInstance(OtelInternalJavadoc.class, ...)` | 创建检查器实例 |
| `addSourceLines(filename, lines...)` | 添加测试源码（内联） |
| `// BUG: Diagnostic contains: <message>` | 标记预期错误位置和消息 |
| `.doTest()` | 执行测试 |

### 正面测试用例（positiveCases）

**目的**: 验证检查器能够正确检测问题。

**测试源码**:
```java
package io.opentelemetry.gradle.customchecks.internal;

// ✗ 类 1：没有 javadoc
// BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers
public class InternalJavadocPositiveCases {

  // ✗ 嵌套类：没有 javadoc
  // BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers
  public static class One {}

  /** Doesn't have the disclaimer. */
  // ✗ 类 2：有 javadoc 但缺少免责声明
  // BUG: Diagnostic contains: doesn't end with any of the applicable javadoc disclaimers
  public static class Two {}
}
```

**验证点**:
1. 外部类缺少 javadoc → 报告错误
2. 嵌套类缺少 javadoc → 报告错误
3. 类有 javadoc 但缺少免责声明 → 报告错误

### 负面测试用例（negativeCases）

**目的**: 验证检查器不会误报。

**测试源码**:
```java
package io.opentelemetry.gradle.customchecks.internal;

/**
 * This class is internal and is hence not for public use. Its APIs are unstable and can change at
 * any time.
 */
// ✓ 外部类有正确的免责声明
public class InternalJavadocNegativeCases {

  /**
   * This class is internal and is hence not for public use. Its APIs are unstable and can change at
   * any time.
   */
  // ✓ 嵌套类有正确的免责声明
  public static class One {}

  // ✓ 非公共类（跳过检查）
  static class Two {}
}
```

**验证点**:
1. 有正确免责声明的类 → 不报告错误
2. 非公共类 → 不报告错误

### 如何运行测试

```bash
# 运行所有测试
./gradlew :custom-checks:test

# 运行特定测试
./gradlew :custom-checks:test --tests OtelInternalJavadocTest

# 运行测试并显示详细输出
./gradlew :custom-checks:test --info

# 查看测试报告
open custom-checks/build/reports/tests/test/index.html
```

### 添加新测试用例

```java
@Test
void testExperimentalDisclaimer() {
  CompilationTestHelper.newInstance(OtelInternalJavadoc.class, OtelInternalJavadocTest.class)
      .addSourceLines(
          "internal/ExperimentalTest.java",
          "package io.opentelemetry.internal;",
          "/**",
          " * This class is internal and experimental. Its APIs are unstable and can change at any time.",
          " * Its APIs (or a version of them) may be promoted to the public stable API in the",
          " * future, but no guarantees are made.",
          " */",
          "public class ExperimentalTest {",
          "}")
      .doTest();
}
```

---

## 6. 集成机制

### 6.1 在 buildSrc 中的引用

**文件**: `buildSrc/src/main/kotlin/otel.errorprone-conventions.gradle.kts:13`

```kotlin
dependencies {
  errorprone("com.google.errorprone:error_prone_core")
  errorprone("com.uber.nullaway:nullaway")
  errorprone(project(":custom-checks"))  // ← 引入 custom-checks
}
```

**集成点**:
- custom-checks 作为 `errorprone` 配置的项目依赖
- ErrorProne 在编译时加载 custom-checks JAR
- 通过 SPI 机制发现并注册自定义检查器

### 6.2 SPI 注册流程

```
custom-checks JAR 打包
    ↓ 包含
META-INF/services/com.google.errorprone.bugpatterns.BugChecker
    ↓ 列出
io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass
    ↓
buildSrc 编译
    ↓ 依赖
custom-checks.jar（在 errorprone 配置中）
    ↓
otel.errorprone-conventions 插件创建
    ↓
子项目应用 otel.java-conventions
    ↓ 继承
otel.errorprone-conventions
    ↓
子项目编译（compileJava）
    ↓
ErrorProne 启动
    ↓ 使用
ServiceLoader.load(BugChecker.class)
    ↓ 扫描
errorprone 配置中的所有 JAR
    ↓ 读取
META-INF/services/com.google.errorprone.bugpatterns.BugChecker
    ↓ 发现
custom-checks 中的检查器
    ↓ 加载并实例化
OtelInternalJavadoc, OtelPrivateConstructorForUtilityClass
    ↓ 注册到
ErrorProne 引擎
    ↓
编译时遍历 AST
    ↓ 调用
每个 BugChecker 的 matchXxx() 方法
    ↓ 收集
Description（问题报告）
    ↓ 输出
编译警告/错误
```

### 6.3 ErrorProne 发现和加载过程

**详细流程**:

#### 步骤 1: ErrorProne 插件初始化

```java
// ErrorProne 插件启动（简化示例）
public class ErrorProneJavaCompiler {
  private List<BugChecker> scanners;

  public void init() {
    // 使用 ServiceLoader 加载所有 BugChecker 实现
    scanners = ServiceLoader.load(BugChecker.class, classLoader)
      .stream()
      .map(ServiceLoader.Provider::get)
      .collect(Collectors.toList());
  }
}
```

#### 步骤 2: 扫描类路径

```
ErrorProne 扫描 errorprone 配置的类路径
    ↓ 查找
JAR 文件中的 META-INF/services/ 目录
    ↓ 查找文件
com.google.errorprone.bugpatterns.BugChecker
    ↓ 找到
custom-checks.jar!/META-INF/services/com.google.errorprone.bugpatterns.BugChecker
    ↓ 读取内容
io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass
```

#### 步骤 3: 加载检查器类

```java
// ServiceLoader 加载类（简化示例）
for (String className : serviceDescriptor.readLines()) {
  Class<?> checkerClass = classLoader.loadClass(className);
  BugChecker checker = (BugChecker) checkerClass.getDeclaredConstructor().newInstance();
  scanners.add(checker);
}
```

#### 步骤 4: 编译时调用

```java
// 遍历 AST 并调用匹配器（简化示例）
for (ClassTree classTree : compilationUnit.getTypeDecls()) {
  for (BugChecker scanner : scanners) {
    if (scanner instanceof BugChecker.ClassTreeMatcher) {
      Description description = ((ClassTreeMatcher) scanner).matchClass(classTree, state);
      if (description != Description.NO_MATCH) {
        reportDiagnostic(description);
      }
    }
  }
}
```

### 6.4 构建流程图

```
┌─────────────────────────────────────────────────────────┐
│ 1. buildSrc 编译阶段                                     │
│    ./gradlew :custom-checks:build                       │
│    → 编译 OtelInternalJavadoc.java                      │
│    → 编译 OtelPrivateConstructorForUtilityClass.java    │
│    → 打包 custom-checks.jar                             │
│       ├── *.class 文件                                  │
│       └── META-INF/services/...BugChecker               │
│    → 发布到 buildSrc 的本地仓库                         │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│ 2. buildSrc 插件创建阶段                                 │
│    otel.errorprone-conventions.gradle.kts 评估         │
│    → 添加依赖：errorprone(project(":custom-checks"))   │
│    → 配置 ErrorProne 编译选项                           │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│ 3. 子项目编译阶段                                        │
│    ./gradlew :api:compileJava                           │
│    → 应用 otel.java-conventions 插件                    │
│       ↓ 包含 otel.errorprone-conventions                │
│    → ErrorProne 加载到编译器类路径                      │
│       ↓ 包含 custom-checks.jar                          │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│ 4. ErrorProne 初始化                                     │
│    → ServiceLoader.load(BugChecker.class)               │
│    → 扫描 custom-checks.jar 的 META-INF/services        │
│    → 发现并加载检查器类                                  │
│       ├── OtelInternalJavadoc                           │
│       └── OtelPrivateConstructorForUtilityClass         │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│ 5. 编译和检查                                            │
│    → javac 生成 AST                                     │
│    → ErrorProne 遍历 AST 节点                           │
│    → 调用每个 BugChecker 的 matchXxx() 方法             │
│       ├── OtelInternalJavadoc.matchClass()             │
│       └── OtelPrivateConstructorForUtilityClass.matchClass() │
│    → 收集 Description（问题报告）                       │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│ 6. 输出结果                                              │
│    → 编译警告输出到控制台                                │
│    → 编译继续（WARNING 不会失败构建）                    │
│    → 生成编译报告（如果配置）                            │
└─────────────────────────────────────────────────────────┘
```

### 6.5 依赖关系图

```
settings.gradle.kts
    ├── include(":buildSrc")
    └── include(":custom-checks")
             │
             ↓
     custom-checks/build.gradle.kts
     ├── plugins: otel.java-conventions
     ├── dependencies: error_prone_core (compileOnly)
     └── 生成: custom-checks.jar + SPI 描述符
             │
             ↓
     buildSrc/src/main/kotlin/otel.errorprone-conventions.gradle.kts
     ├── dependencies: errorprone(project(":custom-checks"))
     └── 配置 ErrorProne 选项
             │
             ↓
     buildSrc/src/main/kotlin/otel.java-conventions.gradle.kts
     ├── plugins: otel.errorprone-conventions
     └── 应用到所有子项目
             │
             ↓
     子项目（如 api/build.gradle.kts）
     ├── plugins: otel.java-conventions
     └── 编译时自动应用 custom-checks
```

---

## 7. 使用指南

### 7.1 如何在项目中启用

**默认行为**: custom-checks 已经自动应用到所有使用 `otel.java-conventions` 插件的子项目。

**验证检查是否生效**:
```bash
# 编译任意子项目
./gradlew :api:compileJava

# 查看编译输出,应包含 ErrorProne 检查结果
# 如果有违规,会看到警告：
# warning: [OtelInternalJavadoc] ...
```

### 7.2 如何禁用特定检查

#### 方法 1: 全局禁用（所有子项目）

**编辑**: `buildSrc/src/main/kotlin/otel.errorprone-conventions.gradle.kts`

```kotlin
tasks.named<JavaCompile>("compileJava") {
  options.errorprone {
    disable("OtelInternalJavadoc")
    disable("OtelPrivateConstructorForUtilityClass")
  }
}
```

#### 方法 2: 项目级别禁用

**编辑**: 特定子项目的 `build.gradle.kts`

```kotlin
tasks.withType<JavaCompile>().configureEach {
  options.errorprone {
    disable("OtelInternalJavadoc")
  }
}
```

#### 方法 3: 源码级别抑制

```java
// 类级别抑制
@SuppressWarnings("OtelInternalJavadoc")
public class MyInternalClass {
  // 整个类的检查被抑制
}

// 方法级别抑制
public class MyInternalClass {
  @SuppressWarnings("OtelPrivateConstructorForUtilityClass")
  public static void utilityMethod() {
    // 仅此方法的检查被抑制
  }
}

// 多个检查抑制
@SuppressWarnings({"OtelInternalJavadoc", "OtelPrivateConstructorForUtilityClass"})
public class MyClass {
  // ...
}
```

### 7.3 如何查看检查报告

#### 控制台输出

```bash
./gradlew :api:compileJava

# 输出示例:
# /path/to/InternalClass.java:10: warning: [OtelInternalJavadoc] This public internal class doesn't end with any of the applicable javadoc disclaimers
# public class InternalClass {
#        ^
```

#### 文本报告

**默认位置**: `build/reports/errorprone/main.txt`

```bash
# 查看报告
cat api/build/reports/errorprone/main.txt

# 搜索特定检查
grep "OtelInternalJavadoc" api/build/reports/errorprone/main.txt
```

#### HTML 报告

ErrorProne 默认不生成 HTML 报告。可以通过配置添加:

```kotlin
tasks.withType<JavaCompile>().configureEach {
  options.errorprone {
    // 生成 HTML 报告（需要额外配置）
    errorproneArgs.add("-XepReportPath=build/reports/errorprone/report.html")
  }
}
```

### 7.4 如何调整严重级别

**在 custom-checks 源码中修改 @BugPattern**:

```java
// 修改前：WARNING（警告,不会失败构建）
@BugPattern(
    summary = "...",
    severity = WARNING)

// 修改后：ERROR（错误,会失败构建）
@BugPattern(
    summary = "...",
    severity = ERROR)
```

**或在 buildSrc 中覆盖严重级别**:

```kotlin
tasks.named<JavaCompile>("compileJava") {
  options.errorprone {
    // 将警告提升为错误
    error("OtelInternalJavadoc")

    // 将错误降级为警告
    warn("OtelPrivateConstructorForUtilityClass")

    // 将检查设为建议（不报告）
    // suggestion("OtelInternalJavadoc")
  }
}
```

### 7.5 如何添加新的自定义检查

参考 [9. 开发指南](#9-开发指南) 章节。

---

## 8. 故障排查

### 8.1 常见错误

#### 错误 1: 编译失败 - 无法访问 com.sun.tools.javac

**错误消息**:
```
error: package com.sun.tools.javac.api does not exist
import com.sun.tools.javac.api.JavacTrees;
                            ^
```

**原因**: 缺少 `--add-exports` 参数,javac 内部包未被导出。

**解决方案**:
```kotlin
tasks.withType<JavaCompile>().configureEach {
  options.compilerArgs.addAll(listOf(
    "--add-exports", "jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED",
    "--add-exports", "jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED"
    // ... 其他包
  ))
}
```

#### 错误 2: 测试失败 - 反射访问被拒绝

**错误消息**:
```
java.lang.reflect.InaccessibleObjectException: Unable to make field ... accessible:
module jdk.compiler does not "opens com.sun.tools.javac.code" to unnamed module
```

**原因**: 测试时缺少 `--add-opens` JVM 参数。

**解决方案**:
```kotlin
tasks.withType<Test>().configureEach {
  jvmArgs("--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED")
  // ... 其他包
}
```

#### 错误 3: 循环依赖

**错误消息**:
```
Circular dependency between the following tasks:
:custom-checks:compileJava
\--- :custom-checks:compileJava (*)
```

**原因**: custom-checks 应用了 `otel.java-conventions`,后者包含对 custom-checks 的依赖。

**解决方案**: 已在 build.gradle.kts 中处理:
```kotlin
configurations {
  named("errorprone") {
    dependencies.removeIf {
      it is ProjectDependency && it.name == project.name
    }
  }
}
```

#### 错误 4: SPI 描述符未打包

**症状**: 检查器未被加载,编译时没有警告。

**检查**:
```bash
# 查看 JAR 内容
jar tf custom-checks/build/libs/custom-checks.jar | grep META-INF/services

# 应该输出:
# META-INF/services/com.google.errorprone.bugpatterns.BugChecker
```

**解决方案**: 确保 `src/main/resources/META-INF/services/` 目录正确,文件名完整:
```
src/main/resources/META-INF/services/com.google.errorprone.bugpatterns.BugChecker
```

#### 错误 5: Java 版本不匹配

**错误消息**:
```
Execution failed for task ':custom-checks:compileJava'.
> error: invalid source release: 21
```

**原因**: 系统 Java 版本 < 21。

**解决方案**:
```bash
# 检查 Java 版本
java -version

# 安装 Java 21
# macOS:
brew install openjdk@21

# Linux:
sudo apt install openjdk-21-jdk

# 或在 gradle.properties 中指定 Java 路径:
org.gradle.java.home=/path/to/jdk-21
```

### 8.2 调试技巧

#### 技巧 1: 启用 ErrorProne 详细日志

```kotlin
tasks.withType<JavaCompile>().configureEach {
  options.errorprone {
    errorproneArgs.add("-XepAllErrorsAsWarnings")  // 所有错误显示为警告
    errorproneArgs.add("-Xep:OtelInternalJavadoc:WARN")  // 指定检查器日志级别
  }
}
```

#### 技巧 2: 打印 ErrorProne 加载的检查器

```bash
# 运行编译并输出调试信息
./gradlew :api:compileJava --debug 2>&1 | grep "BugChecker"

# 应该看到 custom-checks 的检查器被加载
```

#### 技巧 3: 单独编译和测试 custom-checks

```bash
# 清理并重新编译
./gradlew :custom-checks:clean :custom-checks:build

# 查看编译输出
./gradlew :custom-checks:build --info

# 运行测试
./gradlew :custom-checks:test --info
```

#### 技巧 4: 验证 SPI 配置

```bash
# 查看 SPI 描述符内容
cat custom-checks/src/main/resources/META-INF/services/com.google.errorprone.bugpatterns.BugChecker

# 验证类名是否正确
# 应该输出:
# io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
# io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass
```

#### 技巧 5: 测试特定检查器

```java
// 创建临时测试类
public class DebugTest {
  @Test
  public void debug() {
    CompilationTestHelper.newInstance(OtelInternalJavadoc.class, getClass())
      .addSourceLines(
        "Test.java",
        "package io.opentelemetry.internal;",
        "public class Test {}"
      )
      .expectErrorMessage("OtelInternalJavadoc", Predicates.containsPattern("disclaimers"))
      .doTest();
  }
}
```

### 8.3 日志查看

#### Gradle 构建日志

```bash
# 详细日志
./gradlew :custom-checks:build --info

# 调试日志（非常详细）
./gradlew :custom-checks:build --debug

# 堆栈跟踪日志
./gradlew :custom-checks:build --stacktrace

# 完整堆栈跟踪
./gradlew :custom-checks:build --full-stacktrace
```

#### 测试日志

```bash
# 查看测试输出
cat custom-checks/build/reports/tests/test/index.html

# 查看测试标准输出
cat custom-checks/build/test-results/test/TEST-*.xml
```

---

## 9. 开发指南

### 9.1 如何添加新检查

#### 步骤 1: 创建检查器类

**文件**: `src/main/java/io/opentelemetry/gradle/customchecks/MyCustomCheck.java`

```java
package io.opentelemetry.gradle.customchecks;

import static com.google.errorprone.BugPattern.SeverityLevel.WARNING;

import com.google.errorprone.BugPattern;
import com.google.errorprone.VisitorState;
import com.google.errorprone.bugpatterns.BugChecker;
import com.google.errorprone.matchers.Description;
import com.sun.source.tree.ClassTree;

@BugPattern(
    name = "MyCustomCheck",
    summary = "检查描述：例如 '类名应该以 Impl 结尾'",
    severity = WARNING)
public class MyCustomCheck extends BugChecker implements BugChecker.ClassTreeMatcher {

  private static final long serialVersionUID = 1L;

  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 检查逻辑
    String className = tree.getSimpleName().toString();
    if (应该触发检查的条件) {
      return describeMatch(tree);
    }
    return Description.NO_MATCH;
  }
}
```

#### 步骤 2: 注册到 SPI

**编辑**: `src/main/resources/META-INF/services/com.google.errorprone.bugpatterns.BugChecker`

```
io.opentelemetry.gradle.customchecks.OtelInternalJavadoc
io.opentelemetry.gradle.customchecks.OtelPrivateConstructorForUtilityClass
io.opentelemetry.gradle.customchecks.MyCustomCheck  ← 添加新检查器
```

#### 步骤 3: 添加测试

**文件**: `src/test/java/io/opentelemetry/gradle/customchecks/MyCustomCheckTest.java`

```java
package io.opentelemetry.gradle.customchecks;

import com.google.errorprone.CompilationTestHelper;
import org.junit.jupiter.api.Test;

class MyCustomCheckTest {

  @Test
  void positiveCases() {
    CompilationTestHelper.newInstance(MyCustomCheck.class, MyCustomCheckTest.class)
        .addSourceLines(
            "Test.java",
            "package test;",
            "// BUG: Diagnostic contains: 检查描述",
            "public class Test {",
            "}")
        .doTest();
  }

  @Test
  void negativeCases() {
    CompilationTestHelper.newInstance(MyCustomCheck.class, MyCustomCheckTest.class)
        .addSourceLines(
            "TestImpl.java",
            "package test;",
            "public class TestImpl {",  // 符合规则,不报错
            "}")
        .doTest();
  }
}
```

#### 步骤 4: 编译和测试

```bash
# 编译 custom-checks
./gradlew :custom-checks:build

# 运行测试
./gradlew :custom-checks:test

# 验证新检查器在子项目中生效
./gradlew :api:compileJava
```

#### 步骤 5: 文档化

在本 README 中添加新检查器的说明:
- 检查目的
- 检查逻辑
- 使用示例
- 抑制方法

### 9.2 测试最佳实践

#### 实践 1: 分离正面和负面测试

```java
@Test
void positiveCases() {
  // 测试应该触发警告的情况
  CompilationTestHelper.newInstance(...)
      .addSourceLines(...)
      .expectNoDiagnostics()  // ← 错误：正面测试应该有诊断
      .doTest();
}

@Test
void negativeCases() {
  // 测试不应该触发警告的情况
  CompilationTestHelper.newInstance(...)
      .addSourceLines(...)
      .expectNoDiagnostics()  // ✓ 正确
      .doTest();
}
```

#### 实践 2: 测试边界条件

```java
@Test
void testEdgeCases() {
  CompilationTestHelper.newInstance(MyCustomCheck.class, getClass())
      // 测试空类
      .addSourceLines("Empty.java", "package test;", "public class Empty {}")
      // 测试抽象类
      .addSourceLines("Abstract.java", "package test;", "public abstract class Abstract {}")
      // 测试接口
      .addSourceLines("Interface.java", "package test;", "public interface Interface {}")
      .doTest();
}
```

#### 实践 3: 使用描述性的测试名称

```java
// ✗ 不好
@Test
void test1() { ... }

// ✓ 好
@Test
void shouldReportErrorWhenClassNameDoesNotEndWithImpl() { ... }
```

#### 实践 4: 测试 @SuppressWarnings

```java
@Test
void testSuppression() {
  CompilationTestHelper.newInstance(MyCustomCheck.class, getClass())
      .addSourceLines(
          "Test.java",
          "package test;",
          "@SuppressWarnings(\"MyCustomCheck\")",
          "public class Test {",  // 不应报错（被抑制）
          "}")
      .doTest();
}
```

### 9.3 性能考虑

#### 考虑 1: 最小化 AST 遍历

```java
// ✗ 不好：每个类都遍历所有方法
@Override
public Description matchClass(ClassTree tree, VisitorState state) {
  for (Tree member : tree.getMembers()) {
    if (member instanceof MethodTree) {
      // 检查每个方法
    }
  }
  return Description.NO_MATCH;
}

// ✓ 好：仅当必要时遍历
@Override
public Description matchClass(ClassTree tree, VisitorState state) {
  if (!needsCheck(tree)) {
    return Description.NO_MATCH;  // 快速退出
  }
  // 仅在必要时遍历方法
  for (Tree member : tree.getMembers()) {
    // ...
  }
  return Description.NO_MATCH;
}
```

#### 考虑 2: 缓存计算结果

```java
// ✗ 不好：重复计算
private boolean isInternal(VisitorState state) {
  String packageName = getPackageName(state);  // 每次调用都计算
  return packageName.contains("internal");
}

// ✓ 好：缓存包名
private static final Pattern INTERNAL_PATTERN = Pattern.compile("\\binternal\\b");

private boolean isInternal(VisitorState state) {
  String packageName = getPackageName(state);
  return INTERNAL_PATTERN.matcher(packageName).find();  // 使用预编译的正则
}
```

#### 考虑 3: 避免不必要的字符串操作

```java
// ✗ 不好：多次字符串拼接
String message = "Error: " + tree.getSimpleName() + " does not match " + pattern;

// ✓ 好：使用 String.format 或 StringBuilder
String message = String.format("Error: %s does not match %s",
    tree.getSimpleName(), pattern);
```

#### 考虑 4: 快速失败（Fail Fast）

```java
@Override
public Description matchClass(ClassTree tree, VisitorState state) {
  // 步骤 1: 最快的检查（修饰符检查）
  if (!isPublic(tree)) {
    return Description.NO_MATCH;
  }

  // 步骤 2: 快速检查（包名检查）
  if (!isInternal(state)) {
    return Description.NO_MATCH;
  }

  // 步骤 3: 昂贵的检查（访问 javadoc）
  String javadoc = getJavadoc(state);
  if (javadoc != null && javadoc.contains(EXPECTED_COMMENT)) {
    return Description.NO_MATCH;
  }

  return describeMatch(tree);
}
```

---

## 10. 架构和设计模式

### 10.1 插件模式（Plugin Pattern）

**定义**: 通过插件机制扩展核心功能,无需修改核心代码。

**custom-checks 中的应用**:
```
ErrorProne（核心）
    ↓ 定义接口
BugChecker（插件接口）
    ↓ 实现
OtelInternalJavadoc, OtelPrivateConstructorForUtilityClass（插件实现）
    ↓ 注册
SPI（插件注册机制）
    ↓ 加载
ServiceLoader（插件加载器）
```

**优势**:
- ✅ 松耦合：检查器和 ErrorProne 核心解耦
- ✅ 可扩展：添加新检查器无需修改 ErrorProne
- ✅ 热插拔：检查器可以独立开发和部署

### 10.2 访问者模式（Visitor Pattern）

**定义**: 将算法与对象结构分离,在不修改对象结构的情况下定义新操作。

**AST 遍历中的应用**:
```java
// AST 结构（对象结构）
CompilationUnitTree
├── PackageTree
├── ImportTree
└── ClassTree
    ├── MethodTree
    ├── VariableTree
    └── ...

// 访问者（算法）
BugChecker.ClassTreeMatcher {
  Description matchClass(ClassTree tree, VisitorState state);
}
```

**custom-checks 中的使用**:
```java
// ErrorProne 遍历 AST 并调用访问者
public class ErrorProneScanner {
  public void scan(CompilationUnitTree tree) {
    for (Tree node : tree.getTypeDecls()) {
      if (node instanceof ClassTree) {
        ClassTree classTree = (ClassTree) node;
        for (BugChecker checker : checkers) {
          if (checker instanceof ClassTreeMatcher) {
            ((ClassTreeMatcher) checker).matchClass(classTree, state);
          }
        }
      }
    }
  }
}
```

**优势**:
- ✅ 开放-封闭原则：AST 结构不变,检查逻辑可扩展
- ✅ 单一职责：每个检查器专注于一种检查
- ✅ 易于添加新操作：只需实现新的访问者

### 10.3 SPI 模式（Service Provider Interface）

**定义**: Java 标准的插件发现和加载机制。

**组成部分**:
1. **服务接口**：`BugChecker`
2. **服务实现**：`OtelInternalJavadoc`, `OtelPrivateConstructorForUtilityClass`
3. **服务注册**：`META-INF/services/com.google.errorprone.bugpatterns.BugChecker`
4. **服务加载器**：`ServiceLoader.load(BugChecker.class)`

**工作流程**:
```
1. 定义服务接口
   com.google.errorprone.bugpatterns.BugChecker

2. 实现服务
   io.opentelemetry.gradle.customchecks.OtelInternalJavadoc

3. 注册服务（META-INF/services 文件）
   com.google.errorprone.bugpatterns.BugChecker → OtelInternalJavadoc

4. 加载服务
   ServiceLoader<BugChecker> loader = ServiceLoader.load(BugChecker.class);
   for (BugChecker checker : loader) {
     // 使用检查器
   }
```

**优势**:
- ✅ 标准化：遵循 Java 标准机制
- ✅ 解耦：服务使用者和服务实现者解耦
- ✅ 动态发现：运行时自动发现所有实现

### 10.4 委托模式（Delegation Pattern）

**定义**: 将请求转发给另一个对象处理。

**OtelPrivateConstructorForUtilityClass 中的应用**:
```java
public class OtelPrivateConstructorForUtilityClass extends BugChecker {

  // 委托对象
  private final PrivateConstructorForUtilityClass delegate =
      new PrivateConstructorForUtilityClass();

  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 委托给内置检查器
    Description description = delegate.matchClass(tree, state);

    // 包装结果
    if (description != NO_MATCH) {
      return describeMatch(tree);
    }
    return description;
  }
}
```

**优势**:
- ✅ 代码复用：复用 ErrorProne 内置检查器的逻辑
- ✅ 灵活性：可以在委托前后添加自定义逻辑
- ✅ 可扩展：未来可以添加 OpenTelemetry 特定的检查

### 10.5 模板方法模式（Template Method Pattern）

**定义**: 定义算法骨架,将某些步骤延迟到子类实现。

**BugChecker 中的应用**:
```java
// 抽象基类（定义算法骨架）
public abstract class BugChecker implements Serializable {

  // 模板方法
  public final void scan(Tree node, VisitorState state) {
    Description description = match(node, state);  // 调用抽象方法
    if (description != Description.NO_MATCH) {
      report(description);  // 固定步骤
    }
  }

  // 抽象方法（由子类实现）
  protected abstract Description match(Tree node, VisitorState state);
}

// 具体实现
public class OtelInternalJavadoc extends BugChecker
    implements BugChecker.ClassTreeMatcher {

  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 实现具体的检查逻辑
    return Description.NO_MATCH;
  }
}
```

**优势**:
- ✅ 代码复用：通用逻辑在基类中实现
- ✅ 一致性：所有检查器遵循相同的执行流程
- ✅ 扩展点：子类专注于实现检查逻辑

---

## 11. 最佳实践

### 11.1 编写检查器的原则

#### 原则 1: 单一职责

```java
// ✗ 不好：一个检查器做多件事
@BugPattern(name = "MultipleChecks", ...)
public class MultipleChecks extends BugChecker
    implements ClassTreeMatcher, MethodTreeMatcher {

  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 检查类名、javadoc、修饰符...
  }

  @Override
  public Description matchMethod(MethodTree tree, VisitorState state) {
    // 检查方法命名、返回类型...
  }
}

// ✓ 好：每个检查器专注一件事
@BugPattern(name = "ClassNamingCheck", ...)
public class ClassNamingCheck extends BugChecker implements ClassTreeMatcher {
  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 仅检查类名
  }
}

@BugPattern(name = "JavadocCheck", ...)
public class JavadocCheck extends BugChecker implements ClassTreeMatcher {
  @Override
  public Description matchClass(ClassTree tree, VisitorState state) {
    // 仅检查 javadoc
  }
}
```

#### 原则 2: 快速失败

```java
// ✓ 好：尽早返回
@Override
public Description matchClass(ClassTree tree, VisitorState state) {
  // 最快的检查放在最前面
  if (!isPublic(tree)) {
    return Description.NO_MATCH;
  }

  if (!isInternal(state)) {
    return Description.NO_MATCH;
  }

  // 昂贵的操作放在最后
  String javadoc = getJavadoc(state);
  if (javadoc == null) {
    return describeMatch(tree);
  }

  return Description.NO_MATCH;
}
```

#### 原则 3: 清晰的错误消息

```java
// ✗ 不好：模糊的错误消息
@BugPattern(
    summary = "Bad class",
    severity = WARNING)

// ✓ 好：具体、可操作的错误消息
@BugPattern(
    summary = "This public internal class doesn't end with any of the applicable javadoc disclaimers: "
        + "\"This class is internal and is hence not for public use. "
        + "Its APIs are unstable and can change at any time.\"",
    severity = WARNING)
```

#### 原则 4: 避免误报

```java
// ✓ 好：排除明显不需要检查的情况
@Override
public Description matchClass(ClassTree tree, VisitorState state) {
  // 排除测试类
  if (tree.getSimpleName().toString().endsWith("Test")) {
    return Description.NO_MATCH;
  }

  // 排除生成的代码
  if (ASTHelpers.hasAnnotation(tree, "javax.annotation.Generated", state)) {
    return Description.NO_MATCH;
  }

  // 执行检查
  return performCheck(tree, state);
}
```

### 11.2 错误消息设计

#### 设计 1: 提供上下文

```java
// ✗ 不好：缺少上下文
"Missing javadoc"

// ✓ 好：提供完整上下文
"This public internal class doesn't end with any of the applicable javadoc disclaimers: "
    + "\"This class is internal and is hence not for public use. "
    + "Its APIs are unstable and can change at any time.\""
```

#### 设计 2: 提供解决方案

```java
// ✗ 不好：只说明问题
"Utility class should not be instantiable"

// ✓ 好：提供解决方案
"Classes which are not intended to be instantiated should be made non-instantiable "
    + "with a private constructor. Add: private ClassName() {}"
```

#### 设计 3: 使用一致的格式

```
格式: <问题描述>. <预期行为>. <如何修复>

示例:
"This public internal class doesn't end with any of the applicable javadoc disclaimers. "
+ "Internal classes must have a disclaimer. "
+ "Add: /** This class is internal and is hence not for public use. "
+ "Its APIs are unstable and can change at any time. */"
```

### 11.3 性能优化

#### 优化 1: 使用正则表达式缓存

```java
// ✓ 好：编译一次,重复使用
private static final Pattern INTERNAL_PATTERN = Pattern.compile("\\binternal\\b");

private boolean isInternal(String packageName) {
  return INTERNAL_PATTERN.matcher(packageName).find();
}
```

#### 优化 2: 最小化 AST 访问

```java
// ✗ 不好：多次访问 AST
String className1 = tree.getSimpleName().toString();
String className2 = tree.getSimpleName().toString();

// ✓ 好：缓存结果
String className = tree.getSimpleName().toString();
// 使用 className 多次
```

#### 优化 3: 避免不必要的对象创建

```java
// ✗ 不好：每次调用都创建新对象
private List<String> getExpectedComments() {
  return Arrays.asList(
      "This class is internal...",
      "This class is experimental..."
  );
}

// ✓ 好：使用常量
private static final List<String> EXPECTED_COMMENTS = List.of(
    "This class is internal...",
    "This class is experimental..."
);
```

### 11.4 测试覆盖

#### 覆盖 1: 正面测试（应该报错）

```java
@Test
void shouldReportErrorForClassWithoutJavadoc() {
  CompilationTestHelper.newInstance(...)
      .addSourceLines(
          "Test.java",
          "package io.opentelemetry.internal;",
          "// BUG: Diagnostic contains: disclaimers",
          "public class Test {",
          "}")
      .doTest();
}
```

#### 覆盖 2: 负面测试（不应该报错）

```java
@Test
void shouldNotReportErrorForClassWithCorrectJavadoc() {
  CompilationTestHelper.newInstance(...)
      .addSourceLines(
          "Test.java",
          "package io.opentelemetry.internal;",
          "/** This class is internal and is hence not for public use. */",
          "public class Test {",
          "}")
      .doTest();
}
```

#### 覆盖 3: 边界条件

```java
@Test
void shouldSkipTestClasses() {
  // 测试以 "Test" 结尾的类被跳过
}

@Test
void shouldSkipNonPublicClasses() {
  // 测试非公共类被跳过
}

@Test
void shouldSkipNonInternalPackages() {
  // 测试非 internal 包被跳过
}
```

#### 覆盖 4: 抑制机制

```java
@Test
void shouldRespectSuppressWarnings() {
  CompilationTestHelper.newInstance(...)
      .addSourceLines(
          "Test.java",
          "package io.opentelemetry.internal;",
          "@SuppressWarnings(\"OtelInternalJavadoc\")",
          "public class Test {",  // 不应报错（被抑制）
          "}")
      .doTest();
}
```

---

## 12. 扩展阅读

### ErrorProne 官方文档

- **官方网站**: https://errorprone.info/
- **GitHub**: https://github.com/google/error-prone
- **编写自定义检查**: https://errorprone.info/docs/plugins
- **BugChecker API**: https://errorprone.info/api/latest/

### Java Compiler API

- **javac Tree API**: https://docs.oracle.com/en/java/javase/21/docs/api/jdk.compiler/com/sun/source/tree/package-summary.html
- **JavacTrees**: https://docs.oracle.com/en/java/javase/21/docs/api/jdk.compiler/com/sun/tools/javac/api/JavacTrees.html
- **DocCommentTree**: https://docs.oracle.com/en/java/javase/21/docs/api/jdk.compiler/com/sun/source/doctree/DocCommentTree.html

### Service Provider Interface (SPI)

- **ServiceLoader**: https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html
- **SPI 机制详解**: https://www.baeldung.com/java-spi

### 相关章节

- **buildSrc/README.md**:
  - [3.3 otel.errorprone-conventions](../buildSrc/README.md#33-otelerrorprone-conventionsgradlekts) - ErrorProne 配置
  - [5. 架构和设计模式](../buildSrc/README.md#5-架构和设计模式) - 设计模式详解

### OpenTelemetry Java 相关

- **主仓库**: https://github.com/open-telemetry/opentelemetry-java
- **贡献指南**: https://github.com/open-telemetry/opentelemetry-java/blob/main/CONTRIBUTING.md

---

**最后更新**: 2026-01-09
**文档版本**: 1.0.0

**维护者**: OpenTelemetry Java 项目组
**问题反馈**: [GitHub Issues](https://github.com/open-telemetry/opentelemetry-java/issues)
