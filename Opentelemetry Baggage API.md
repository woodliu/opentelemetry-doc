# Opentelemetry Baggage

[TOC]

## Baggage API

### 总览

Baggage API包括：

- `Baggage`
- 一个`Context`中与`Baggage`交互的函数

这些函数描述了一种完全通过Context与Baggage进行交互的方式。取决于具体的语言，在实现中API可能会通过表示整个Baggage内容的结构体或不可变对象来实现这些功能。然后可以通过单次操作从Context中添加或删除此结构。例如，[Clear](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/baggage/api.md#clear)函数可以用于在上下文中设置一个空的Baggage对象/结构体。[Get all](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/baggage/api.md#get-all)函数可以从函数调用返回整个Baggage对象。 如果实现了这样的用法，那么Baggage的对象/结构必须是不可变的，这样包含的Context也将保持不变。

Baggage API必须能够在不安装SDK的情况下正常使用，这对跨Baggage的透明传播来说是必需的。 如果将Baggage propagator安装到API中，则无论有没有安装SDK，它都可以使用。

#### Baggage

`Baggage`用来对遥测进行注释，以及给metrics, traces, 和logs添加上下文和信息。它是一个抽象数据类型，描述了用户自定义属性的一组name/value对。`Baggage`中的每个name必须与一个确切的value关联。

#### Get all

返回`Baggage`的name/value对，name/value对的顺序并不重要。取决于语言规范，返回的值可以是name/value对的集合的不变集合或不变的迭代器。

可选参数：

`Context`: 包含从哪里获得baggages的`Baggage`

#### Get baggage

通过先前的事件访问name/value对的value。Baggage API应该提供一个函数，使用一个context和一个name作为输入，并返回value。然会的value关联到给定的name，如果没有给定name，则返回null。

必需的参数：

`Name`：与返回的value关联的name

可选的参数：

`Context`：包含从哪里获得baggages表项的`Baggage`

#### Set baggage

为了记录一个name/value对的value，Baggage API应该提供一个函数，使用一个context，一个name，一个value作为输入，并返回一个包含(一个带新value的)`Baggage`的`Context`。

必需的参数：

`Name`：设置value的name，类型为string。

`Value`：设置的value，类型为string。

可选的参数：

Metadata：与name-value对关联的元数据。它应该是一个不透明的包装器，用于没有语义意义的字符串，用于扩展未来的功能。

`Context`：包含从哪里获得baggages表项的`Baggage`

#### Remove baggage

为了记录一个name/value对，Baggage API应该提供一个函数，使用一个context和一个name作为输入，并返回一个包含不再包含选定name的`Context`。

必需的参数：

`Name`：要移除的name。

可选的参数：

`Context`：包含要移除的baggages表项的`Baggage`

#### Clear

为了避免想不信任的处理发送任何name/value对，，Baggage API应该提供一个函数来移除一个上下文中的所有baggages表项，并返回一个不包含`Baggage`的`Context`。

### Baggage Propagation

`Baggage`可能跨处理边界或任意边界来传播(process，$OTHER_BOUNDARY1, $OTHER_BOUNDARY2等)。

API层或扩展包必须包含如下`Propagator`s：

- 实现[W3C Baggage 规范](https://w3c.github.io/baggage)的TextMapPropagator

查看[Propagators Distribution](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#propagators-distribution)来了解Propagator是如何分发的。

注意：W3C baggage规范目前没有为可选的元数据分配语义意义。

[extract](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#extract)时，propagator 应该将所有的元数据作为一个元数据实例表项；[inject](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#inject-1)时，propagator 应该按照W3C规范格式添加元数据。

注意：

如果propagator 无法解析`baggage`首部时，`extract`必须返回不带baggage 表项的Context。

如果出现`baggage`首部，但不包含任何表项时，`extract`必须返回不带baggage 表项的Context。

### 冲突解决

如果一个新添加的name/value的name与现有的name相同，则优先使用新的name/value对。新value会替换老value。

## Propagators API

横切关注点(Cross-cutting concerns)会使用`Propagator`s将状态发送到下一个处理阶段，`Propagator`被定义为一个从应用交互的信息中读写上下文的对象。每个关注点为每种支持的Propagator类型创建一组`Propagator`s 。

`Propagator`s 会利用`Context`来为每个横切关注点inject和extract数据(如traces 和`Baggage`)

传播通常是通过特定于库的请求拦截器和Propagators的合作来实现的，拦截器会检测入站和出站的请求，并分别使用`Propagator`的extract 和inject operations(解析见下文)。

用户可以使用Propagators API来编写工具库。

### Propagator Types

为了跨处理边界传播带内上下文数据，`Propagator`类型定义了特定传输的限制，并绑定到数据类型。

Propagators API目前定义了一种Propagator类型：

- `TextMapPropagator`：该类型可以给Carriers(解释见下)注入key/value字符串对，会从Carriers抽取key/value字符串对。

未来将会添加一个二进制的`Propagator`类型(参见[#437](https://github.com/open-telemetry/opentelemetry-specification/issues/437))。

#### Carrier

一个Carrier是`Propagator`s 使用的一个中间组件，用于读取和写入数据。每个特定的`Propagator`s类型都定义了预期的Carrier类型，如string映射或byte数组

#### Operations

为了从carriers分别写入数据和读取数据，`Propagator`s 必须定义`Inject`和`Extract`的操作。每个`Propagator`类型必须定义特定的carrier类型，可以定义额外的参数。

##### Inject

用于将数据注入一个carrier，例如将首部注入一个HTTP请求。

必需的参数：

- 一个`Context`：Propagator必需首先从Context中检索合适的值，如 `SpanContext`, `Baggage` 或其他cross-cutting concern上下文。
- 包含传播字段的carrier，如入站的消息或http响应。

##### Extract

从入站的请求中抽取数值，例如，HTTP请求的首部。

如果从carrier为cross-cutting concern解析某个数值，则实现中不能抛出异常，且不能在`Context`中保存新的数值，目的是为了保留任何先前存在的有效值。

必需的参数：

- 一个`Context`。
- 包含传播字段的carrier，如入站的消息或http响应。

根据传入的`Context`参数返回一个新的`Context`，包含抽取的值，可以是`SpanContext`, `Baggage` 或其他cross-cutting concern上下文。

### TextMap Propagator

`TextMapPropagator`会从carriers中注入或抽取数值(这些数值是跨处理边界传输的)，作为一个cross-cutting concern作为字符串的key/values对。

无论是客户端侧(写入)还是服务端侧(抽取)，carrier上传播的数据通常是HTTP请求。

为了增加兼容性，key/values对必须仅包含US-ASCII字符(HTTP首部字段使用这类编码的字符，参见[RFC 7230](https://tools.ietf.org/html/rfc7230#section-3.2))。

`Getter` 和`Setter`是可选的辅助函数，分别用于抽取和注入数据，它们定义为与carrier分开的对象，以避免运行时分配，这是因为不需要通过额外的carrier的接口实现对象来访问其内容。

`Getter`和`Setter`必须是无状态的，并且允许保存为常量，以便有效地避免运行时分配。

#### 字段

为前面定义的传播字段，如果要重用carrier，则在调用inject之前应该删除这些字段。

字段被定义为carrier中识别特定格式的字符串键。

例如，如果一个carrier为单独使用或不可变的请求对象时，则不需要清除字段；如果是可变的，可重试的对象，则后续调用时应该首先清除这些字段。

使用场景为：

- 允许实现分配字段，特别是如gRPC Metadata这类系统
- 允许迭代器的单次传递。

返回的字段将会被`TextMapPropagator`使用。

#### Inject

将值输入一个carrier，必需的参数与上述定义的[Inject](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#inject)相同。

可选参数为：

- 一个用于设置传播的key/value对的`Setter`。Propagators在设置多个对时可能会多次调用该函数。它一个额外的参数，可以用于将数据注入carrier。

##### Setter 参数

Setter 是Inject的一个参数，允许`TextMapPropagator`将传播字段设置到一个carrier中。下面描述了一个使用`Set`方法实现`Setter`类的方式。

###### Set

使用给定的值替换传播字段。

必需的参数：

- 持有传播字段的carrier。例如出站消息或一个HTTP请求
- 字段的key
- 字段的value

实现中应该保留大小写(如不应该将`Content-Type` 转变为`content-type`)，除非使用的协议对大小写不敏感，否则必需保留大小写。

#### Extract

从入站请求中抽取数值。必需的参数与上面定义的[Extract](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/context/api-propagators.md#extract)相同。

可选的参数：

- `Getter`，用于获取传播的key，这是一个可选的参数，不同语言可以定义各自的抽取数据辅助函数。

返回一个根据入参的`Context`生成的新`Context`。

##### Getter 参数

Getter是Extract的一个参数，允许`TextMapPropagator`从一个carrier中读取传播的字段。下面描述了一个使用`Get`和`Keys`方法实现`Getter`类的方式。

###### Keys

Keys函数必须返回carrier中的所有keys的列表。

必需的参数：

- carrier的传播字段，如一个HTTP请求

为了迭代特定carrier中的所有keys，`Propagator`s 可以使用可变的key名称来调用`Keys`函数。例如，可以使用`uberctx-{user-defined-key}`模式([Jaeger传播格式](https://www.jaegertracing.io/docs/1.18/client-libraries/#baggage).)来检测所有的keys。

###### Get

Get函数必需返回给定传播key的第一个值，如果不存在则返回null。

必需的参数：

- carrier的传播字段，如一个HTTP请求
- 字段的key

Get函数负责处理大小写敏感的问题。如果该函数要处理一个HTTP请求对象，则必须大小写敏感。

### Injectors和Extractors作为独立的接口

各语言在实现时，可以将一个`Propagator`类型作为一个暴露了`Inject` 和`Extract`方法的单独对象，或将其按职责划分为独立的`Injector`s 和`Extractor`s。一个`Propagator`可以实现为：包含独立的`Injector`s 和`Extractor`s。

### 复合Propagator

实现时，必须提供一个功能来将不同cross-cutting concerns的多个`Propagator`s 组合起来作为一个单独的实体。

一个复合`Propagator`可以由一组`Propagator`s 构成，或一组injectors和extractors。最后的复合`Propagator`将按照指定的顺序调用`Propagator`s, `Injector`s, or `Extractor`s。

每个复合Propagator都会实现一个特定的`Propagator`类型，如`TextMapPropagator`，不同的`Propagator`类型会操作不同的数据类型。

必须提供如下功能来支持如下操作：

- 创建一个复合的propagator
- 从复合propagator中执行Extract
- 将数据Inject到复合propagator

#### 创建一个复合propagator

必需的参数：

- 一组`Propagator`s ，或一组injectors和extractors

使用指定给的`Propagator`s返回一个新的`Propagator`。

#### Extract

必需的参数：

- 一个`Context`
- 包含传播字段的carrier
- 获取传播的key时调用的`Getter`实例

#### Inject

必需的参数：

- 一个`Context`
- 包含传播字段的carrier
- 设置传播key/value对的`Setter`。为了设置多个对，Propagators 可能会多次调用该函数。

### 全局的Propagators

OpenTelemetry API 必须提供一种方式来为每个支持的`Propagator`类型获取一个propagator 。工具库应该在所有远程调用中调用propagators 来提取和注入上下文。

注：特定的工具库可能会使用专有的上下文传播协议或通过硬编码指定一个协议(不建议这么做)。这种场景下，工具库可能不会使用API提供的propagators，且硬编码了上下文的抽取和注入逻辑。

除非明确配置，否则OpenTelemetry API必须使用无操作的propagators。上下文传播可以用于多个遥测信号-traces，metrics，logging等。因此，可以为任一个遥测信号单独启用上下文传播。例如，即使为logs或metrics启动用了跟踪上下文传播，但也可以不配置span exporter。

像 ASP.NET这样的平台可能预定义开箱即用的propagators。如果预定义了propagators，则propagators应该默认为一个包含W3C Trace Context Propagator和Baggage `Propagator` 的复合`Propagator`。这类平台必须允许禁用或覆盖预定义的propagators。

#### 获取Global Propagator

每个支持的`Propagator`类型必须包含该方法。

返回一个全局的`Propagator`，通常是一个复合实例。

#### 设置Global Propagator

每个支持的`Propagator`类型必须包含该方法。

必需的参数：

- 一个`Propagator`：通常是一个复合实例。

### Propagators 的发布

OpenTelemetry 组织包含的propagators官方列表必须作为OpenTelemetry 的扩展包发布：

- [W3C TraceContext](https://www.w3.org/TR/trace-context/). 可以作为OpenTelemetry API的一部分进行发布。
- [W3C Baggage](https://w3c.github.io/baggage). 可以作为OpenTelemetry API的一部分进行发布。
- [B3](https://github.com/openzipkin/b3-propagation).
- [Jaeger](https://www.jaegertracing.io/docs/latest/client-libraries/#propagation-format).

其他`Propagator`s 实现了特定于供应商的协议，如 AWS X-Ray trace首部协议，这类`Propagator`s 由对应的供应商或OpenTelemetry组织进行维护和发布，维护这些社区的原因是：

- Propagators 仅仅是一小段代码，其功能通常是公开记录的(与exporters不同)
- 用户在使用propagators 时，经常不会对应的tracing或metrics供应商。例如，tracing 供应商的客户可能希望使用特定云供应商的propagator来请求该云供应商的服务。
- 仅需要保留一小部分的propagators ，当供应商切换到W3C TraceContext后，propagators 的数量将会逐步减少。

#### B3 要求

B3同时具有单和多首部编码。为了在实现时达到最大兼容，为OpenTelemetry中的B3传播建立了以下准则：

##### Extract

Propagators 必须尝试使用单和多首部格式来抽取B3编码的数据。当抽取时，优先处理单首部版本。

##### Inject

当注入B3时，propagators将会：

- 必须默认使用单首部格式注入B3
- 提供提供配置来将默认的注入格式修改为B3多首部。