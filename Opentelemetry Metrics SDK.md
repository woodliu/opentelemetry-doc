## Metrics SDK

[TOC]

### 目标

本文档包含两个部分，在第一部分中列出了实现OpenTelemetry Metric SDK的默认需求。实现者需要根据这些规则来实现OpenTelemetry API。

在第二部分中，以OpenTelemetry-Go Metric SDK为例描述了SDK模型的架构细节，给出所需要实现的内容，以此作为对实现者的指导，而不需要强制跨语言精确地去复制这种模型体系结构。

### 期望

SDK实现者在实现OpenTelemetry API时应该遵守该语言的最佳实践和运行时环境。实现者应该遵循 [OpenTelemetry library guidelines](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md)给出的有关安全和性能的规定。

### SDK 术语

Metrics SDK提供了一种Metrics API的实现，包含如下术语(**本文后续将直接采用如下术语，不作翻译**)：

- **Meter**：支持OpenTelemetry Metric API的接口，与Resources 和Instrumentation Library绑定在一起
- **MeterProvider**：通过给定的Instrumentation Library获取**Meter**实例的接口

这些术语用于描述API和SDK的边界，但它们都是API级别的结构。我们可以使用API级别的术语来描述SDK和API之间的边界，但从API的角度看，SDK是不透明的，且没有结构。

本文给出了默认OpenTelemetry SDK的主要组件的内部结构，通过术语来解释每个组件在将metric数据从输入(API级别的事件)导出到输出(metric呈现格式)中所扮演的角色。我们使用**Export Pipeline**来描述SDK级别的功能。

 export pipeline中有三个数据流会经过的主要组件，按顺序为：

1. **Accumulator**：接收API通过Instrument获取的metric事件，并根据活动的Instrument和Label Set对计算出一个Accumulation 
2. **Processor**：从Accumulator接收Accumulations ，并转换为ExportRecordSet
3. **Exporter**：接收ExportRecordSet，并转换为某种协议(如grpc)，将其发送出去

**Controller** 组件在export pipeline中协调Accumulator、Processor和Exporter。

Metrics API规范定义了如下术语：

- **Metric Instrument**：开发者用于操作工具的API对象
- **Synchronous Instrument**：用户通过应用程序上下文调用的metric Instrument
- **Asynchronous Instrument**：通过从SDK的回调调用的metric Instrument
- **Metric Descriptor**：描述一个metric Instrument
- **Metric Event**：单个记录到或观察到的(Instrument, Label Set, Measurement)
- **Collection Interval**: Accumulator.Collect()调用的周期
- **Label**: 描述metric Event属性的key-value
- **Label Set**: 包含唯一的keys的key-values集
- **Measurement**: 来自synchronous instrument的证书或浮点数
- **Observation**: 来自asynchronous instrument的证书或浮点数

[Resource SDK](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/sdk.md) 定义了如下术语：

- **Resource**: 描述进程的一组具有唯一keys的key-value集
- [**Instrumentation Library**](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumentation-library)：与一个工具包关联的名称和版本

下面为架构中重要的数据类型：

- **Aggregator**: 以一种有用的方式汇总一个或多个measurements 
- **Aggregator Snapshot**: 在采集期间拷贝synchronous instrument aggregator的副本
- **AggregatorSelector**: 选择分配给metric instrument 的Aggregator
- **Accumulation**: 包含Instrument, Label Set, Resource, 和Aggregator snapshot, 由Accumulator生成
- **Aggregation**: 由特定的aggregator聚合一个或多个事件产生的结果,  由Processor生成
- **AggregationKind**: 描述了Aggregation支持的API类型 (e.g., Sum)
- **ExportKind**: Delta, Cumulative, 或Pass-Through的一种
- **ExportKindSelector**: 为一个metric Instrument选择ExportKind 
- **ExportRecord**: 包含Instrument, Label Set, Resource, Timestamp(s), 和Aggregation
- **ExportRecordSet**: 一些列的export records.

术语**SDK instrument** 指Instrument的底层实现。

### 数据流图表

