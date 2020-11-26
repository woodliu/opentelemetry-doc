## Opentelemetry Metrics API

[TOC]

*注：为了理解的一致性，本文档将使用[SDK规定的术语](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/sdk.md#sdk-terminology)，不做翻译。注意区分Measurements和instrument的区别，前者指的是度量数据，后者是一个工具*

### 概览

OpenTelemetry Metrics API用于捕获计算机程序运行期间的产生的度量数据。Metrics API是专门为处理原始度量数据而设计的，旨在(高效,同步)持续地生成这些度量数据的摘要。下面"API"特指OpenTelemetry Metrics API。

API通过几种不同性能级别的调用规范来提供捕获原始度量数据的功能。无论是哪种调用规范，都将*metric event*定义为捕获新度量时发生的逻辑事件，SDK会将该时刻作为一个隐式的时间戳。

这里的"semantic"或"semantics"指的是，当在API下发生metric event时，如何赋予这些metric event意义。下面将描述一个*标准实现*来帮助理解相关的含义，标准实现会对每种类型的metric event执行聚合操作。

监控和告警通常会使用metric event提供的经聚合以及类型转换后的数据。但metric event还有许多其他用途，如记录聚合或tracing和logging系统中的原始度量数据。为此，[Opentelemetry要求将API和SDK分开](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md#requirements)，这样就可以在运行期间配置不同的SDK。

#### 在没有安装SDK情况下的API行为

在没有安装Metrics SDK的情况下，Metrics API仅包含无操作(no-ops)的功能函数，即所有对API的调用都不会产生任何影响。Meters必须返回指标的无操作实现。从用户的角度看，对这些API的调用将不会产生任何错误，可以直接忽略返回值。当调用这些API时，API不能抛出异常。

#### Measurements

术语"捕获"用于描述当用户给API传入一个度量时产生的动作。捕获的结果取决于配置的SDK，如果没有安装SDK，默认对捕获的事件不做任何响应。这种做法的目的是为了传达"度量时可能发生任何事"，具体取决于SDK，同时也意味着用户已在采用某种度量。出于性能和语义方面的原因，API允许用户在两种度量方法中进行选择(*adding和grouping*)。

术语"adding"用于指定添加某些度量的特征，意味着只有总和才被认为是有意义的信息。这些度量值可以用算术加法组合起来，通常是某个值的实际数量(例如字节数)。

当一组值(也称为总体)被假定为有用的信息时，就可以使用grouping(下称**分组**)度量。一个分组度量要么无法使用算法加法组合(如请求延时)的度量；要么是虽然可以使用加法，但目的是监视值的分布(例如，队列大小)。中值被认为是分组测量的有用信息。

分组instruments在语义上会比adding instruments捕获更多的信息，但在定义上，分组度量要比adding度量开销大。通常用户只需选择adding instruments ，除非希望通过额外的开销来获取更多的信息。

#### Metric Instruments

一个*metric instrument*指API用于捕获原始度量数据的设备。下面列出了标准的instruments，每种instrument都有各自的用途。API有意避免使用改变instrument语义解释的可选特性，而更倾向于通过instruments提供单一方法，且每种方法都有固定的解释。

API捕获的所有度量都与使用度量的instrument相关联，instrument会赋予其语义属性。通过调用`Meter` API可以创建和定义instrument，该API是面向用户的SDK入口点。

通过如下方式类区分不同的Instruments ：

1. 同步性：synchronous instrument由用户在分布式上下文中调用(例如，关联的span、Baggage等)。asynchronous instrument没有上下文，由SDK在每个收集间隔内调用。
2. Adding vs Grouping：一个adding指标用于记录adding度量，与上面描述的grouping instrument相反
3. 单调性：一个单调instrument也是一个adding instrument，计算出的和的级数是非递减的。单调instrument对于监控频率信息很有用。

下面的metric instrument展示了其同步，adding以及/或单调性：

> 下表中的"No"表示该instrument为另外一个属性，如SumObserver的同步(Synchronous)为`No`，则表示它是异步的。

| Name              | Synchronous | Adding | Monotonic |
| ----------------- | ----------- | ------ | --------- |
| Counter           | Yes         | Yes    | Yes       |
| UpDownCounter     | Yes         | Yes    | No        |
| ValueRecorder     | Yes         | No     | No        |
| SumObserver       | No          | Yes    | Yes       |
| UpDownSumObserver | No          | Yes    | No        |
| ValueObserver     | No          | No     | No        |

synchronous instruments对于在分布式环境中(即，关联的span，Baggage等)收集的度量很有用。asynchronous instruments用于周期性采集开销比较大的度量信息。更多参见下文的同步和异步instrument的特性。

同步和异步adding指标有一个明显的差异：synchronous instruments用于捕获sum的变化，而asynchronous instruments直接捕获sum。更多参见下文的[adding指标的特性](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#adding-and-grouping-instruments-compared)。

单调adding instruments非常重要，因为它们支持频率计算。更多参见下文[选择metric指标](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#monotonic-and-non-monotonic-instruments-compared)。

instrument定义描述了instrument的一些属性，包含名称和类型。其他属性则是可选的，包含描述和度量单位。一个instrument描述与它产生的数据相关。

#### 标签(labels)

标签指与metric event相关联的key-value属性，与tracing API的Span属性类似。每个标签都会对metric event进行分类，允许对事件进行过滤和分组，然后进行分析。

每个instrument 调用规范(见下)都会使用一组标签作为一个参数，这组标签定义了一个唯一的key-value映射。通常传递给API的标签格式为key:value，在这种情况下，规范规定通过获取列表中出现的最后一个value来解析key的重复项。

synchronous instrument的度量通常与具有完全相同标签集的其他度量进行组合，从而优化度量结果。更多参见下文的[通过聚合合并度量](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/metrics/api.md#aggregations)。

#### Meter接口

API定义了一个`Meter`接口，该接口包含一组instrument构造函数，以及一种以语义上的原子方式捕获批量度量数据的工具。

第三方代码可以使用全局的`Meter`实例来嵌入第三方代码。使用该实例允许代码静态地初始化metric instruments，而无需显示注入依赖。在应用安装SDK并通过服务的provider接口或其他特定语言支持的方式初始化全局Meter实例前，该实例仅作为一个无操作(no-op)的实现。注意，可以不使用全局实例：支持同时运行Opentelemetry SDK的多个实例。

作为一个强制性步骤，API要求调用者提供获得一个`Meter`实现时提供 instrumenting library 的名称(可选版本)。库名用于标识由该库产生的工具，可以用于禁用工具，配置聚合以及应用采样策略等目的。更多细节可以参见[TracerProvider](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#tracerprovider)。

#### 聚合

*Aggregation* 指将程序运行的一段时间内产生的多个度量合并为确切的或估计的统计信息的过程。

每个instrument 都指定了一个符合该instrument 语义的默认聚合，用于解释其属性并让用户了解如何使用聚合。在没有任何配置覆盖的情况下，默认聚合提供了一种开箱即用的方式。

adding instruments (`Counter`, `UpDownCounter`, `SumObserver`, `UpDownSumObserver`)默认使用Sum聚合。计算Sum聚合的详细信息各不相同，但从用户的角度看，它可以用于监控捕获的数值的总和。同步和异步instruments 的区别在于如何指定exporter的工作方式，更多参见[SDK specification (WIP)](https://github.com/open-telemetry/opentelemetry-specification/pull/347)。

`ValueRecorder`指标默认使用[TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636) 聚合。

`ValueObserver`指标默认使用LastValue 聚合。这种聚合会持续观测最后一个值，及其时间戳。

还有其他标准的聚合方式，特别对于分组instruments，通常会倾向于获取不同的摘要信息，如直方图，分位数总结，基数估计和其他类型的概要数据结构。

默认的Opentelemetry SDK实现了一个[Views API (WIP)](https://github.com/open-telemetry/oteps/pull/89)，支持在单个instrument层面配置非默认的聚合行为。虽然Opentelemetry SDK可以配置为以非标准方式处理instrument，但仍用户仍希望可以根据其语义来选择对应的instrument(使用默认的聚合进行解释)。

#### 时间

时间是metric events的基本属性，但并不需要明确给出。用户不需要给metric events提供时间戳。不建议SDK捕获每个事件的当前时间戳（通过读取时钟），除非明确需要计算每个事件的高精度时间戳。

这种不需要实现的需求源自对metric报告的优化，即配置一个相对短的周期(如1秒)来采集metric数据，并使用一个时间戳来描述一批暴露出来的数据(当跨分钟或小时进行聚合时，时间精度并不是非常重要)。

聚合通常是在一个连续时间内的一系列事件上进行计算的，这段时间被称为采集间隔。由于SDK控制何时进行采集，因此可以采集聚合的数据，但仅需要在每个采集间隔读取一次时钟。默认的SDK采用了这种方式。

使用synchronous instruments产生的metric events是立即发生的，这些落入采集间隔的数据会与其他使用相同instrument和标签集的事件一起聚合。由于各个事件都可能同时发生，因此无法很好地定义*最近的事件*。

Asynchronous instruments允许SDK通过在每次收集间隔执行一次观察的方式来评估metric instrument 。由于与采集耦合(与synchronous instruments不同)，因此这些instruments明确定义了最近的事件。例如将针对某个时刻使用相同的instrument和标签集的*Last Value*定义为最近一次采集间隔内测得的值。

由于metric events的时间戳是隐式的，因此可以将一些列的metric events看作是一个时间序列事件。但在SDK规范中保留了这个术语，它指代数据格式的一部分，这些数据格式以序列的方式显式地表示带有时间戳的值，这些值是一段时间内的原始度量的聚合结果。

#### Metric Event格式

无论哪种类型的instrument，其Metric events具有相同的逻辑表达。通过instrument捕获的metric事件包含：

- 时间戳(隐式)
- instrument定义(名称，类型，描述，度量单位)
- 标签集(key和value)
- 值(有符号整数或浮点数)
- 启动时与SDK相关的[资源](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/resource/sdk.md)

同步事件还有一个额外的属性，即处于活动状态的分布式上下文（包含Span，Bagage等）。

### Meter provider

通过初始化和配置OpenTelemetry Metrics SDK，可以获得具体的`MeterProvider`实现。本文不会指定如何构造一个SDK，仅说明这些SDK必须实现`MeterProvider`。一旦配置完成，应用或库会选择是否使用`MeterProvider`接口的全局实例，或使用依赖注入来更好地配置provider。

#### 获取一个Meter

可以通过`MeterProvider`及其`GetMeter(name, version)`方法来创建一个新的`Meter`实例。通常会将`MeterProvider`作为单实例使用。实现时应该提供一个全局默认的`MeterProvider`。`GetMeter`方法包含两个字符串参数：

- name(必要)：name用于标识 instrumentation library(如`io.opentelemetry.contrib.mongodb`)。当指定了无效的名称(null或空字符串)时，将返回一个默认的`Meter`实现，不应该返回null或抛出异常。如果应用所有者将SDK配置为禁用该库生成的遥测，则此时`MeterProvider`可以返回一个无操作(no-op)的`Meter`。
- version(可选)：指定instrumentation library的版本(如`1.0.0`)。

每个不同名称的`Meter`都会为它的metric instrument创建一个独立的命名空间，这样instrumentation library就可以使用相同的instrument名称来报告metric。不建议将`Meter`的名称作为instrument名称的一部分，因为这样会导致instrumentation library无法使用相同的名称来捕获metrics。

#### 全局Meter provider

很多场景下，使用全局实例看起来像是一个相反的模式(*单例模式*)。但在大多数场景下，为了在不使用依赖注入的情况下合并来自相互依赖库的遥测数据，这反而是一种正确的模式，出于这种原因，Opentelemetry 语言对应的API应该提供一个全局实例。各个语言提供的全局实例必须保证通过全局`MeterProvider`分配的`Meter`以及通过这些Meter实例分配的instrument的初始化推迟到全局SDK首次初始化时。

##### 获取全局的MeterProvider

由于全局`MeterProvider`是单例的，且仅支持一个单独的方法，调用者可以使用全局`GetMeter`的来获取全局的`Meter`。例如，global.GetMeter(name, version)会在全局`MeterProvider`上调用`GetMeter`，并返回一个命名的`Meter`实例。

##### 设置全局的MeterProvider

全局的函数会安装一个MeterProvider作为全局SDK，例如在初始化之后使用`global.SetMeterProvider(MeterProvider)`安装SDK。

### instrument属性

由于API与SDK是划分开的，具体的实现将决定metric event是如何被处理的。因此，应该按照语义和预期的解释来选择instrument。通过多种属性来定义单个instrument的语义和细节，可以帮助选择合适的instrument。

#### instrument命名要求

metric instrument主要通过名称进行定义，这也是外部系统引用的方式。metric名称遵循如下语法：

1. 非空字符串
2. 大小写不敏感
3. 第一个字符必须是非数字，非空格，非标点
4. 后续的字符必须使用字母数字，'_', '.', 和'-'。

metric instrument属于一个命名空间，通过相关的`Meter`实例进行创建。当多个instrument使用相同的名称进行注册时，`Meter`必须返回错误。

Metric instrument名称应该具有语义意义，它独立于原始的Meter名称。例如，当监测一个http server时，"latency"并不是一个合适的instrument名称，因为它代表的意义太宽泛。相反，应该采用如"http_request_latency"这样的instrument，通过名称可以告知观察者延迟测量的语义。多个 instrumentation libraries可能生成该instrument。

#### 同步和异步instruments比较

Synchronous instruments是在请求中调用的，这意味着它们有一个关联的分布式上下文(包括Span、Baggage等)。在一个给定的采集间隔内，可能有多个metric event对应一个Synchronous instruments。

Asynchronous instruments 通过回调函数(在每次采集间隔时)进行报告，缺少上下文。每个周期每个标签组只能报告一个值。如果针对相同的标签集，应用观察到了多个值，则会仅会保留最后一个值。

#### Adding和分组instruments比较

Adding instruments 用于捕获有关总和的信息，根据定义，只有总和才有意义。对于这些instruments 来说，单独的事件是没有意义的。例如两个`Counter`事件`Add(N)`和`Add(M)`等价于一个`Counter` 事件`Add(N + M)`。这是因为`Counter`是同步的，且synchronous adding instruments会将捕获到的度量进行加法运算。

Asynchronous adding instruments(如`SumObserver`)用于直接捕获总和。如，对于给定的instrument 和标签集，在`SumObserver`观察序列中，Last Value定义了instruments的和。

同步和异步场景下，每个采集间隔内都会将adding instruments以低成本的方式聚合成单个数字，而不会丢失信息。这种属性使得adding instruments相比分组instruments具有更高的性能。

与记录完整的数据相比，默认情况下分组instruments会使用一种相对低廉的聚合方式。但仍然比默认的adding instruments(Sum)开销大。与只关心sum的adding instruments不同，分组instruments可以配置开销更大的聚合器。

#### 单调和非单调instruments比较

单调性仅适用于adding instruments。`Counter` 和`SumObserver`被定义为是单调的，因为它们的和是非减的。 以`UpDown-`开头的instrument是非单调的，意味着总和可以增加，减少，或保持不变。

单调instruments通常用于捕获关于和的信息，这些信息用于监控速率变化。本API定义的单调属性指非递减的和。Metric API不考虑非递增和。

#### 函数名称

每个instrument都支持一个函数，对函数的命名应该传达instrument的语义。

Synchronous adding instruments支持`Add()`函数，意味着这些instruments 会计算和，不会直接捕获和(sum)。

Synchronous grouping instruments支持`Record()`，意味着这些instruments 捕获单个事件，不仅仅是和(sum)。

所有的asynchronous instruments都支持`Observe()`函数，意味着，每个度量间隔仅会捕获一个值。

### instrument

#### Counter

`Counter`是最常用的synchronous instrument，该instrument支持通过`Add(increment)`函数来报告和，仅限非负增量。与所有adding instrument相同，默认的聚合为`Sum`。

`Counter`的例子如：

- 计算接收的字节数
- 计算完成的请求数
- 计算创建的账户数
- 计算检查点数
- 计算5xx错误数

上述示例instruments对于监控这些这些场景下的的速率变化很有用。在这些场景中，通常会报告一个sum是如何改变的，而非根据每次度量进行计算和报告sum。

#### UpDownCounter

`UpDownCounter`类似`Counter`，不同之处在于`Add(increment)`支持负增长。这使得`UpDownCounter`不适用于计算速率聚合。该instrument可以通过`Sum`进行聚合，但结果是非单调的。通常用于计算使用资源的总数，或请求的上升和下降。

`UpDownCounter`的例子为：

- 计算活动的请求数
- 计算使用new和delete的内存
- 计算`enqueue` 和`dequeue`的队列长度
- 计算信号量的`up` 和`down`操作。

这些例子监控一组进程的资源使用情况。

#### ValueRecorder

`ValueRecorder`是一个grouping synchronous instrument ，用于记录所有分组数字(正数或负数)。`Record(value)`捕获的值被认为属于一个正在汇总的分布中的单个事件。当捕获总和没有意义的度量值时，或虽然捕获的数字本身就是adding的，但认为单独的数值增长更具有现实意义时，应该选择`ValueRecorder`。

最常使用`ValueRecorder`的场景是捕获延时度量。延时度量并没有adding的意义，因此没有必要知道所有处理的请求的延迟总和。使用`ValueRecorder instrument 捕获延时度量通常是因为对平均值、中位数和其他个别事件的汇总统计数据感兴趣。

`ValueRecorder`的默认聚合会计算最小和最大值，事件值的总和以及事件的总数，允许监控输入值的速率、平均值和范围。

使用`ValueRecorder`的例子为：

- 捕获各种类型的时间信息
- 捕获飞行员所经历的加速
- 捕获喷油器的喷嘴压力
- 捕捉MIDI按键的速度

`ValueRecorder`捕获度量中用到的adding仅仅是累加，但最终关注的是数值的分布，而不仅仅是总和：

- 捕获请求大小
- 捕获账户负载
- 捕获队列大小
- 捕获一些板英尺的木材。

这些例子展示了，虽然这些度量是可以adding的，但选择`ValueRecorder`而不是`Counter`或`UpDownCounter`意味着关心的维度不仅限于总和。如果不关心采集到的分布情况，那么可以选择后者的某一个。使用`ValueRecorder`捕获分布情况对于观察性设置非常重要。

使用这类instrument需要注意，它们比adding度量开销大。

#### SumObserver

`SumObserver`是Counter对应的asynchronous instrument，用于通过 `Observe(sum)`捕获单调和。Sum的名字可以提醒用户，该instrument用于直接捕获和。使用`SumObserver`来捕获从0开始并在整个处理生命周期中上升且不会下降的所有值。

`SumObserver`的例子为：

- 捕获处理用户/系统的CPU秒数
- 捕获缓存miss数

当计算一个度量的开销比较大时，可以选择使用`SumObserver`(这样就不会因为每个请求而浪费计算资源)。例如，一个系统调用需要捕获处理使用的CPU，因此需要周期性地进行采集，而不是针对每个请求都进行采集。当通过测量单个变更来计算总和是不切实际或比较浪费资源的情况下，也可以使用`SumObserver`。例如，即使缓存miss的和是单个缓存miss事件的总和，但使用`Counter`同步捕获每个事件会浪费大量资源。

#### UpDownSumObserver

`UpDownSumObserver`是对应`UpDownCounter`的asynchronous instrument。可以通过`Observe(sum)`捕获非单调总数。使用`UpDownSumObserver`来捕获从0开始并在整个处理生命周期中上升和下降的任何值。

`UpDownSumObserver`的例子为：

- 捕获进程堆大小
- 捕获活动的分片数
- 捕获开始的/完成的请求数
- 捕获当前队列大小

选择`UpDownSumObserver`而不选择同步`UpDownCounter`的原因与选择`SumObserver`而不选择同步Counter的原因相同。如果计算一个度量的开销比较大，或对应的变更非常频繁，那么使用`UpDownSumObserver`来进行度量比较现实。

#### ValueObserver

`ValueObserver`是对应`ValueRecorder`的asynchronous instrument，可以通过`Observe(value)`捕获一部分组度量。这些instrument通常用于捕获比较难以计算的度量，因为它可以让SDK控制评估的频率。

`ValueObserver`的例子为：

- 捕获CPU风扇速度
- 捕获CPU温度

注意这些例子都使用分组度量。在上面的`ValueRecorder`场景中，给出了在请求期间捕获同步adding度量的示例(例如，请求看到的当前队列大小)。那么在异步场景中，用户如何决定选择`ValueObserver`还是`UpDownSumObserver`。

考虑一个场景，如何异步地报告一个队列的大小。逻辑上，`ValueObserver` 和`UpDownSumObserver`都可以应用于这种场景。asynchronous instrument只会在每个采集间隔采集一个度量。这种情况下，`UpDownSumObserver`会报告一个当前的总和，而`ValueObserver`会报告一个当前的总和(等于最大和最小)，且总数为1。如果没有聚合，此时的结果是相等的。

当只有一个数据点时，定义默认聚合并没有什么意义。默认的聚合在执行空间聚合时才会有用，意思是跨标签集或在分布式设置中合并测量。虽然一个`ValueObserver`在每个采集间隔仅观测一个值，但默认的聚合将指定如何将它与其它值进行聚合，而无需其他配置。

因此，当考虑应该选择`ValueObserver`还是`UpDownSumObserver`时，建议选择拥有更合适的默认聚合的instrument。如果想要观测一组机器上的队列大小，且仅关心聚合的队列大小，那么可以选择`SumObserver`，该asynchronous 会生成一个总和，而不是分布。如果想要观测一组机器上的队列大小，且对这些机器上的队列大小分布比较感兴趣，那么可以选择`ValueObserver`。

#### Interpretation

那么这些instrument 最根本的不同是什么，为什么只有三种，为什么不是一种或十种？

从前面可以看到，这些instrument按照同步，adding，和/或单调进行了分类。这种方式给予每个instrument特殊的语义，通过这种方式提升了metric event的性能和解释性。

建立不同类型的instrument 是非常重要的，因为大部分场景下，SDK会提供默认的开箱即用的功能，而无需配置其他行为。对instrument的选择不仅仅取决于事件的含义，也取决于用户调用的函数的名称。函数名`Add()`，用于adding instrument，`Record()`用于grouping instrument，`Observe()`用于asynchronous instrument，各自传达了这些动作的意义。

单个instrument的属性和标准实现描述如下：

| **Name**              | Instrument kind               | Function(argument) | Default aggregation                                          | Notes                                       |
| --------------------- | ----------------------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------- |
| **Counter**           | Synchronous adding monotonic  | Add(increment)     | Sum                                                          | Per-request, part of a monotonic sum        |
| **UpDownCounter**     | Synchronous adding            | Add(increment)     | Sum                                                          | Per-request, part of a non-monotonic sum    |
| **ValueRecorder**     | Synchronous                   | Record(value)      | [TBD issue 636](https://github.com/open-telemetry/opentelemetry-specification/issues/636) | Per-request, any grouping measurement       |
| **SumObserver**       | Asynchronous adding monotonic | Observe(sum)       | Sum                                                          | Per-interval, reporting a monotonic sum     |
| **UpDownSumObserver** | Asynchronous adding           | Observe(sum)       | Sum                                                          | Per-interval, reporting a non-monotonic sum |
| **ValueObserver**     | Asynchronous                  | Observe(value)     | LastValue                                                    | Per-interval, any grouping measurement      |

> TIPs:
>
> 注意其默认的聚合模式，sum类型的聚合模式会对给出的数值进行累加，适用于通过exporter进行数值的累加计算。如果用户已经给出了结果，则可以使用ValueObserver直接获取该值，不进行聚合计算。

#### 构造器

`Meter`接口支持创建并注册新的metric instrument。instrument构造器是通过在它构造的instrument类型上添加一个`New-`前缀来命名的，使用构造器模式或该语言中的其他惯用方法。

在本规范中，每种instrument至少有一个构造函数。例如，如果需要专门处理整数和浮点指针数，则OpenTelemetry API将为每种instrument类型提供2个构造函数。

将instruments绑定到单个`Meter`实例有两个好处：

- 可以在首次使用之前就从零状态导出instruments，而无需显式的注册调用
- 二进制名称和版本与metric event隐式地关联起来

一些现有的metric系统支持静态地分配metric instruments，并在使用时提供等价的`Meter`接口。在一个典型的statsd客户端示例中，现有的代码可能无法在一个方便的地方存储新的metric instruments。如果这种方式是一种负担，则建议使用全局`MeterProvider`构造一个静态`Meter`，并构造和使用全局范围的metric instruments。

上述场景类似使用现有Prometheus客户端的用户，此时可以使用全局`Registerer`分配instruments。这类代码可能无法在定义instruments的地方访问一个合适的`MeterProvider` 或`Meter`实例。如果成为一种负担，建议通过全局Meter provider构建出的静态命名的`Meter`来构建metric instruments。

应用希望构建长生命的instruments。SDK中的instruments是永久的，没有方法删除。

### 标签集

语义上，一组标签为一个唯一的字符串key到value的映射。在整个API中，必须以相同的惯用格式来传递一组标签。通常的表达方式包括一个顺序的key:value列表，或一个key:value的映射。

当标签一一个顺序的key:value列表进行传递时，如果发现重复的key，则对任何给定的key，将使用列表中的最后一个value来构造唯一的映射。

标签值的类型通常被exporters假定为字符串，尽管作为语言级别的决策，但标签值类型可以是该语言中具有字符串表示形式的任何惯用类型。

用户不需要提前声明将会被API的metric instruments使用的标签key集。用户在调用API时可以为任何metric event设置任何标签。

#### 标签性能

在整个metric数据的生成中，对标签的处理是一个很大的成本。

SDK对处理聚合的支持取决于查找*instrument-标签集组合对*的活动记录的能力。这种方式允许对度量进行组合。可以通过使用绑定synchronous instruments和批量报告函数 (`RecordBatch`, `BatchObserver`)来降低标签的开销。

#### 可选：有序列表

作为语言级别的决策，API可能会支持标签key排序。这种情况下，用户可能会指定有序的标签keys序列，用于从一个有序的标签values序列中创建一个无序的标签集，例如：

```go
var rpcLabelKeys = OrderedLabelKeys("a", "b", "c")

for _, input := range stream {
    labels := rpcLabelKeys.Values(1, 2, 3)  // a=1, b=2, c=3

    // ...
}
```

这种方式被指定为一种语言可选的特性，因为它的安全性以及作为监控输入的值取决于源语言中的类型检查。传递无序的标签(即从key到value的映射)被认为是更安全的选择。

## Synchronous instrument细节

下面介绍Synchronous instrument的细节。

#### 同步调用规范

metric API提供了三种语义上等价的方法来使用Synchronous instruments捕获度量：

- 调用绑定的instruments，它们具有一组预先关联的标签
- 直接调用instruments，传入相关的一组标签
- 使用一组标签来批量记录多个instruments的度量

三种方式都会生成相同的metric events，但在性能和便捷度上不尽相同。

metric API的性能取决于输入新度量所做的工作，通常主要是处理标签产生的开销。绑定instruments是性能最高的调用规范，因为它们可以将处理标签的成本分摊到许多用途上(*首先固定一组标签，然后应用到多个度量上*)。通过`RecordBatch()`记录多个度量值是提高性能的一个不错的选择，因为处理标签的成本分布到了多个度量值上(*其实就是多个度量使用了一组标签，这样处理一组标签就可以满足多个度量的要求，通过这种方式降低了开销*)。直接调用规范是最方便的，但也是通过API输入度量使用的性能最低的调用规范。

##### 绑定instrument的调用规范

当兼顾到性能要求，且一个metric instrument重复使用了相同的标签集时，开发人员可能会选择使用*绑定instrument*调用规范来进行优化。对于绑定instrument，它要求重复使用特定的instrument和标签。如果带有相同标签的instrument多次被使用，通过获取与标签相对应的绑定instrument，可以达到最高的性能。

为了绑定一个instrument，可以使用`Bind(labels...)`方法来返回支持同步API的接口(即`Add()` 或`Record()`)。绑定instrument不需要通过标签进行调用，因为与metric event对于的标签已经绑定到了instrument上。

绑定instrument会提升性能，但也会消耗SDK中的资源。绑定instrument必须提供一个`Unbind()`方法来让用户取消绑定，并释放相关的资源。注意`Unbind()`不会暗示删除时间戳，仅保证SDK在没有等待处理更新后忘记timeseries的存在。

例如，使用相同的标签重复更新一个counter：

> 通过给instrument绑定预先定义的标签，后续使用instrument时将不会使用标签

```go
func (s *server) processStream(ctx context.Context) {

  // The result of Bind() is a bound instrument
  // (e.g., a BoundInt64Counter).
  counter2 := s.instruments.counter2.Bind(
      kv.String("labelA", "..."),
      kv.String("labelB", "..."),
  )
  defer counter2.Unbind()

  for _, item := <-s.channel {
     // ... other work

     // High-performance metric calling convention: use of bound
     // instruments.
     counter2.Add(ctx, item.size())
  }
}
```

##### 直接instrument调用规范

当使用方便比性能更重要时，或事先不知道值时，用户可能会直接操作metric instruments，意味着需要在调用时提供标签。这种方式提供了最好的便利性。

例如，更新一个counter：

```go
func (s *server) method(ctx context.Context) {
    // ... other work

    s.instruments.counter1.Add(ctx, 1,
        kv.String("labelA", "..."),
        kv.String("labelB", "..."),
        )
}
```

直接调用非常方便，因为无需申请和保存绑定的instrument。当一个instrument很少被使用，或很少使用相同的标签时就可以使用这种方式。与绑定instrument不同，使用直接调用规范不会长期消耗SDK资源。

##### RecordBatch 调用规范

这是最后一个输入度量的调用规范，与直接调用规范类似，但同时支持多个度量。`RecordBatch` API支持输入多个度量，意味着对多个instruments进行语义上的原子更新。调用`RecordBatch`可以将标签的处理成本分摊到多个度量中。

```go
func (s *server) method(ctx context.Context) {
    // ... other work

    s.meter.RecordBatch(ctx, labels,
        s.instruments.counter.Measurement(1),
        s.instruments.updowncounter.Measurement(10),
        s.instruments.valuerecorder.Measurement(123.45),
    )
}
```

另外一个用于*record batch*的有效接口使用了构造器模式：

```go
    meter.RecordBatch(labels).
        put(s.instruments.counter, 1).
        put(s.instruments.updowncounter, 10).
        put(s.instruments.valuerecorder, 123.45).
        record();
```

使用*record batch*调用规范在语义上与直接调用相同，只是增加了原子性。由于通过单个调用来输入值，从exporter的角度来看，SDK能够实现原子更新，因为SDK可以以队列的方式实现单次批量更新，或者仅使用一次锁。与直接调用相同，使用批量调用规范不会长期消耗SDK资源。

#### 与分布式上下文进行关联

Synchronous measurements在运行时隐式地与分布式上下文关联，其中可能包括Span和Baggage项。Metric SDK可以以多种方式使用此信息，这是OpenTelemetry中特别关注的一个功能。

##### Baggage into metric labels

OpenTelemetry 支持的Baggage 用于在分布式计算中将标签从一个进程传递到下一个进程。有时会使用分布式baggage项作为metric标签来聚合度量数据。

必须通过显式的配置才能使用Baggage，使用Views API (WIP)选择特定的baggage项作为标签。由于使用Baggage标签的开销非常大，默认的SDK不会在export pipeline中自动使用Baggage标签。

配置应用于Baggage标签的视图的[相关工作正在进行中](https://github.com/open-telemetry/oteps/pull/89)

### Asynchronous instrument

下面介绍Asynchronous instrument的细节。

#### 异步调用规范

metrics API提供了两种语义上等价的方法来使用asynchronous instrument捕获度量：要么通过单instrument回调，要么通过多instrument批量回调。

无论是单还是批量，asynchronous instruments只能通过一个回调进行观测。对于null的observer回调，构造器会返回无操作的instrument。如果为asynchronous instruments指定了多个回调，则会将其视为错误。

每个instrument的不同标签集不能观察到一个以上的值。当一个instruments和标签集观测到多个值时，会采用最后一个观测到的值，并丢弃之前的值，不会返回错误。

##### 单instrument observer

单instrument回调会绑定到一个instrument。回调会接收一个带有`Observe(value, labels…)`函数的`ObserverResult`。

```go
func (s *server) registerObservers(.Context) {
     s.observer1 = s.meter.NewInt64SumObserver(
         "service_load_factor",
          metric.WithCallback(func(result metric.Float64ObserverResult) {
             for _, listener := range s.listeners {
                 result.Observe(
                     s.loadFactor(),
                     kv.String("name", server.name),
                     kv.String("port", listener.port),
                 )
             }
          }),
          metric.WithDescription("The load factor use for load balancing purposes"),
    )
}
```

##### Batch observer

`BatchObserver`回调支持在一个回调中观测多个instruments。它会接收一个带有`Observe(labels, observations...)`函数的`BatchObserverResult `。

通过在asynchronous instrument上调用`Observation(value)`来返回观测值。

```go
func (s *server) registerObservers(.Context) {
     batch := s.meter.NewBatchObserver(func (result BatchObserverResult) {
          result.Observe(
             []kv.KeyValue{
                 kv.String("name", server.name),
                 kv.String("port", listener.port),
             },
             s.observer1.Observation(value1),
             s.observer2.Observation(value2),
             s.observer3.Observation(value3),
          },
    )

     s.observer1 = batch.NewSumObserver(...)
     s.observer2 = batch.NewUpDownSumObserver(...)
     s.observer3 = batch.NewValueObserver(...)
}
```

##### 异步观测结果构造当前值集合

允许asynchronous instrument回调对针对每个instrument、每个不同的标签集、每个回调调用来观测一个值。一个回调调用记录的一组值代表该instrument的当前快照。 这组值定义了直到下一个采集间隔前，该instrument的Last Value。

asynchronous instrument需要对它认为是“当前”的每个标签集记录一个观察结果，意味着即使在上次回调调用结束后，某个值没有任何变动，异步调用也能够观测到它。不观测某个标签集意味着其对应的值不再是当前值。如果在采集间隔中未观察到Last Value，则该值将不再是当前的值，因此该值将变得不确定。

可以在asynchronous instrument中定义Last Value，因为SDK会协调采集，并报告所有的当前值。另外一个对该属性的解释为，SDK可以在内存中保留一个观察值的采集间隔值，用于查找任何instrument和标签集的当前Last Value。通过这种方式，asynchronous instrument支持查询当前值，不依赖于采集间隔的持续时间，而使用单点时间采集的数据。

回顾一下，Last Value并不是为asynchronous instrument定义的，它是一个精确的数值，因为并没有很好地定义"当前"的概念。为了确定asynchronous instrument记录的最后一个值，需要检查采集数据的窗口，因为并没有机制来保证每个间隔都会记录当前值。

##### asynchronous instrument定义时间比例

上述asynchronous instrument的当前集合的概念同样使用于监控比率。当一种instrument的一组观测值加起来是一个整体时，那么可以使用观测值除以相同间隔内采集的的观测值之和来计算其相对贡献。通过这种方式定义了当前相对贡献，计算方式与收集间隔时间无关，得益于asynchronous instrument的属性。

### 并行

对于支持并行执行的语言，Metrics API提供了特定的保证和安全性。但并不是所有的API函数都可以被安全地并发执行：

**MeterProvider** - 所有函数都可以并发执行

**Meter** - 所有函数都可以并发执行

**Instrument** - 所有指标的所有方法都可以并发执行

**Bound Instrument** - 所有绑定指标的所有方法都可以并发执行

### 相关的工作

如下工作正在进行中

#### Metric Views

API不支持为metric Instrument配置聚合。

*View API*定义为一个SDK机制的接口，该机制支持配置聚合，包括应用了哪些操作符(sum、p99、last-value等)以及使用了哪些维度。

参见 [current issue discussion on this topic](https://github.com/open-telemetry/opentelemetry-specification/issues/466) 和 [current OTEP draft](https://github.com/open-telemetry/oteps/pull/89).

#### OTLP Metric 协议

如上所述，OTLP协议旨在以无内存的方式导出metric标准数据。 该协议的几个细节正在制定中。 参见[current protocol](https://github.com/open-telemetry/opentelemetry-proto/blob/master/opentelemetry/proto/metrics/v1/metrics.proto)。

#### Metric SDK默认实现

OpenTelemetry SDK 默认支持metric API，默认SDK的规范正在进行中，参见 [current draft](https://github.com/open-telemetry/opentelemetry-specification/pull/347)。
