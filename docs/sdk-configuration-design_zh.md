# 🚨 仅供存档使用！ 🚨
> 这是一份历史设计文档，不会随着 API 的演进而更新。有关 SDK 配置的最新示例，请参阅
> [手动仪表化文档](https://opentelemetry.io/docs/languages/java/instrumentation/)
> 和 [SDK 配置文档](https://opentelemetry.io/docs/languages/java/configuration/)。
> 本文档仅出于历史目的在此保留，旨在帮助未来的读者理解某些设计决策背后的基本原理。

# SDK 配置设计

本文档概述了我们对 SDK 用户配置的一些目标。这是对 https://github.com/open-telemetry/opentelemetry-java/issues/2022 中开始的讨论的延续。

## 目标受众

我们的配置方案涉及几个不同的目标受众。

- **应用程序开发人员（即最终用户）**。通常不了解链路追踪，但希望将 OpenTelemetry 添加到他们的应用程序中，并在控制台中看到追踪数据。应用程序开发人员的数量将永远在增加，而以下几类人员则相对固定。

- **Dev-ops / 框架开发人员**。编写组件或框架以支持向其应用程序开发人员提供链路追踪。可能会编写自定义 SDK 扩展（如导出器、采样器）以适应其内部基础设施，因此至少对 SDK 呈现的链路追踪有一定的熟悉度。

- **遥测扩展作者**。编写自定义 SDK 扩展，通常是为了支持特定的后端。非常熟悉遥测技术。

- **OpenTelemetry 维护者**。编写 SDK 代码。

在做决定时，特别是关于复杂性的决定，我们总是优先考虑应用程序开发人员，然后是框架开发人员，最后是维护者。这是因为我们预计那些对链路追踪领域知识较少的人需要比那些知识较多的人更简单的体验。此外，通过使最终用户体验尽可能简化，可以获得最大的收益，因为我们预计他们的数量将远多于其他受众。

## 目标与非目标

### 目标

- **提供配置 SDK 的单一入口点**。对于不太熟悉 SDK 的最终用户，我们希望将所有内容整合在一起，以提供可发现性和更简单的最终用户代码。如果有几个明确的用例受益于不同的入口点，我们可以为每个用例提供对应的入口点。

- **很好地适应常见的 Java 惯用语**，如依赖注入，或常见的框架如 Spring。

- **减少陷阱或配置失误的机会**。

- **旨在为最常见的用例提供配置后的 SDK 的最佳性能**。

### 非目标

- **为自定义 SDK 提供最佳体验**。通常，自定义 SDK 的体验负担可以落在其作者身上，我们针对标准用法（即完整的 SDK）进行优化。本文档中提到的“SDK”均指包含所有信号的完整 SDK。

- **确保一切都是可自动配置的**。这超出了 SDK 的范围，而是留给自动配置层，这将在下面描述，但不作为核心 SDK 的一部分。SDK 提供了一个自动配置扩展作为选项，它不是主 SDK 组件的内部部分。

## 配置 SDK 实例

SDK 公开了它支持的所有信号的配置选项。用户对如何使用 SDK 都有不同的要求；例如，他们可能会根据后端使用不同的导出器。因为我们无法猜测用户需要的配置，所以我们预计 SDK 必须由用户在配置后才能使用。

配置 SDK 的目标是：

- 选项的可发现性
- 最终用户易于使用，例如，需要更少的复杂代码
- 避免要求重复配置，这可能导致错误或混淆
- 尽可能提供良好的默认值

在 Java 中，构建器（Builder）模式是配置实例的常用模式。让我们看看这可能是什么样子的。最简单的配置是当用户想要获得默认体验，使用特定的导出器导出到端点时。

SDK 构建器将简单地接受其组件作为构建器参数。它只允许设置 SDK 实现，不适用于部分 SDK。

```java
class OpenTelemetrySdkBuilder {
  public OpenTelemetrySdkBuilder setTracerProvider(SdkTracerProvider tracerProvider);
  public OpenTelemetrySdkBuilder setPropagators(ContextPropagators propagators);
  public OpenTelemetrySdk buildAndRegisterGlobal();
  public OpenTelemetrySdk build();
}
```

Metrics（指标）尚未 GA（正式发布），必须单独配置。最终，Metrics 配置将成为 `OpenTelemetrySdkBuilder` 的一部分。

一个非常简单的配置可能如下所示：

```java
class HelloWorld {
  public static void main(String[] args) {
    SdkTracerProvider tracerProvider =
        SdkTracerProvider.builder()
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    OtlpGrpcSpanExporter.builder()
                        .setEndpoint("https://collector-service:4317")
                        .build())
                    .build())
            .build();

    OpenTelemetrySdk openTelemetry =
        OpenTelemetrySdk.builder()
            .setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();

    PeriodicMetricReaderFactory periodicMetricReaderFactory =
            PeriodicMetricReader.create(
                OtlpGrpcMetricExporter.builder()
                    .setEndpoint("https://collector-service:4317")
                    .build(),
                Duration.ofMillis(1000));

    SdkMeterProvider sdkMeterProvider =
            SdkMeterProvider.builder()
                .registerMetricReader(periodicMetricReaderFactory)
                .buildAndRegisterGlobal();
 }
}
```

这段代码：

- 使用 OTLP 将 Span 导出到 `collector-service`
  - 使用 BatchSpanProcessor。
- 使用 OTLP 将指标导出到 `collector-service`
  - 使用 PeriodicMetricReader。
- 使用 ParentBased(AlwaysOn) 采样器。
- 使用标准的随机 ID。
- 使用默认资源（Resource），该资源仅包含 SDK（或我们决定包含在核心 SDK 中的任何其他资源，不包括扩展）。
- 使用默认时钟（Clock），它使用 Java 8 / 9+ 优化的 API 来获取时间。
  - 用户设置此项的唯一真正原因是用于单元测试，而不是用于生产。
- 强制执行与属性数量等相关的默认数值限制。
- 启用单一的 w3c 传播器。

因为导出通常是最终用户唯一需要配置的方面，所以这是配置 SDK 最简单的 API。

让我们看一个更复杂的例子：

```java
class HelloWorld {
  public static void main(String[] args) {
    Resource resource = Resource.getDefault().merge(CoolResource.getDefault());
    Clock clock = AtomicClock.create();
    SdkTracerProvider tracerProvider =
        SdkTracerProvider.builder()
            .setResource(resource)
            .setClock(clock)
            .addSpanProcessor(CustomAttributeAddingProcessor.create())
            .addSpanProcessor(CustomEventAddingProcessor.create())
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    OtlpGrpcSpanExporter.builder()
                        .setEndpoint("https://collector-service:4317")
                        .setTimeout(Duration.ofSeconds(10))
                        .build())
                    .setMaxExportBatchSize(10000)
                    .build())
            .addSpanProcessor(
                SimpleSpanProcessor.create(
                    ZipkinSpanExporter.builder()
                        .setEndpoint("https://zipkin-service:9411")
                        .build()))
            .setSampler(Sampler.traceIdRatioBased(0.5))
            .setSpanLimits(SpanLimits.builder().setMaxNumberOfAttributes(10).build())
            .setIdGenerator(TimestampedIdGenerator.create())
            .build();

    OpenTelemetrySdk openTelemetry =
        OpenTelemetrySdk.builder()
            .setPropagators(
                ContextPropagators.create(
                    TextMapPropagator.composite(
                        W3CTraceContextPropagator.getInstance(),
                        B3Propagator.injectingSingleHeader())))
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();

    OpenTelemetrySdk openTelemetryForJaegerBackend =
        OpenTelemetrySdk.builder()
            .setPropagators(ContextPropagators.create(JaegerPropagator.getInstance()))
            .setTracerProvider(tracerProvider)
            .build();

    PeriodicMetricReaderFactory periodicMetricReaderFactory =
        PeriodicMetricReader.create(
                OtlpGrpcMetricExporter.builder()
                    .setEndpoint("https://collector-service:4317")
                    .build(),
                Duration.ofMillis(1000));
    SdkMeterProvider meterProvider =
        SdkMeterProvider.builder()
            .setResource(resource)
            .setClock(clock)
            .registerView(
                InstrumentSelector.builder()
                    .setInstrumentType(InstrumentType.COUNTER)
                    .build(),
                View.builder().build())
            .registerMetricReader(periodicMetricReaderFactory)
            .buildAndRegisterGlobal();
 }
}
```

这配置了资源、时钟、导出器、Span 处理器、传播器、采样器、追踪限制和指标。它配置了两个具有不同传播器的 SDK。遗憾的是，我们无法实现只设置一次属性的目标——`Resource` 和 `Clock` 在信号之间共享，但被重复配置。另一种选择是将所有设置扁平化到 `OpenTelemetrySdkBuilder` 上——这有一个缺点，即它使得为 SDK 的每个配置创建新的 tracer / meter 提供程序变得非常自然，这可能导致资源（如线程、工作器、TCP 连接）的大量重复。因此，这种方式可以更清楚地表明这些信号本身是功能齐全的，而 `OpenTelemetry` 只是一个信号的集合。

请记住，重新配置 `Clock` 预计将是一个极其罕见的操作。

另一件需要记住的事情是，应用程序开发人员很少会做到这一步，我们可以预期实际上是框架开发人员在使用 SDK 的完整配置功能，很可能是通过将其绑定到一个单独的配置系统来实现的。

### 为什么要构建实例

我们发现，即使在用户采用的早期阶段，用户也希望构建 SDK 的实例。

- 与依赖注入（如 Spring）集成，方式与许多其他库类似。
- 允许在同一个应用程序中拥有多个实例，用于多关注点的单类加载器场景。
- 允许管理 SDK 的生命周期，例如，随着无服务器运行时的生命周期一起关闭和启动。

### 在框架中配置 SDK

Java 应用程序非常普遍地使用依赖注入框架编写，如 Spring、Guice、Dagger、HK 等。它们都遵循非常相似的模式。

```java
@Component
public class OpenTelemetryModule {
    @Bean
    public Resource resource() {
        return Resource.getDefault().merge(CoolResource.getDefault());
    }

    @Bean
    public Clock otelClock() {
        return AtomicClock.create();
    }

    @Bean
    public TracerSdkProvider tracerProvider(Resource resource, Clock clock, MonitoringConfig config) {
        return SdkTracerProvider.builder()
            .setResource(resource)
            .setClock(clock)
            .addSpanProcessor(CustomAttributeAddingProcessor.create())
            .addSpanProcessor(CustomEventAddingProcessor.create())
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    OtlpGrpcSpanExporter.builder()
                        .setEndpoint(config.getOtlpExporter().getEndpoint())
                        .setTimeout(config.getOtlpExporter().getTimeout())
                        .build())
                    .setBatchQueueSize(config.getOtlpExporter().getQueueSize())
                    .setExporterTimeout(config.getOtlpExporter().getTimeout())
                    .build())
            .addSpanProcessor(
                SimpleSpanProcessor.create(
                    ZipkinSpanExporter.builder()
                        .setEndpoint(config.getZipkinExporter().getEndpoint())
                        .build()))
            .setSampler(
                config.getSamplingRatio() != 0
                    ? Sampler.traceIdRatioBased(config.getSamplingRatio())
                    : Sampler.parentBased(Sampler.alwaysOn()))
            .setSpanLimits(
                SpanLimits.builder().setMaxNumberOfAttributes(config.getMaxSpanAttributes()).build())
            .setIdGenerator(TimestampedIdGenerator.create())
            .build();
    }

    @Bean
    public MeterSdkProvider meterProvider(Resource resource, Clock clock) {
        return SdkMeterProvider.builder()
            .setResource(resource)
            .setClock(clock)
            .registerView(
                InstrumentSelector.builder()
                    .setInstrumentType(InstrumentType.COUNTER)
                    .build(),
                View.builder().build())
            .build();
    }

    @Bean
    public MeterSdkProvider meterProvider(Resource resource, Clock clock, PeriodicMetricReaderFactory periodicMetricReaderFactory) {
        return SdkMeterProvider.builder()
            .setResource(resource)
            .setClock(clock)
            .registerView(
                InstrumentSelector.builder()
                    .setInstrumentType(InstrumentType.COUNTER)
                    .build(),
                View.builder().build())
            .registerMetricReader(periodicMetricReaderFactory)
            .build();
    }

    @Bean
    public OpenTelemetry openTelemetry(SdkTracerProvider tracerProvider, SdkMeterProvider meterProvider) {
        GlobalMeterProvider.set(meterProvider);
        return OpenTelemetrySdk.builder()
            .setPropagators(
                ContextPropagators.create(
                    TextMapPropagator.composite(
                        W3CTraceContextPropagator.getInstance(),
                        B3Propagator.injectingSingleHeader())))
            .setTracerProvider(tracerProvider)
            .buildAndRegisterGlobal();
    }

    @Bean
    @ForJaeger
    public OpenTelemetry openTelemetryJaeger(SdkTracerProvider tracerProvider, SdkMeterProvider meterProvider) {
        return OpenTelemetrySdk.builder()
            .setPropagators(ContextPropagators.create(JaegerPropagator.getInstance()))
            .setTracerProvider(tracerProvider)
            .build();
    }

    @Bean
    public AuthServiceStub authService(@ForJaeger OpenTelemetry openTelemetry, AuthConfig config) {
        return AuthServiceGrpc.newBlockingStub(ManagedChannelBuilder.forEndpoint(config.getEndpoint()))
          .withInterceptor(TracingClientInterceptor.create(openTelemetry));
    }

    @Bean
    public ServletFilter servletFilter(OpenTelemetry openTelemetry) {
        return TracingServletFilter.create(openTelemetry);
    }
}

// 使用一些已插桩的客户端
@Component
public class MyAuthInterceptor {

  private final AuthServiceStub authService;

  @Inject
  public MyAuthInterceptor(AuthServiceStub authService) {
    this.authService = authService;
  }

  public void doAuth() {
    if (!authService.getToken("credential").isAuthenticated()) {
      throw new HackerException();
    }
  }
}

// 直接使用 tracer，不太常见
@Component
public class MyService {

  private final Tracer tracer;
  private final Meter meter;

  @Inject
  public MyService(TracerProvider tracerProvider, MeterProvider meterProvider) {
    tracer = tracerProvider.get("my-service");
    meter = meterProvider.get("my-service");
  }

  public void doLogic() {
    Span span = tracer.spanBuilder("logic").startSpan();
    try (Scope ignored = span.makeCurrent()) {
      Thread.sleep(1000);
    } finally {
      span.end();
    }
  }
}
```

## SDK 的全局实例

构建的实例在大多数 Java 应用程序中都很方便使用，因为有依赖注入。因为它具有易于推理的初始化顺序，并绑定到依赖顺序中（即使依赖注入是通过构造函数调用手动完成的），我们鼓励应用程序开发人员只使用它。

然而，在某些边缘情况下，无法注入实例。著名的例子是 MySQL——MySQL 拦截器是通过调用默认构造函数初始化的，无法传递已构建的信号提供程序实例。对于这种情况，我们必须将 SDK 实例存储到全局变量中。预计框架或最终用户会将 SDK 设置为全局变量，以支持需要此功能的仪表化。

在设置 SDK 之前，访问全局 `OpenTelemetry` 将返回一个无操作的 `DefaultOpenTelemetry`。这是因为库仪表化可能正在使用全局变量，甚至可能在处理请求期间（而不仅仅是在初始化时）使用它。因此，我们不能抛出异常。相反，如果在类路径上检测到 SDK，我们将记录一次 `SEVERE` 警告，表明 API 在 SDK 配置之前已被访问，并提供有关用户如何解决该问题的说明。SDK 必须在应用程序的早期进行配置，以确保它适用于应用程序中的所有逻辑，这通常由配置框架（如 Spring）来保证。对于应用程序开发人员来说，在绝大多数情况下，此限制应该不会产生任何影响。

MySQL 是唯一已知的需要全局 SDK 实例的边缘情况。如果不存在这种边缘情况，我们甚至可能根本不支持它。

不过，请参阅下面关于 Java Agent 的特别说明。

## SDK 组件内的遥测

SDK 组件，如导出器或远程采样器，可能希望发出遥测数据以供自身处理。但是，SDK 组件必须在 SDK 完全构建之前初始化。我们不支持部分构建的 SDK，因为无法推理其行为。同样，我们也不支持在构建之前使用 SDK 的全局实例。因此，需要 `OpenTelemetry` 的 SDK 组件必须延迟接受它。这是一个限制，但鉴于此类组件很少由应用程序开发人员开发，通常由框架作者或 OpenTelemetry 维护者开发，因此该限制被认为是合理的。

如果此机制内置于 SDK 中，它可能看起来像这样：

```java
interface OpenTelemetryComponent {
  default void setOpenTelemetry(OpenTelemetry openTelemetry) {}
}
interface SpanExporter extends OpenTelemetryComponent {
}
public class BatchExporter implements SpanExporter {

  private volatile Tracer tracer;

  @Override
  public void setOpenTelemetry(OpenTelemetry openTelemetry) {
        tracer = openTelemetry.getTracerProvider().get("spanexporter");
  }

  @Override
  public void export() {
        Tracer tracer = this.tracer;
        if (tracer != null) {
          tracer.spanBuilder("export").startSpan();
        }
  }
}
public class OpenTelemetrySdkBuilder {
    public OpenTelemetrySdkBuilder addSpanExporter(SpanExporter exporter) {
        tracerProvider.addSpanExporter(exporter);
        components.add(exporter);
    }

    public OpenTelemetrySdkBuilder setSampler(Sampler sampler) {
        tracerProvider.setSampler(sampler);
        components.add(sampler);
    }

    public OpenTelemetry build() {
        OpenTelemetrySdk sdk = new OpenTelemetrySdk(tracerProvider.build(), meterProvider.build());
        for (OpenTelemetryComponent component : components) {
          component.setOpenTelemetry(sdk);
        }
    }
}
```

框架作者将会更加轻松，因为大多数依赖注入框架本身就支持延迟注入。

```java
@Component
public class MonitoringModule {

    @Bean
    @ForSpanExporter
    public Tracer tracer(TracerProvider tracerProvider) {
    return tracerProvider.get("spanexporter");
    }
}

@Component
public class MyExporter implements SpanExporter {

    private Lazy<Tracer> tracer;

    @Inject
    public MyExporter(@ForSpanExporter Lazy<Tracer> tracer) {
    this.tracer = tracer;
    }

    @Override
    public void export() {
    tracer.get().spanBuilder("export").startSpan();
    }
}
```

## OpenTelemetry 配置的不可变性

上述内容试图避免允许已构建的 SDK 被修改，即它是浅层不可变的。允许修改会使代码更难推理（任何组件，即使在业务逻辑深处，也可以毫无阻碍地更新 SDK），可能会降低性能（大多数操作需要 volatile 读取），并且如果实现不当会产生线程安全问题。特别是，与撰写本文时的状态相比：

- `TracerSdkManagement.addSpanProcessor` 不需要了。我们需要一个可变的 SDK 来允许 Span 处理器使用全局 API 进行遥测，但因为我们改为将处理 [SDK 组件内的遥测](#SDK 组件内的遥测) 的复杂性推给这些组件（维护者将拥有更多领域知识），所以允许从最终用户 API 中删除此修改器方法。

- `TracerSdkManagement.updateTraceConfig` - 我们不应允许在顶层替换配置，而应考虑将 `TraceConfig` 设为接口，SDK 默认实现是一个始终返回常量配置的简单实现。这使得不可变性的上述好处在不需要动态更新的常见情况下得以保留。在需要动态更新的地方，可以用可变实现替换它，而不是使 SDK 配置可变。这将更新方法排除在最终用户 API 之外，并且通常会给框架开发人员更多的控制权，让他们自己处理动态性，而不会让最终用户有机会对其产生负面影响。例如，spring-boot 可以将追踪配置更新连接到 actuator（其管理接口），而不必担心业务逻辑通过调用 `updateTraceConfig` 绕过此机制。

一些可能由可变性导致的极易出错的代码：

```java
class SleuthUsingService {

  @Inject
  private OpenTelemetry openTelemetry;

  public void doLogic() {
    // 我的逻辑很重要，所以总是对其采样！
    OpenTelemetrySdk.getTracerManagement().updateTraceConfig(config -> config.setSampler(ALWAYS_ON));
    // 这项服务能够影响其他服务，即使 Sleuth 打算“管理 SDK”。
    // 与 javaagent 不同，它无法阻止访问我们可能提供的 SDK 方法。
    doSampledLogicWhileOtherServicesAlsoGetSampled();
  }
}
```

## 库的仪表化

由于可观测性的配置包含在 `OpenTelemetry` 实例中，因此预计库仪表化在配置可观测性时接受一个 `OpenTelemetry` 实例（通常作为其例如追踪拦截器的构建器参数）。

## 自动配置

上述内容介绍了 SDK 的编程式配置，并建议核心 SDK 没有其他配置机制。没有 SPI，也不处理环境变量或系统属性。有许多配置机制，例如 Spring Boot。如果我们考虑在核心 SDK 之上的一层进行自动配置，与这些系统的集成将变得更容易推理。

### Java 自动插桩代理 (Java Auto-Instrumentation Agent)

Java 自动插桩代理是自动配置 SDK 的主要手段。它包含系统属性、环境变量和 SPI，允许用户仅通过应用代理即可拥有完全设置好的追踪配置。实际上，代理甚至不允许用户直接使用 SDK，而是主动阻止它。与其出现核心 SDK 中有一些自动配置而代理中有一些的情况，不如将其全部移至代理中。代理已经公开了导出器 SPI——它还可以公开用于自定义上述手动配置的 SDK 组件的 SPI。

- 我们也可以考虑拥有一个非常相似的自动配置包装器工件作为 SDK 扩展。但我们会假设核心 SDK 始终是手动配置的。

为了允许代理用户将追踪应用于他们自己的代码，代理应尝试对依赖注入进行插桩，以提供使用代理配置的 SDK 的 `OpenTelemetry` 实例，例如它应将其添加到 Spring `ApplicationContext` 中。但是，对于不可用依赖注入的情况，别无选择，只能通过全局变量提供对 SDK 的访问。我们可以预期，即使删除了代理并使用了不同的配置机制（如上述手动配置或 Spring Sleuth），这种用法仍能正常工作。

### SDK 自动配置包装器

对于非代理用户，我们仍然可以提供一种非编程式的解决方案来配置 SDK——它可以是一个不同的工件，包含类似于我们目前拥有的 SPI，支持环境变量和其他自动配置。单一入口点方法 `initialize()` 可以确定配置，初始化 `OpenTelemetry`，并将其设置为全局变量。由于此工件在我们的控制之下，`opentelemetry-api` 检查类路径是否存在包装器并自动调用它是合理的。

### Spring Sleuth

[Spring Sleuth](https://spring.io/projects/spring-cloud-sleuth)（或任何类似的可观测性感知服务器框架，如 [curio-server-framework](https://github.com/curioswitch/curiostack/blob/master/common/server/framework/src/main/java/org/curioswitch/common/server/framework/monitoring/MonitoringModule.java) 或公司 devops 团队开发的内部框架）也是一种自动配置 SDK 的机制。通常，我们预计 Sleuth 用户不会使用 java agent。

上面使用 `@Bean` 的示例展示了 Sleuth 如何工作。特别是，我们希望它拥有自己的一套配置属性——通过确保我们不在核心 SDK 中实现配置属性，而只在代理或可能的配置包装器等配置层中实现，我们避免了因拥有重复变量而产生混淆的可能性（实际上，OpenTelemetry 命名可能会被忽略并被 Spring 命名覆盖）。

## 部分 SDK

我们允许在不使用我们的 SDK 的情况下实现 OpenTelemetry API 的特定信号。例如，MeterProvider 可以用 micrometer 实现。因此，每个信号也必须以例如 `TracerSdkProviderBuilder` 的形式呈现其所有选项。我们预计绝大多数用户会使用 `OpenTelemetrySdkBuilder`——虽然与信号提供程序构建器有一些重复，但这正是维护者可以做的工作，以便为使用整个 SDK 的最常见用例提供最简单的接口。

如果没有 SPI，初始化部分 SDK 的方法是使用 `DefaultOpenTelemetry`。

```java
@Bean
public OpenTelemetry openTelemetry() {
  return DefaultOpenTelemetry.builder()
    .setTraceProvider(TracerSdkProvider.builder().build())
    .setMeterProvider(MicrometerProvider.builder().build())
    .build();
}
```

由于这应该是一个相当次要的用例，并且通常由框架开发人员处理，这似乎是合理的。我们也可以希望，在重要的地方，由部分 SDK 的作者提供一站式入口点。

```java
@Bean
public OpenTelemetry openTelemetry() {
  return OpenTelemetrySdkWithMicrometer.builder()
    .addSpanExporter()
    .setMeterRegistry()
    .build();
}
```

## 考量过的替代方案

### 始终允许使用全局 OpenTelemetry

我们讨论了 `OpenTelemetry` 不可变的一些[优点](#OpenTelemetry 配置的不可变性)。这一决定的主要副作用之一是不允许在配置之前使用全局变量。另一种方法可能是从一个在配置时发生突变的核心开始，即使在配置之前进行了引用，全局使用仍然有效。生命周期变得难以推理，即 `OpenTelemetry` 何时真正准备好使用？依赖注入使其明确，而全局变量则不然。对于不太常见的最终用户用例（动态配置）或非最终用户可以处理的原因（遥测内的遥测），这似乎也有性能影响。

### OpenTelemetry 组件的 SPI 加载

我们可以使用 SPI 检测 OpenTelemetry 组件，但我们预计部分 SDK 不会那么常见。与其采用在我们的代码中初始化部分 SDK 的由内而外的方法，不如鼓励一种由外而内的方法，即创建一个特定于部分 SDK 的包装器。这减少了配置 `OpenTelemetry` 中的魔法，一切都通过我们的单一入口点发生。