从外部看， Metrics SDK 实现了`Meter` 和`MeterProvider`接口，从内部看， Metrics SDK为每个metric数据封装了一个export pipeline，包含四个重要组件。

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201021184836065-657657100.png)

Accumulator组件是将metric event并发地传递给Aggregator的地方，这是负责SDK性能的组件。Accumulator 负责bound 和unbound Instrument，更新、同步、拷贝Aggregator 状态、调用Observer Instrument以及唤醒采集周期。

Processor 组件是exporter pipeline中**可定制**的组件。Processor 负责通过一个独立的`AggregationSelector`接口为特定的Instrument选择Aggregators ，用于减少维数，以及用于在DELTA和CUMULATIVE数据表示之间进行转换。Processor 接口支持任意协议独立的数据转换，且可以将Processor连接在一起，形成更复杂的export pipelines。

Exporter 组件将处理的数据转换为特定的协议，并将其转发到某处。根据 [library guidelines](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md)，exporter应该包含最小的功能，定制时最好通过Processors来表示。

Controller 组件协调在一个采集间隔内的采集动作，处理和导出采集间隔内的metric数据，确保对export pipeline的访问是同步的。

### 要求

下面列出了针对OpenTelemetry SDK主要组件的要求。

上面展示了一个抽象的数据流图，将本文档使用的标准组件名与数据路径做了映射。上图并非要说明SDK的整体架构，仅命名了一个export pipeline的过程，并将组件放到了上下文中。

SDK要求这些组件使用标准名称，这样做有助于在OpenTelemetry中构建一致性。每个SDK都应该包含这些组件，以及下面列出的接口，尽管每个SDK的实际组织可能会因可用的库和源语言的性能特征而有所不同。例如，一个SDK可能为每个instrument实现了一个Accumulator，或者可以为每个采集周期使用一个Accumulator（假设支持多采集周期）。

#### SDK

SDK封装了OpenTelemetry Metric export pipeline，实现了[Meter接口](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#meter-interface)，并管理SDK instrument，Resource和Instrumentation Library 元数据。

##### MeterProvider

###### Shutdown

该方法为provider提供了一种清理环境的方法。

每个MeterProvider实例只能调用一次`Shutdown`，在调用`Shutdown`之后，将不允许获得`Meter`。对于这些调用，SDK应该返回一个有效的无操作Meter。

`Shutdown`应该提供一种方式来让调用者知道是否成功，失败或超时。

当出现超时时，`Shutdown`应该结束或中止。`Shutdown`可以被实现为一个阻塞API，或异步API，通过回调或事件通知调用者。语言库作者可以决定是否配置shutdown的超时时间。

##### SDK：Instrument注册

OpenTelemetry SDK负责确保单个Meter的实现不会报告具有相同名称但不同定义的多个Instrument。为了实现该要求，如果一个Meter已经注册了一个相同名称的metric，则SDK必须拒绝在该Meter上重复注册该Instrument。该要求适用于尝试注册一个具有相同定义的Instrument。我们假设单个instrumentation library可以使用单个Instrument定义，而不是依赖SDK的重复注册。

不同的Meter具有不同的instrumentation library名称，允许在不同的instrumentation library中注册相同名称的Instrument，这种情况下，SDK必须将它们认为是不同的Instruments。

SDK负责实现API规范中包含的metric名称的语法要求。

##### SDK: RecordBatch() 函数

TODO: *Add functional requirements for RecordBatch()*.

##### SDK: Collect() 函数

SDK负责实现`Collect()`函数，该函数会调用一个或多个Accumulators。本规范刻意避免在SDK和Accumulator之间建立特定关系；还有一个实现细节，即一个SDK是否会维护一个Accumulator，或每个Instrument对应一个Accumulator，或介于两者之间的某些配置。

SDK `Collect()`函数必须通过活动的synchronous instruments以及所有注册的asynchronous instruments的Accumulations来调用Processor。

SDK必须允许在评估asynchronous instrument回调期间使用synchronous metric instruments。但通过asynchronous instrument回调来使用synchronous instruments是有副作用的，这种情况下，SDK应该在处理asynchronous instrument之后再处理synchronous instruments，这样synchronous measurements会作为asynchronous observations采集间隔的一部分进行处理。

#### Accumulator

下图展示了API和Accumulator之间的关系，以及synchronous instruments的细节。

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201022161455731-746288535.png)

