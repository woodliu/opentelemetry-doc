## Opentelemetry Trace SDK

[TOC]

### 采样

采样是一种机制，通过减少采集和发送到后端的采样数量来控制由OpenTelemetry引入的噪声和开销。

可以在trace采集的不同阶段实现采样。最早的采样可能发生在创建trace之前，最晚的采样可能发生在流程之外的Collector上。

OpenTelemetry API有两个负责数据采集的属性：

- `IsRecording`：`Span`的字段。如果为`false`，则当前`Span`丢弃了所有的跟踪数据(属性，事件，状态等)。用户可以使用该属性来决定是否避免采集高昂的跟踪数据。[Span Processor](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#span-processor)必须接收该字段为`true`的spans。但是，除非设置了Sampled标志，否则[Span Exporter](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#span-exporter) 不应该接受这些数据。
- `Sampled`：`SpanContext`上的`TraceFlags`中的标志。该标志会通过`SpanContext`传播给子Spans。更多详情，参见[W3C Trace Context specification](https://www.w3.org/TR/trace-context/#sampled-flag)。该标志指出`Span`已经被`sampled`，且将会被导出。[Span Exporters](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#span-exporter)必须接收`Sampled`设置为`true`的spans，且不应该接收设置为`false`的spans。

`SampledFlag == false` 和`IsRecording == true`意味着当前`Span`记录了信息，但子`Span`可能不会。

`SampledFlag == true` 和`IsRecording ==false `意味着分布式上下文发生了分歧，OpenTelemetry API 不允许这种组合。

下表概况了 `IsRecording` 和`SampledFlag`的组合行为：

| `IsRecording` | `Sampled` Flag | Span Processor receives Span? | Span Exporter receives Span? |
| ------------- | -------------- | ----------------------------- | ---------------------------- |
| true          | true           | true                          | true                         |
| true          | false          | true                          | false                        |
| false         | true           | Not allowed                   | Not allowed                  |
| false         | false          | false                         | false                        |

SDK定义了[`Sampler`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampler) 接口，以及一组[内置的samplers](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#built-in-samplers)，并将一个`Sampler`与每个[`TracerProvider`]关联起来。

当创建一个Span时，SDK必须在真正创建span前查询`Sampler`的[`ShouldSample`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#shouldsample)方法，并采取相应措施：参见如何使用[`ShouldSample`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#shouldsample)的返回值来设置Span的`IsRecording`和`Sampled`，以及上表来判定是否将`Span`传给`SpanProcessor`。非记录的span可以使用与在没有安装api实现的情况下创建span的机制来实现(有时称为`NoOpSpan` or `DefaultSpan`)。

### Sampler

`Sampler`接口允许用户创建自定义的Samplers，用于根据信息(通常为`Span`创建前的可用信息)返回一个采样的`SamplingResult`。

#### ShouldSample

返回一个将要创建的`Span`的采样Decision 。

需要的参数：

- [`Context`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/context.md) ：带父`Span`的Context，Span的SpanContext可能是无效的，用于指定根span。
- `TraceId` ：需要创建的`Span`的TraceId。如果父`SpanContext` 包含一个有效的`TraceId`, 则两个span的TraceId必须匹配。
- NamTraceId e：需要创建的`Span`的名称。
- `SpanKind` ：需要创建的`Span`的SpanKind。
- 需要创建的`Span`的初始`Attributes`。
- 需要创建的`Span`的链接的集合。通常用于批量操作，参见 [Links Between Spans](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#links-between-spans).

返回值：

返回一个名为`SamplingResult`的输出，其包含：

- 一个采样`Decision`。为如下枚举之一：
  - `DROP` ： `IsRecording() == false`, span将不会记录，并丢弃所有的事件和属性。
  - `RECORD_ONLY` ： `IsRecording() == true`, 但不能设置`Sampled`标志。
  - `RECORD_AND_SAMPLE` ： `IsRecording() == true` 且必须设置`Sampled` 标志。
- 一组将会添加到Span的span属性，返回的对象必须时不可变的(多个对象可能会返回不同的不可变对象)
- 一个通过新的`SpanContext`与Span关联的`Tracestate`。如果sampler 返回空的`Tracestate`，则`Tracestate`将会被清空，如果samplers 不期望更改它，则通常应该返回传入的Tracestate。

#### GetDescription

返回sampler 名称或配置的简短描述，可能用于显示在debug页面或日志中，例如`"TraceIdRatioBased{0.000100}"`。

描述不能随着时间变更，调用者可以缓存其返回值。

### Built-in samplers

OpenTelemetry 支持大量内置的samplers，默认的samplers为`ParentBased(root=AlwaysOn)`。

#### AlwaysOn

- 总是返回`RECORD_AND_SAMPLE`
- 描述必须是`AlwaysOnSampler`

#### AlwaysOff

- 总数返回DROP
- 描述必须是`AlwaysOffSampler`

#### TraceIdRatioBased

- `TraceIdRatioBased`必须忽略父`SampledFlag`。为了体现父`SampledFlag`，`TraceIdRatioBased`应该作为下面指定的`ParentBased` sampler的委托。
- 描述必须是`TraceIdRatioBased{0.000100}`。

##### `TraceIdRatioBased`  sampler算法的要求：

- 采样算法必须是确定性的。是否对一个给定`TraceId`的trace进行采样不取决于语言，时间等。实现中在决定采样决策时必须对`TraceId`使用确定的哈希。为了保证这点，所有子Span上运行的sampler都必须采用相同的决策。
- 使用给定采样率的`TraceIdRatioBased`  sampler必须同时对低采样率的`TraceIdRatioBased` sampler 使用的所有traces进行采样。当后端系统希望采用比前端系统更高的采样率运行时，这一点很重要，这样将会采样所有前端的trace，而额外的traces仅将在后端上进行采样。

#### ParentBased

- 这是一个组合的sampler，ParentBased用于区分如下场景：
  - 没有父span(根span)
  - 远端父span(`SpanContext.IsRemote() == true`) 且`SampledFlag` 等于`true`
  - 远端父span(`SpanContext.IsRemote() == true`) 且`SampledFlag` 等于`false`
  - 本地父span(`SpanContext.IsRemote() == false`) 且`SampledFlag` 等于`true`
  - 本地父span(`SpanContext.IsRemote() == false`) 且`SampledFlag` 等于`false`

必需参数：

- `root(Sampler)` ： 用于根span的调用

可选参数：

- `remoteParentSampled(Sampler)` (默认: AlwaysOn)
- `remoteParentNotSampled(Sampler)` (默认: AlwaysOff)
- `localParentSampled(Sampler)` (默认: AlwaysOn)
- `localParentNotSampled(Sampler)` (默认: AlwaysOff)

| Parent  | parent.isRemote() | parent.IsSampled() | Invoke sampler             |
| ------- | ----------------- | ------------------ | -------------------------- |
| absent  | n/a               | n/a                | `root()`                   |
| present | true              | true               | `remoteParentSampled()`    |
| present | true              | false              | `remoteParentNotSampled()` |
| present | false             | true               | `localParentSampled()`     |
| present | false             | false              | `localParentNotSampled()`  |

### Tracer Provider

#### Tracer创建

新的Tracer实例的创建需要通过一个`TracerProvider`(参见 [API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#tracerprovider))。`TracerProvider`的`name`和`version`参数必须用于创建一个[`InstrumentationLibrary`](https://github.com/open-telemetry/oteps/blob/master/text/0083-component.md)实例(该实例保存在创建的`Tracer`上)。

配置(即[Span processors](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#span-processor) 和[`Sampler`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampling)))必须由`TracerProvider`管理，且必须提供一些方式来对其进行配置(至少包括`Tracer`的创建和初始化)。

TracerProvider 可能会提供方法来更新配置，如果更新了配置(如，添加了一个`SpanProcessor`)，则更新后的配置必须应用到所有已经返回的`Tracers`(即，无论Tracer是变更之前还是之后从TracerProvider获取到的)。注意：在落实方面，这意味着`Tracer`实例有一个到它们的TracerProvider的引用，且只能通过这个引用来访问配置。

#### ShutDown

该方法为provider提供了一种清理方式。

`ShutDown`只能被每个`TracerProvider`实例调用一次。在调用`ShutDown`之后，后续将不允许获得`Tracer`，这种情况下，SDKs应该返回一个无操作的`Tracers`。

`ShutDown`应该提供一种方式来让调用者知道是否成功，失败还是超时。

`ShutDown`应该在一段时间内结束或中断。`ShutDown`可以被实现为一个阻塞API或通过回调或事件通知调用者的异步API。语言库作者可以决定是否需要可配置shutdown的超时时间。

### 额外的Span接口

[API级别的Span接口定义](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span-operations)只定义了通过只写方式来访问span。这样做是正确的，因为工具和应用不会使用保存的给应用逻辑使用的数据。然而，SDK需要最终在某些位置读取这些数据，因此SDK规范定义了一组类`Span`的参数要求：

- 可读span：接收该参数的函数必须能够访问添加到span的所有的[信息](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span-data-members)。特别要求要能够访问与span相关的`InstrumentationLibrary` 和`Resource` 信息。此外，它必须能可靠地确定Span是否结束(有些语言实现会将结束的时间戳置为null，其他则可能使用一个hasEnded布尔值)。

  接收该参数的函数可能不能修改Span。

  注：通常该功能用于实现一个新的接口或不可变值类型。在一些语言中，SpanProcessors 可能具有不同于exporters的可读span类型(如，一个`SpanData`可能包含一个不可变的快照，而一个`ReadableSpan`接口可能会直接从某些`Span`接口操作的底层数据结构中直接读取信息)

- 读/写span：接收该参数的函数必须同时能够访问完整的span [API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span-operations)，且能够检索所有添加到span的信息(与可读span相同)。

使用此函数调用的函数必须以某种方式获得相同的`Span`实例以及[span创建API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span-creation)返回()或将返回)给用户的类型（例如，`Span`可能是传递给该函数的参数之一，或者可以提供一个getter功能）。

### Span 采集的限制

错误码可能会将意料之外的属性，事件和链接添加到一个span。如果采集是无界的，则可能会很快耗尽内存，导致无法安全恢复的崩溃。

为了防止此类错误，SDK Spans可能会在每次采集时的元素超过1000的限制时丢弃属性，链接和事件。SDKs可能会提供方式来修改此限制。

如果有可配置的限制，SDK应该遵守[SDK环境变量](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/sdk-environment-variables.md#span-collection-limits)指定的环境变量。

应该有日志来让用户知道是否因为上限而导致属性，事件或链接的丢失。为了防止过多的日志记录，不应该为每个span或每个丢弃的属性、事件或链接发出日志。

### Span processor

Span processor是一个接口，用于span start和end方法调用的钩子。只有当[`IsRecording`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#isrecording)为true是才会调用Span processor。

内置的Span processor负责批处理和将span转换为可输出的表达方式，并将批量传给exporters。

可以在SDK的`TracerProvider`上直接注册Span processors，然后按照注册的顺序进行调用。

每个注册到`TracerProvider`上的processor 都作为pipeline(包含span processor和可选的exporter)的一个开始。SDK必须允许使用单独的expoter结束每个pipeline。

SDK必须允许用户实现和配置自定义的processer并为高级场景装饰内置的processors，如标记或过滤。

下图展示了`SpanProcessor`和SDK中其他组件之间的关系：

```
  +-----+--------------+   +-------------------------+   +-------------------+
  |     |              |   |                         |   |                   |
  |     |              |   | Batching Span Processor |   |    SpanExporter   |
  |     |              +---> Simple Span Processor   +--->  (JaegerExporter) |
  |     |              |   |                         |   |                   |
  | SDK | Span.start() |   +-------------------------+   +-------------------+
  |     | Span.end()   |
  |     |              |
  |     |              |
  |     |              |
  |     |              |
  +-----+--------------+
```

#### Interface definition

##### OnStart

当一个span开始时会调用OnStart。该方法在启动span的线程上是同步的，因此不应该阻塞或抛出异常。

参数：

- `span`：提供给开始span的读写span对象。应该保存一个到该span对象的引用，对span的更新应该反映在该对象中。例如，用于在后台线程中创建一个周期性评估/打印关于所有活动的span的信息的SpanProcessor 。
- `parentContext`：SDK确定的span的父`Context`(显式传递的Context，当前`Context`或一个空的`Context`)

返回值：Void

##### OnEnd(Span)

在一个span结束之后(即设置了结束时间戳)会调用`OnEnd`，必须在[`Span.End()` API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#end)中同步调用该方法，因此不应该阻塞或抛出异常。

参数：

- 提供给结束span的读写span对象。注意：解释传入的Span可能在技术上是可写的，由于此时已经结束，因此不允许修改。

返回值：Void

##### Shutdown()

停止processor。该方法会让processor 执行清理操作。

每个`SpanProcessor`实例只能调用一次`Shutdown`。在这之后，不允许调用`OnStart`, `OnEnd`, 或`ForceFlush`。SDK应该忽略这些调用。

`Shutdown`应该提供一种方式来让调用者知道是否成功，失败或超时。

`Shutdown`必须包含`ForceFlush`的效果。

`Shutdown`应该在超时时间内完成或中断。`Shutdown`可以被实现为一个阻塞API或一个通过回调或事件通知调用者的非阻塞API。

##### ForceFlush()

将还没有导出的span导出到配置的`Exporter`上。

`ForceFlush`应该提供一种方式来让调用者知道是否成功，失败或超时。

`ForceFlush`应该只能在明显需要时才会被调用，例如当一些FaaS供应商可能会在调用之后挂起进程，但在此之前需要将完整的span导出去。

`ForceFlush`应该在超时时间内结束或中断。`ForceFlush`可以被实现为一个阻塞API或一个通过回调或事件通知调用者的非阻塞API。

#### 内置的span processors

标准的OpenTelemetry SDK 必须实现simple和batch processors。其他场景下的处理应该优先考虑[OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#collector)中的实现。

##### Simple processor

这是一个`SpanProcessor`的实现。在span结束后，它将传递完成的span，并在完成时以友好的span数据表示形式导出到配置的`SpanExporter`中。

可配置的参数：

- `exporter`：spans推送数据的exporter。

##### Batching processor

这是一个`SpanProcessor`的实现，它会创建一批完成的spans，并将它们以友好的span数据表示形式传递给配置的`SpanExporter`。

可配置的参数：

- `exporter`：spans推送数据的exporter。
- `maxQueueSize`：队列的最大值。当达到最大值后会丢弃spans。默认为2048。
- `scheduledDelayMillis`：两个连续的导出之间的延迟间隔，单位为毫秒，默认5000。
- `exportTimeoutMillis`：在取消之前导出动作的最厂事件, 默认为`30000`.。
- `maxExportBatchSize` : 每次批量导出的最大值。必须小于或等于 `maxQueueSize`. 默认为512。

### Span Exporter

`Span Exporter`定义了必须实现的协议指定的接口，这样就可以以插件形式插入OpenTelemetry SDK，支持发送遥测数据。

该接口的目标时最小化实现协议指定的遥测exporters的负担。协议上的exporter 将主要是一个简单的遥测数据编码器和发送器。

#### 接口定义

exporter 必须支持两种功能：**Export** 和**Shutdown**。在强类型语言中，通常会有两个独立的`Exporter`接口，一个接受span (SpanExporter)，另一个接受metrics (MetricsExporter)。

##### Export(batch)

导出一批可读的spans。协议上的exporters将实现该功能，通常用于序列化数据并将其传输到目的地。

对于相同的exporter实例，不能同时调用Export() 。只有在当前调用返回后才能再次调用该函数。

Export() 必须不能阻塞，且必须有一个合理的上限，超过这个上限后，调用必须超时并产生错误结果(失败)。

exporter需要负责重试逻辑。由于需要的逻辑可能会严重以来指定的协议和发送的span后端，因此默认的SDK不应该实现重试逻辑。

参数：

batch：一组批量的可读spans。确切的数据类型由语言决定，通常时某种列表。如，java中通常为`Collection<SpanData>`。

返回**:** ExportResult:

ExportResult可能为如下之一：

- `Success`：批量导出成功。对于协议上的exporters意味着数据已经发送到网络上，且已经传递给了目的服务。

- `Failuer`：导出失败，必须丢弃这组数据。例如，可能是因为这组span包含错误的数据，且无法被序列化。

注意：可以通过异步机制或回调来返回结果。

##### Shutdown()

停止exporter。用于给exporter执行清理操作。

每个`Exporter`实例只能调用一次`Shutdown`。且后续不允许对`Export`的调用，否则返回`Failuer`结果。

`Shutdown`不应该阻塞(例如，如果在它尝试刷新数据时目标时不可用的)。语言库作者可以决定是否可配置shutdown的超时时间。

#### 进一步的语言规范

基于上面列出的通用接口定义，库作者必须为特定语言定义确切的接口。

鼓励作者在接口边界上使用高效的数据结构，便于协议上的exporters将数据快速格式化为网络格式，并将内存管理器的压力降到最低。后者通常需要了解如何优化快速生成的短期遥测数据结构，更方法特定语言的内存管理器的使用。一般建议尽可能减少分配的数量，并在可能的情况下使用分配区域，从而避免在出现在遥测数据的高速率情况下分配/解除分配/收集操作的爆炸式增长。

##### 例子

下面是特定语言中的`Exporter`接口的样子，仅用于演示。

**Go SpanExporter Interface**

```go
type SpanExporter interface {
    Export(batch []ExportableSpan) ExportResult
    Shutdown()
}

type ExportResult struct {
    Code         ExportResultCode
    WrappedError error
}

type ExportResultCode int

const (
    Success ExportResultCode = iota
    Failure
)
```

**Java SpanExporter Interface**

```java
public interface SpanExporter {
 public enum ResultCode {
   Success, Failure
 }

 ResultCode export(Collection<ExportableSpan> batch);
 void shutdown();
}
```