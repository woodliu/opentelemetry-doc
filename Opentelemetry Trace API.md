## Opentelemetry Trace API

[TOC]

Tracing API包含三个主要类：

- [`TracerProvider`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#tracerprovider)：API的入口，提供访问 `Tracer`s。
- [`Tracer`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#tracer)：用于创建 `Span`s。
- [`Span`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span)：用于跟踪操作。

### 数据类型

鉴于不同的语言和平台使用不同的方式来表达数据，因此本章节仅定义了通用的API需求。

#### Time

OpenTelemetry 可以操作纳秒级别的时间数值，表达方式取决于具体的实现语言。

##### Timestamp

时间戳是自Unix出现以来经过的时间。

- 最小精度是毫秒
- 最大精度是纳秒

##### Duration

持续时间是两个事件之间经过的时间。

- 最小精度是毫秒
- 最大精度是纳秒

### TracerProvider

可以通过`TracerProvider`访问`Tracer`s。

在实现API时，`TracerProvider`应该是一个保存了所有配置的有状态对象。

通常`TracerProvider`作为一个主导位置来访问， 因此，API应该提供一种设置/注册和访问全局默认`TracerProvider`的方法。

尽管有了全局的`TracerProvider`，但一些应用仍然希望使用多个`TracerProvider`实例，例如，为了给每个处理器(如`SpanProcessor`s)提供不同的配置(进而提供给`Tracer`s )，或者为了更方便地使用依赖注入框架。因此，实现`TracerProvider`时应该允许创建任意多的`TracerProvider`实例。

#### TracerProvider 

`TracerProvider` 必须实现如下功能：

- 获取一个Tracer

##### 获取Tracer

API 必须接受如下参数：

- name(必须)：该名称必须标识[instrumentation library](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/overview.md#instrumentation-libraries)(如`io.opentelemetry.contrib.mongodb`)而不是[instrumented library](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/glossary.md#instrumented-library)。如果指定了无效的名称(null或空字符串)，则会返回一个默认的Tracer实现，而不是null或抛出异常。如果不支持命名功能(即一个与观察性无关的实现)，则一个实现了OpenTelemetry API的库可能会忽略该名称，并为所有的调用返回一个默认的实例。如果应用所有者将SDK配置为禁用由此库生成的遥测，则一个TracerProvider可以返回一个无操作的Tracer。
- version(可选)：指定instrumentation library的版本(如`1.0.0`).

API没有指定是否或在哪种情况下会返回相同或不同的`Tracer`实例。

实现中不能要求用户为了获取配置变更而重复使用相同的name+version来获取一个`Tracer`。可以通过允许使用过期的配置或保证新的配置也会应用到前面返回的`Tracer`s中来实现。

注意：可以在`TracerProvider`中存储任何可变配置，并让`Tracer`对象包含一个到`TracerProvider`的引用，由此获得`TracerProvider`保存的配置。如果每个Tracer必须保存一份配置(如为了禁用特定的Tracer)，则Tracer可以在TracerProvider的map中使用name+version进行查找，或可以通过TracerProvider的注册中心来返回所有的`Tracer`s，如果发生了变更则更新配置。

### Context 交互

本节定义了所有与[Context](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/context.md)交互的Tracing API操作。

API必须提供如下一个与`Context`实例交互的功能：

- 从一个`Context`接口中抽取`Span`
- 将`Span`注入一个`Context`接口

上述功能是必须的，因为API用户不应该使用Tracing API实现使用的[Context Key](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/context.md#create-a-key)

如果语言支持隐式地传播`Context`([参见](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/context.md#optional-global-operations))，则API应该提供如下功能：

- 从隐式的上下文中获取当前活动的span。等价于获取隐式的上下文，然后从中抽取`Span`
- 将当前活动的span设置到隐式的上下文中，等价于获取隐式的上下文，然后将`Span`注入其中。

上述所有功能都只在上下文API上进行操作，并且可能作为trace模块的静态方法公开，作为trace模块内部的类方法(它可能被命名为`TracingContextUtilities`)，或作为[`Tracer`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#tracer) 的类方法。应该尽量实现API中的这种功能。

### Tracer

tracer用于创建`Span`s。

注意，`Tracers`通常不负责配置，具体的配置由`TracerProvider`负责。

#### Tracer 操作

一个Tracer必须提供如下功能：

- [创建一个新的Span](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span-creation)

### SpanContext

一个`SpanContext`表示一个Span的一部分，必须序列化并沿着分布式上下文传播。`SpanContext`s是不可变的。

Opentelemetry `SpanContext`的表达方式遵循[W3C TraceContext 规范](https://www.w3.org/TR/trace-context/)，它包含两个标识符，一个`TraceId`和一个`SpanId`，以及一系列的公共`TraceFlags`和系统指定的`TraceState`值。

`TraceId`：一个有效的trace标识符，是一个16字节数组，其中至少有一个非零字节。

`SpanId`：一个有效的span标识符，是一个8字节数组，其中至少有一个非零字节。

`TraceFlags`：包含trace的详细信息，与TraceState值不同，TraceFlags会出现在所有的trace中。当前版本的规范中仅支持一个flag，称为[sampled](https://www.w3.org/TR/trace-context/#sampled-flag)。

`TraceState`：包含供应商指定的trace标识数据，为一系列key-value对。TraceState允许在系统的trace中引入多个trace系统。详情参见[W3C Trace Context规范](https://www.w3.org/TR/trace-context/#tracestate-header)。

#### 检索TraceId和SpanId

API必须允许使用如下格式检索`TraceId`和`SpanId`：

- 十六进制：返回小写的十六进制编码的`TraceId`(结果必须是一个32个十六进制的小写字符串)和`SpanId`(结果必须是一个16个十六进制的小写字符串)
- 二进制：返回`TraceId`的二进制表示形式(必须是16字节数组)和`SpanId`(必须是8字节数组)

API不应该保留内部保存的细节。

#### IsValid

调用`IsValid`后会返回一个布尔值，当SpanContext 具有非0的`TraceID` 和非0的`SpanId`时返回`true`。必须提供该功能。

#### IsRemote

调用`IsRemote`后会返回一个布尔值，当SpanContext 从一个远端父span传播过来时返回`true`。必须提供该功能。当通过[Propagators API](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#propagators-api)抽取一个`SpanContext`时，`IsRemote`必须返回true，而对于任何子span的SpanContext，则必须返回false。

#### TraceState

`TraceState`是[`SpanContext`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#spancontext)的一部分，使用一组不可变的key/value对(由[W3C Trace Context 规范](https://www.w3.org/TR/trace-context/#tracestate-header)正式定义)表示。Tracing API必须至少提供如下一种方式来操作`TraceState`：

- 通过给定的key获取value
- 添加一个新的key/value对
- 通过给定的key更新现有的value
- 删除一个key/value对

这些操作必须遵循[W3C Trace Context 规范](https://www.w3.org/TR/trace-context/#mutating-the-tracestate-field)中的规则。所有可变操作都必须返回一个已应用了修改的新`TraceState`。根据[W3C Trace Context 规范](https://www.w3.org/TR/trace-context/#mutating-the-tracestate-field)中指定的规则，`TraceState`必须始终有效。每个可变的操作必须校验输入参数。如果传入了无效的参数，则不能返回包含无效数据的`TraceState`，且必须遵循[通用错误处理指南](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/error-handling.md)(即通常不能返回null或抛出异常)。

注意，由于`SpanContext`是不可变的，因此不能使用新的`TraceState`来对`SpanContext`进行更新，对`SpanContext`的更新操作只有在执行[`SpanContext` 传播](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md) 或[遥测数据导出](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#span-exporter)前才有意义，这两种情况下，`Propagators` 和`SpanExporters`可能会在序列化到网络上前创建一个修改过的`TraceState`。

### Span

一个`Span`表示一个trace中的一个单独的操作。Span可以嵌套成trace树。每个trace都包含一个根span，通常会对整个操作进行描述，还可以选择性地描述其子操作的一个或多个子span。

`Span`封装了如下内容：

- span名称
- 唯一标识`Span`的不可变[`SpanContext`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#spancontext) 
- 父span，格式为 [`Span`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#span), [`SpanContext`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#spancontext), or null
- 一个[`SpanKind`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#spankind)
- 一个开始时间戳
- 一个结束时间戳
- [`Attributes`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)
- 一系列连接到其他`Span`s的[`Link`s](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#specifying-links) 
- 一系列带时间戳的[`Event`s](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#add-events)
- 一个[`Status`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#set-status)

span名称简明传达了Span的工作，例如，一个RPC方法名称，一个函数名称，或子任务的名称或大型计算的一个阶段。span名称应该是最通用的字符串，它应该标识一个(统计上)感兴趣的span类，而不应该既是一个独立的Span实例，同时又是可读的。  即"get_user" 是一个合理的名称，而"get_user/314159"("314159"是用户ID)不是一个好的名称，因为它的基数比较高。通用性应该优先于人的可读性。

例如，下面是一个用于获取(假设的)帐户信息的终端上的潜在span名称:

| Span Name                 | Guidance                                               |
| ------------------------- | ------------------------------------------------------ |
| `get`                     | 太泛了                                                 |
| `get_account/42`          | 太详细                                                 |
| `get_account`             | 好, and account_id=42 would make a nice Span attribute |
| `get_account/{accountId}` | 好 (using the "HTTP route")                            |

`Span`的开始和结束时间戳反应了操作经过的真实时间。

例如，如果一个span表示一个请求响应的周期(即HTTP或RPC)，则span应该包含一个对应第一个子操作开始的时间，以及对应最后一个子操作结束的时间。这些操作包含：

- 从请求接收数据
- 解析数据
- 中间件和额外的处理逻辑
- 业务逻辑
- 构建响应
- 发送响应

可能会创建子span(或某些事件)可来表示子操作(此时需要提供更详细的可观测性)。子span应该分别计算子操作的时间，并添加额外的属性。

`Span`的开始时间应该设置为span创建时的时间。在Span创建之后，应该修改名称，设置`Attribute`s，添加`Event`s，设置`Status`。在Span的结束时间设置之后就不能修改这些信息。

`Span`并非用于在流程内传播信息。为了防止误用，实现时不应该提供对`Span`属性(除它的`SpanContext`之外)的访问。

供应商可能会实现Span接口来影响特定于供应商的逻辑。然而，在可供替代的实现中不能允许调用者直接创建`Span`s。所有的`Span`s必须通过`Tracer`创建。

#### Span创建

不能够提供除`Tracer`以外的创建`Span`的API。

默认情况下，不能将新创建的`Span`作为当前活动的`Span`，但该功能可能作为一个独立的操作。

该API必须接收如下参数：

- span名称，必需参数

- 父`Context`或表明新的`Span`应该是根`Span`。API可能包含一个可选项，用于提供一个默认行为：隐式地将当前Context作为父Context。该API不能接受 `Span` 或`SpanContext`作为父Context，只接受完整的`Context`。

  Span的语义上的父Span必须遵循中[Determining the Parent Span from a Context](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#determining-the-parent-span-from-a-context)描述的规则。

- [`SpanKind`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#spankind)，如果没有指定，默认为Span`Kind.Internal`。

- [`Attributes`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)：该属性可能用于做出采用决定([sampling description](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampling))。如果没有指定，则假设采集的内容为空。

  只要可能，用户应该在span创建时设置已知的属性，而不应该后续通过调用`SetAttribute`进行设置。

- `Link`s：有序的Links序列，参见API[定义](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#specifying-links)。

- `Start timestamp`：默认为当前时间。该参数应该在传入span创建时间后设置。如果在Span逻辑启动时调用API，则API使用者不能显式地设置该参数。

每个span有0或一个父span，以及0或多个子spans(通常用于关联多个操作)。相关的span树组成了一条trace。如果一个span没有父span，则它是根span。每个trace包含一个根span，作为trace中其他spans的共同祖先。实现中必须提供一个选项来创建根span，且必须为每个创建的根span生成一个新的`TraceId`。对于一个有父span的Span，其`TraceId`必须与父span的`TraceId`相同。即子span默认必须继承父span的所有`TraceState`。

如果一个Span的子span是由另外一个流程创建的，就说`Span`包含远端父span。每个propagator的反序列化必须在父`SpanContext`上将`IsRemote`设置为true，这样就能知道一个Span的父Span是否是远端创建的。

所有创建的span必须能够结束，由用户负责保证。如果用户忘记结束span，则可能导致内存或其他资源(如用于迭代所有span的CPU周期)的泄露。

##### 从一个Context确定父Span

当从一个`Context`创建一个新的`Span`时，则该`Context`可能会包含一个表示当前活动实例的`Span`，且该`Span`会作为父span。如果`Context`中没有`Span`，则新创建的Span将会作为根span。

不能在Context中直接将`SpanContext`设置为活动状态，但可以[在Span中封装SpanContext](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#wrapping-a-spancontext-in-a-span)。例如，Propagator可能需要这些封装的方法来执行上下文的抽取。

##### 指定links

在`Span`创建期间，用户必须能够记录到其他Span的链接。链接的Span可以来自相同或不同的trace。在Span创建之后就不能添加`Link`s。

一个`Link`包含如下属性：

- 连接的`Span`的`SpanContext`
- 描述连接的0或多个[`Attributes`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)

Span创建API必须提供：

- 一个用于记录单个`Link`的API，以`Link`属性作为参数。该API可能叫作`AddLink`。该API将`Span`的`SpanContext`(以及可选的`Attributes`)作为单个参数或封装它们的不可变对象进行链接，

Links 应该保持设置它们的顺序。

#### Span操作

除了用于检索`Span`的`SpanContext`和记录状态的函数之外，在span完成之后，将不会调用以下任何一项：

##### 获取Context

Span接口必须提供：

- API会为给定`Span`返回`SpanContext`。该返回值可能在`Span`结束之后继续使用。返回的值必须在整个Span的生命周期中保持不变。这个API可能称为`GetContext`。

##### IsRecording

如果`Span`正在记录(如使用`AddEvent`操作event，使用`SetAttributes`设置属性，使用`SetStatus`设置状态等)信息，则返回true。在Span结束后，通常会变为无记录，此时`IsRecording`应该因此为结束的Span返回false。注意：流式实现中无法判定一个span是否结束，这是一种在Span结束之后`IsRecording`的返回值不能变的场景。

`IsRecording`不应该接收任何参数。

该标志用于避免对一个Span属性进行大量计算或当一个Spa

在整个trace都被采集的情况下，这个标志可能是`true`，通过这种方式可以允许记录和处理关于单个Span的信息，而无需将其发往后端。一个例子可能是记录和处理所有传入请求，用于处理和构建SLA / SLO延迟图，同时仅将一个子集（采样范围）发送到后端。

##### 设置属性

Span必须能够设置与它相关的[`Attributes`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)。

Span接口必须提供：

- 一个用于设置单个`Attribute`(该属性作为参数传入)的接口，可能称为`SetAttribute`。为了避免额外的分配，实现中可能为每种可能的值类型提供单独的API。

使用与已存在属性相同的key设置属性时，应该覆盖现有属性的值。

注意，OpenTelemetry项目记录了某些具有特定语义的[标准属性](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/README.md)。

注意，[Samplers](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampler)只考虑在创建span期间出现的信息，在这之后出现的任何变更，包括新增或修改的属性，都不能改变采样的Decision(参见[Samplers](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampler))。

##### 添加事件

一个`Span`必须能够添加事件。事件需要包含一个时间值，为该事件添加到`Span`的时间。

一个`Event`在结构上由以下属性定义：

- 事件名称
- 时间戳，为添加的时间或用户自定义的时间戳。
- 描述时间的0或多个 [`Attributes`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/common/common.md#attributes)。

Span接口必须提供：

- 用于记录单个`Event`的API，`Event`属性作为参数传递，API可能被称为`AddEvent`。该API将事件名称，可选的`Attributes`和一个可选的`Timestamp`(用于指定事件发生的时间)作为单独的参数，或封装了它们的不可变对象作为参数。如果用户没有提供自定义的时间戳，则事件中应该自动将时间设置为API调用事件的时刻。

事件应该保持它们被记录的顺序，通常会与时间的时间戳进行匹配。但使用用户自定义的时间戳时，事件可能是乱序的。

当用户提供了自定义时间戳或当开始或结束span时，消费者应该注意事件的时间戳可能会早于span的创建或晚于span的结束。本规范并没有要求标准化乱序的时间戳。

注意，OpenTelemetry项目记录了某些具有特定语义的[标准事件名称和keys](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/README.md)。

注意：[`RecordException`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#record-exception)是一个`AddEvent`的特殊变体，用于记录异常事件。

##### 设置状态

设置`Span`的`Status`。如果使用了该API则会覆盖默认的`Span`状态，即`Unset`。

`Status`在结构上由以下属性定义：

- `StatusCode`，下面列出的数值之一。
- 可选的`Description`，作为`Status`的描述消息

`StatusCode`为如下某个值：

- `Unset`：默认的状态
- `Ok`：由Application开发者或操作者确认该操作已成功完成。
- `Error`：操作包含错误。

Span接口必须提供：

- 一个用于设置`Status`的API，API可能被称为`SetStatus`。该API将StatusCode，可选的Description作为单独的参数，或封装了它们的不可变对象作为参数。

除非发生如下情况，否则状态码应该保持为unset：

当状态被工具库设置为`ERROR`时，应该记录状态码，且状态码具有可预测性。状态码只能根据语义规范中定义的规则将其设置为ERROR。对于没有被语义规范涵盖的操作，工具库应该发布自己的规则，包括状态码。

通常，除非经过明确配置，否则工具库不应该将状态码设置为`Ok`，且除非发生错误，工具库应该将状态码保持为`Unset`。

应用开发者和操作者可能会将状态码设置为`Ok`。

分析工具应该通过抑制可能产生的任何错误来响应`Ok`状态。例如，抑制繁杂的404错误。

只有最后一个调用的状态值才会被记录，实现时可以忽略前面的调用。

##### 更新名称

更新`Span`的名称。在更新之后，任何基于`Span`名称的采样行为都取决于具体的实现。

注意 [Samplers](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/sdk.md#sampler)仅考虑在span创建期间出现的信息。在这之后出现的任何变更，包括新增或修改的属性，都不能改变采样的Decision。

名称更新的备选方案可能是在后期创建Span，当Span使用过去的时间戳开始时(此时最终的Span名称是已知的)或报告具有期望名称的子Span时。

必需的参数：

- 新的span名称，它将取代`Span`开始时传递的内容

##### End

描述当前(或指定时间的)span结束的信号。

实现时应该忽略所有后续对`End`的调用和其他Span方法。即Span在结束之后变为了无记录状态(也有例外，如当Tracer是一个流事件，且没有与`Span`关联的可变状态)。

语言SIGs可以在API中提供除End之外的方法，这些方法也可以结束span，用于支持特定于语言的特性，比如Python中的语句。

`End`不能对子span产生任何影响，这些子span可以继续运行，后续再结束。

`End`不能将任何`Context`中活动的`Span`变为非活动，且必须可以通过包含的Context将结束的span用作父span。 另外，在span结束后，用于将span放入Context 的机制都必须仍然有效。

参数：

- 可选，指定结束的时间戳。如果忽略该参数，则以当前时间为准。

该API必需是非阻塞的。

##### 记录异常

为了便于记录异常，如果语言使用了异常，则应该提供一个`RecordException`方法。它是[`AddEvent`](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/api.md#add-events)的特殊变体，对于此处未指定的任何内容，都适用与`AddEvent`相同的要求。

该方法的签名取决于各自的语言，可以适当地重载。该方法必须使用 [异常语义规范](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/trace/semantic_conventions/exceptions.md)中的规范来记录异常。API需要的最少的参数要求为一个异常对象。

如果提供了`RecordException`，则该方法必须接受一个可选的参数来提供附加的事件属性(应该使用与`AddEvent`方法相同的方式)。如果属性使用的名称已经被使用过，则优先使用附加的属性。

注意：`RecordException`可以看作是`AddEvent`的变体，包含额外的异常参数以及其他可选的参数(因为它们具有默认的异常语义约定)。

#### Span生存期

Span的生存期表示记录Span对象的开始和结束时间戳的过程。

- 当创建Span时记录开始时间
- 当操作结束后记录结束时间

开始和结束时间与事件的时间戳一样，必须在调用对应的API时记录时间。

#### 封装Span中的SpanContext 

API必须提供一个操作来使用实现了Span接口的对象来封装SpanContext 。这样做的目的是为了在操作中(如流程内的`Span`传播)将`SpanContext`作为一个`Span`暴露出去。

如果需要新的类型来支持这类操作，则应该尽量不公开该类型 (例如仅通过暴露一个函数来返回对应Span接口类型的内容)。如果需要公开一个新的类型，则该类型应该命名为`NonRecordingSpan`。

定义的行为如下：

- `GetContext()`必须返回封装的`SpanContext`。
- `IsRecording`必须返回`false`来表示事件、属性和其他元素没有被记录，即被丢弃了。

Span的其余功能必须定义为无操作(no-op)的操作。注意：这包含End，因此，作为一般规则中的一个例外，结束这样一个Span是不必要的(甚至没有帮助)。

API必须实现该功能，且不应该被重写。

### SpanKind

`SpanKind`描述了一个Trace中的Span，父span，子span之间的关系。SpanKind描述了独立的属性，有利于跟踪系统的分析。

`SpanKind`描述的第一个属性反应了一个Span是远端的子span还是父span。关注带远端的父span的Spans，是因为它们是外部负载的来源。关注带远端子span的Spans，是因为它反应了一个非本地系统的依赖性。

`SpanKind`描述的第二个属性反应了一个子span是否是一个同步调用。当一个子span是同步时，一般情况下会等待父span结束。由于同步span可能会导致总体跟踪延迟，因此一个跟踪系统有必要了解这个属性。异步场景可以是远程的，也可以是本地的。

为了使`SpanKind`更有意义，调用者应该不应该赋予单个Span多个目的。例如，一个服务侧的span可能不会直接作为另一个远端span的父span。作为一个简单的指南，工具应该在为远端调用提取和序列化SpanContext 前就创建一个新的Span。

下面是可能的SpanKinds：

- `SERVER` ：表示span涵盖同步RPC或其他请求的服务器端处理。该span作为等待响应的远程`CLIENT` span的子span。
- `CLIENT` ：表示span描述了一个到远端服务的同步请求。该span作为远程`SERVER`  span的父span，并等待响应。
- `PRODUCER`： 表示span描述了一个异步请求的父span。该父span期望在对应的子`CONSUMER` span前结束，甚至在子span开始前结束。在使用批量消息传递的场景中，跟踪单个消息需要为每个要创建的消息创建一个新的`PRODUCER` span。
- `CONSUMER`： 表示span描述了一个异步`PRODUCER`请求的子span。
- `INTERNAL` ：默认值。表示span代表了一个应用的内部操作，与远程父操作或子操作相反。

下面是对这些类型的总结：

|            |             |              |                 |                 |
| ---------- | ----------- | ------------ | --------------- | --------------- |
| `SpanKind` | Synchronous | Asynchronous | Remote Incoming | Remote Outgoing |
| `CLIENT`   | yes         |              |                 | yes             |
| `SERVER`   | yes         |              | yes             |                 |
| `PRODUCER` |             | yes          |                 | maybe           |
| `CONSUMER` |             | yes          | maybe           |                 |
| `INTERNAL` |             |              |                 |                 |

### 并发

对于支持并发执行的语言，在执行Tracing APIs时需要提供保证和安全性。并不是所有的API功能都可以被并发地调用。

**TracerProvider** - 所有的方法都可以并发调用

**Tracer** -  所有的方法都可以并发调用

**Span** - Span的 所有方法都可以并发调用

**Event** - Events是不可变的，可以被并发使用

**Link** - Links是不可变的，可以被并发使用

### 包含的Propagators

API层或扩展包必须包含如下`Propagator`s：

- 实现了[W3C TraceContext Specification](https://www.w3.org/TR/trace-context/)的`TextMapPropagator`

查看 [Propagators Distribution](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#propagators-distribution) 来了解propagators是如何被分发的