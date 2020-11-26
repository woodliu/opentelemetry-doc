## OpenTelemetry: 经得起考验的工具

摘自：https://blog.newrelic.com/product-news/what-is-opentelemetry/

[TOC]

### 什么是OpenTelemetry？

OpenTelemetry合并了OpenTracing和OpenCensus项目，提供了一组API和库来标准化遥测数据的采集和传输。OpenTelemetry提供了一个安全，厂商中立的工具，这样就可以按照需要将数据发往不同的后端。

OpenTelemetry项目由如下组件构成：

- 推动在所有项目中使用一致的规范
- 基于规范的，包含接口和实现的APIs
- 不同语言的SDK(APIs的实现)，如 Java, Python, Go, Erlang等
- Exporters：可以将数据发往一个选择的后端
- Collectors：厂商中立的实现，用于处理和导出遥测数据

#### 术语

如果刚接触Opentelemetry，那么需要了解如下术语：

- Traces：记录经过分布式系统的请求活动，一个trace是spans的有向无环图

- Spans：一个trace中表示一个命名的，基于时间的操作。Spans嵌套形成trace树。每个trace包含一个根span，描述了端到端的延迟，其子操作也可能拥有一个或多个子spans。

- Metrics：在运行时捕获的关于服务的原始度量数据。Opentelemetry定义的metric instruments(指标工具)如下。Observer支持通过异步API来采集数据，每个采集间隔采集一个数据。

  | ame               | Synchronous | Adding | Monotonic |
  | ----------------- | ----------- | ------ | --------- |
  | Counter           | Yes         | Yes    | Yes       |
  | UpDownCounter     | Yes         | Yes    | No        |
  | ValueRecorder     | Yes         | No     | No        |
  | SumObserver       | No          | Yes    | Yes       |
  | UpDownSumObserver | No          | Yes    | No        |
  | ValueObserver     | No          | No     | No        |

- Context：一个span包含一个**span context**，它是一个全局唯一的标识，表示每个span所属的唯一的请求，以及跨服务边界转移trace信息所需的数据。OpenTelemetry 也支持**correlation context**，它可以包含用户定义的属性。**correlation context**不是必要的，组件可以选择不携带和存储该信息。
- Context propagation：表示在不同的服务之间传递上下文信息，通常通过HTTP首部。 Context propagation是Opentelemetry系统的关键功能之一。除了tracing之外，还有一些有趣的用法，如，执行A/B测试。OpenTelemetry支持通过多个协议的Context propagation来避免可能发生的问题，但需要注意的是，在自己的应用中最好使用单一的方法。

### OpenTelemetry的好处

通过将OpenTracing 和OpenCensus 合并为一个开放的标准，OpenTelemetry提供了如下便利：

- 选择简单：不必在两个标准之间进行选择，OpenTelemetry可以同时兼容 OpenTracing和OpenCensus。
- 跨平台：OpenTelemetry 支持各种语言和后端。它代表了一种厂商中立的方式，可以在不改变现有工具的情况下捕获并将遥测数据传输到后端。
- 简化可观测性：正如OpenTelemetry所说的"高质量的观测下要求高质量的遥测"。希望看到更多的厂商转向OpenTelemetry，因为它更方便，且仅需测试单一标准。

### 如何使用OpenTelemetry

OpenTelemetry APIs 和SDKs有很多快速使用指南和[文档](https://opentelemetry.io/docs/)帮助快速入门，如[Java快速指南](https://github.com/open-telemetry/opentelemetry-java/blob/master/QUICKSTART.md)展示了如何获取跟踪程序、创建spans、添加属性，以及跨不同spans传递context。

将OpenTelemetry trace APIs插装到应用程序后，就可以使用预先编译好的[OpenTelemetry 库中的exporters ](https://opentelemetry.io/registry/)将trace数据发送到观测平台，如New Relic或其他后端。

metrics和logs的规范仍在开发阶段，但一旦完成，它们将在实现OpenTelemetry的主要目标中发挥重要作用：确保库和框架包含所有内置的遥测数据类型，使开发人员无需进行检测即可提取遥测数据。

### OpenTelemetry 架构组件

由于OpenTelemetry旨在成为一个为厂商和可观察性后端提供的跨语言框架，因此它非常灵活且可扩展，但同时也很复杂。OpenTelemetry的默认实现中，其架构可以分为如下三部分：

1. OpenTelemetry API
2. OpenTelemetry SDK，包括
   - Tracer pipeline
   - Meter pipeline
   - shared Context layer
3. Collector

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023091008244-135880184.png)

#### OpenTelemetry API

应用开发者会使用 Open Telemetry API对其代码进行插桩，库作者会用它(在库中)直接编写桩功能。API不处理操作问题，也不关心如何将数据发送到厂商后端。

API分为四个部分：

1. A Tracer API
2. A Metrics API
3. A Context API
4. 语义规范

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023091114262-375314713.png)

##### Tracer API

Tracer API 支持生成spans，可以给span分配一个`traceId`，也可以选择性地加上时间戳。一个Tracer会给spans打上名称和版本。当查看数据时，名称和版本会与一个Tracer关联，通过这种方式可以追踪生成sapan的插装库。

##### Metric API