对于一个synchronous instrument，Accumulator会：

1. 将每个活动的Label Set 映射到一条记录中，该记录由两个相同类型的Aggregator实例组成
2. 在映射中输入新记录，如果需要，调用AggregationSelector
3. 更新当前的Aggregator 实例，用以响应并发API事件
4. 在当前Aggregator 实例上调用Aggregator.SynchronizedMove：a)拷贝其值到Aggregator快照实例中；b)将当前的Aggregator重置为0状态
5. 调用Processor.Process。处理每一个生成的Accumulation (即Instrument, Label Set, Resource, 和Aggregator snapshot)

Accumulator必须提供选项来将一个[Resource](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/sdk.md)与它生成的Accumulations进行关联。

Synchronous metric instruments可以同步使用，除非源语言不支持该特性。SDK Accumulator 组件应该注意同步产生的性能压力。

Accumulator可能会使用排他锁定来维护synchronous  instruments的更新。在调用Aggregator 时，Accumulator不应该持有排他锁，因为Aggregators可能具有更高的同步期望。

##### Accumulator: Collect()函数

Accumulator必须实现Collect方法来为活动的instruments构建和处理当前的Accumulation值，即在上一次采集之后会更新这些值。Collect方法必须调用Processor来处理对应调用之前的所有metric events的Accumulations。

必须在Collect期间使用Aggregator上的*同步移动*操作来计算Accumulations。该操作会同步拷贝当前的Aggregator，并将其重置为0状态，这样Aggregator会在当前采集周期处理结束之后，下次采集周期开始时立即开始累加事件。一个Accumulation 定义为同步拷贝的Aggregator与Label Set, Resource, 和metric Descriptor的组合。

TODO: *Are there more Accumulator functional requirements?*

#### Processor

TODO *Processor functional requirements*

#### Controller

TODO *Controller functional requirements*

#### Aggregator

TODO *Aggregator functional requirements*

如果可能的话，Sum Aggregator应该使用原子操作。

### 模型实现