[Metric API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md)提供了多种类型的Metric instruments(桩功能)，如*Counters* 和*Observers*。Counters 允许对度量进行计算，Observers允许获取离散时间点上的测量值。例如，可以使用Observers 观察不在Span上下文中出现的数值，如当前CPU负载或磁盘上空闲的字节数。

##### Context API

[Context API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/context.md) 会在使用相同"context"的spans和traces中添加上下文信息，如[W3C Trace Context](https://www.w3.org/TR/trace-context/), [Zipkin B3首部](https://github.com/openzipkin/b3-propagation), 或 [New Relic distributed tracing](https://docs.newrelic.com/docs/understand-dependencies/distributed-tracing/get-started/introduction-distributed-tracing) 首部。此外该API允许跟踪spans是如何在一个系统中传递的。当一个trace从一个处理传递到下一个处理时会更新上下文信息。Metric instruments可以访问当前上下文。

##### 语义规范

OpenTelemetry API包含一组[语义规范](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#semantic-conventions.md)，该规范包含了命名spans，属性以及与spans相关的错误。通过将该规范编码到API接口规范中，OpenTelemetry 项目保证所有的instrumentation(不论任何语言)都包含相同的语义信息。对于希望为所有用户提供一致的APM体验的厂商来说，该功能非常有价值。

#### OpenTelemetry SDK

OpenTelemetry SDK是OpenTelemetry API的实现。该SDK包含三个部分，与上面的API类似：Tracer, 一个Meter, 和一个shared Context layer

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023094415727-1521041046.png)

理想情况下，SDK应该满足99%的标准使用场景，但如果有必要，可以自定义SDK。例如，可以在Tracer pipeline实现中自定义除核心实现(如何与共享上下文层交互)外的其他任何内容，如Tracer pipeline使用的采样算法。

##### Tracer pipeline

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023095457731-1133586441.png)

当配置[SDK](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md)时，需要将一个或多个`SpanProcessors`与Tracer pipeline的实现进行关联。`SpanProcessors`会查看spans的生命周期，然后在合适的时机将spans传送到一个`SpanExporter`。SDK中内置了一个简单的SpanProcessor，可以将完成的spans直接转发给exporter 。

SDK还包含一个批处理实现，按照可配置的间隔分批次转发已完成的spans。但由于`SpanProcessor`的实现可以接受插件，因此可以在完成自己的实现后赋予其自定义的行为。例如，如果遥测后端支持观测"正在进行的"spans，那么可以创建一个SpanProcessor实现，将所有span状态变更涉及的spans转发出去。

Tracer pipeline的最后是`SpanExporter`。一个exporter的工作很简单：将OpenTelemetry 的spans转换为遥测后端要求的表达格式，然后转发给该后端即可。提供定制化的SpanExporter是遥测厂商参与OpenTelemetry生态系统的最简单方式。

##### Meter pipeline

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023100454037-1961919967.png)

##### Meter pipeline

Meter pipeline要远比Tracer pipeline负载，而metrics也远比span复杂。下面的描述基于java SDK实现，可能跨语言会有所不同。

Meter pipeline会创建和维护多种类型的metric工具，包括Counters 和Observers。每个工具的实例都需要以某种方式聚合。默认情况下，Counters通过累加数值进行聚合，而Observers通过采集记录到的最后一个数值进行聚合。所有的工具默认都有一个聚合。

(在本文编写之际，metric工具的自定义聚合配置仍然在起草阶段)。

不同语言的Meter pipeline的实现会有所不同，但所有场景下，metric的聚合结果都会被传递到`MetricExporter`。与spans类似，供应商可以提供自己的exporter，将由metric aggregators生成的聚合数据转换为遥测后端所需的类型。

OpenTelemetry支持两种类型的exporter：基于exporters的"push"，即exporter按照时间间隔将数据发送到后端；基于exporters的"pull"，即后端按照需要请求数据。New Relic 是一个基于push的后端，而Prometheus是一个基于push的后端。

##### shared Context layer

shared Context layer位于Tracer和Meter pipeline之间，允许在一个执行的span的上下文中记录所有非observer的metric。可以使用*propagators*自定义Context，在系统内外传递span上下文。OpenTelemetry SDK提供了一个基于W3C Trace Context规范的实现，但也可以根据需要来包含 Zipkin B3 propagation等。

#### Collector

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201023102816807-91460195.png)

> 下面对collector的描述来自[官方文档](https://opentelemetry.io/docs/collector/about/)

OpenTelemetry Collector提供了一种厂商中立的实现，无缝地接收，处理和导出遥测数据。此外，它移除了为支持发送到多个开源或商业后端而使用的开源可观察性数据格式(如Jaeger，Prometheus等)的运行，操作和维护。

OpenTelemetry collector可以扩展或嵌入其他应用中。下面应用扩展了collector：

- [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)
- [jaeger](https://github.com/jaegertracing/jaeger/tree/master/cmd/opentelemetry)

如果要创建自己的collector发行版，可以参见这篇blog： [Building your own OpenTelemetry Collector distribution](https://medium.com/p/42337e994b63)。

如果要构建自己的发行版，可以使用[OpenTelemetry Collector Builder](https://github.com/observatorium/opentelemetry-collector-builder) 。