本模型实现基于[OpenTelemetry-Go SDK](https://github.com/open-telemetry/opentelemetry-go)，本节为实现者提供指南。

#### Accumulator: Meter实现

为了构造一个Accumulator，需要提供Processor和options

```go
// NewAccumulator constructs a new Accumulator for the given
// Processor and options.
func NewAccumulator(processor export.Processor, opts ...Option) *Accumulator

// WithResource sets the Resource configuration option of a Config.
func WithResource(res *resource.Resource) Option
```

Controller会使用`Collect()`方法来调用Accumulator，对采集进行协调。

```go
// Collect traverses the list of active instruments and exports
// data.  Collect() may not be called concurrently.
//
// During the collection pass, the Processor will receive
// one Process(Accumulation) call per current aggregation.
//
// Returns the number of accumulations that were exported.
func (m *Accumulator) Collect(ctx context.Context) int
```

##### 实现与用户级别的Metric API匹配的SDK级别的API

该接口位于SDK和API的边界处，包含3个方法：

```go
// MeterImpl is the interface an SDK must implement to supply a Meter
// implementation.
type MeterImpl interface {
        // RecordBatch atomically records a batch of measurements.
        RecordBatch(context.Context, []label.KeyValue, ...Measurement)

        // NewSyncInstrument returns a newly constructed
        // synchronous instrument implementation or an error, should
        // one occur.
        NewSyncInstrument(descriptor Descriptor) (SyncImpl, error)

        // NewAsyncInstrument returns a newly constructed
        // asynchronous instrument implementation or an error, should
        // one occur.
        NewAsyncInstrument(
                descriptor Descriptor,
                runner AsyncRunner,
        ) (AsyncImpl, error)
}
```

这些方法覆盖了实现OpenTelemetry Metric API所需的所有入口。

[`RecordBatch`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#recordbatch-calling-convention)是一个直接由Accumulator实现的用户级别的API。另外两个instrument构造器用于创建同步和异步SDK instruments。

用户通常对[Metric API `Meter` 接口](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#meter-interface)感兴趣，该接口通过[Metric API `MeterProvider` 接口](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#meter-provider)获得。可以通过封装SDK `Meter`实现来构建`Meter`接口。

```go
// WrapMeterImpl constructs a named `Meter` implementation from a
// `MeterImpl` implementation.  The `instrumentationName` is the
// name of the instrumentation library.
func WrapMeterImpl(impl MeterImpl, instrumentationName string, opts ...MeterOption) Meter
```

该方法的选项：

- 可以为命名的Meter添加instrumentation library版本

instrument注册：

```go
// NewUniqueInstrumentMeterImpl returns a wrapped metric.MeterImpl with
// the addition of uniqueness checking.
func NewUniqueInstrumentMeterImpl(impl metric.MeterImpl) metric.MeterImpl {
```

##### 访问instrument descriptor

API `Descriptor` 方法根据API级别的构造定义了instrument ，包括名称，instrument类型，描述和度量单位。在实现SDK时必须提供方式来访问传递给constructor的Descriptor。

```go
// InstrumentImpl is a common interface for synchronous and
// asynchronous instruments.
type InstrumentImpl interface {
        // Descriptor returns a copy of the instrument's Descriptor.
        Descriptor() Descriptor
}
```

###### Synchronous SDK instrument

[Synchronous SDK instrument](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#synchronous-instrument-details) 支持直接和绑定调用规范。

```go
// SyncImpl is the implementation-level interface to a generic
// synchronous instrument (e.g., ValueRecorder and Counter instruments).
type SyncImpl interface {
        // InstrumentImpl provides Descriptor().
        InstrumentImpl

        // Bind creates an implementation-level bound instrument,
        // binding a label set with this instrument implementation.
        Bind(labels []label.KeyValue) BoundSyncImpl

        // RecordOne captures a single synchronous metric event.
        RecordOne(ctx context.Context, number Number, labels []label.KeyValue)
}

// BoundSyncImpl is the implementation-level interface to a
// generic bound synchronous instrument
type BoundSyncImpl interface {

        // RecordOne captures a single synchronous metric event.
        RecordOne(ctx context.Context, number Number)

        // Unbind frees the resources associated with this bound instrument.
        Unbind()
}
```

##### Asynchronous SDK instrument

[Asynchronous SDK instrument](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#asynchronous-instrument-details) 支持单观察者和匹配观测者调用规范。用于执行Observer回调的接口会传递给constructor，这样asynchronous instruments就不需要其他API级别的访问方法。

```go
// AsyncImpl is an implementation-level interface to an
// asynchronous instrument (e.g., Observer instruments).
type AsyncImpl interface {
        // InstrumentImpl provides Descriptor().
        InstrumentImpl
}
```

传递给asynchronous SDK instrument constructor 的异步"runner"接口(AsyncRunner)同时支持[单个](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#single-instrument-observer)和[批量](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#batch-observer)调用规范。

### Export pipeline细节

TODO: define AggregatorSelector, Aggregator, Accumulation, ExportKind, ExportKindSelector, Aggregation, AggregationKind ExportRecord, ExportRecordSet

### Processor细节

TODO: define the Processor interface

#### Basic Processor

TODO: define how ExportKind conversion works (delta->cumulative required, cumulative->delta optional), Memory option (to not forget prior collection state).

#### Reducing Processor

TODO: Label filter, LabelFilterSelector

### Controller 细节

TODO: Push, Pull

### Aggregator Implementations

TODO: Sum, LastValue, Histogram, MinMaxSumCount, Exact, and Sketch.

### Pending issues

#### ValueRecorder instrument default aggregation

TODO: The default SDK behavior for `ValueRecorder` instruments is still in question. Options are: LastValue, Histogram, MinMaxSumCount, and Sketch.

[Spec issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636) [OTEP 117](https://github.com/open-telemetry/oteps/pull/117) [OTEP 118](https://github.com/open-telemetry/oteps/pull/118)

#### Standard sketch histogram aggregation

TODO: T.B.D.: DDSketch considered a good choice for ValueRecorder instrument default aggregation.