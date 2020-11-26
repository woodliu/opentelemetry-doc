## Collector的配置和使用

[TOC]

### Collector配置

collector通过pipeline处理[service](https://opentelemetry.io/docs/collector/configuration/#service)中启用的数据。pipeline由接收遥测数据的组件构成，包括：

- [Receivers](https://opentelemetry.io/docs/collector/configuration/#receivers)
- [Processors](https://opentelemetry.io/docs/collector/configuration/#processors)
- [Exporters](https://opentelemetry.io/docs/collector/configuration/#exporters)

其次还可以通过扩展来为Collector添加功能，但扩展不需要直接访问遥测数据，且不是pipeline的一部分。扩展同样可以在[service](https://opentelemetry.io/docs/collector/configuration/#service)中启用。

#### Receivers

receiver定义了数据如何进入OpenTelemetry Collector。必须配置一个或多个receiver，默认不会配置任何receivers。

下面给出了所有可用的receivers的基本例子，更多配置可以参见[receiver文档](https://github.com/open-telemetry/opentelemetry-collector/blob/master/receiver/README.md)。

```yaml
receivers:
  opencensus:
    address: "localhost:55678"

  zipkin:
    address: "localhost:9411"

  jaeger:
    protocols:
      grpc:
      thrift_http:
      thrift_tchannel:
      thrift_compact:
      thrift_binary:

  prometheus:
    config:
      scrape_configs:
        - job_name: "caching_cluster"
          scrape_interval: 5s
          static_configs:
            - targets: ["localhost:8889"]
```

#### Processors

Processors运行在数据的接收和导出之间。虽然Processors是可选的，但有时候会建议使用Processors。

下面给出了所有可用的Processors的基本例子，更多参见[Processors文档](https://github.com/open-telemetry/opentelemetry-collector/blob/master/processor/README.md)。

```yaml
processors:
  attributes/example:
    actions:
      - key: db.statement
        action: delete
  batch:
    timeout: 5s
    send_batch_size: 1024
  probabilistic_sampler:
    disabled: true
  span:
    name:
      from_attributes: ["db.svc", "operation"]
      separator: "::"
  queued_retry: {}
  tail_sampling:
    policies:
      - name: policy1
        type: rate_limiting
        rate_limiting:
          spans_per_second: 100
```

#### Exporters

exporter指定了如何将数据发往一个或多个后端/目标。必须配置一个或多个exporter，默认不会配置任何exporter。

下面给出了所有可用的exporters的基本例子，更多参见[exporters文档](https://github.com/open-telemetry/opentelemetry-collector/blob/master/exporter/README.md)。

```yaml
exporters:
  opencensus:
    headers: {"X-test-header": "test-header"}
    compression: "gzip"
    cert_pem_file: "server-ca-public.pem" # optional to enable TLS
    endpoint: "localhost:55678"
    reconnection_delay: 2s

  logging:
    loglevel: debug

  jaeger_grpc:
    endpoint: "http://localhost:14250"

  jaeger_thrift_http:
    headers: {"X-test-header": "test-header"}
    timeout: 5
    endpoint: "http://localhost:14268/api/traces"

  zipkin:
    endpoint: "http://localhost:9411/api/v2/spans"

  prometheus:
    endpoint: "localhost:8889"
    namespace: "default"
```

#### Service

Service部分用于配置OpenTelemetry Collector根据receivers, processors, exporters, 和extensions sections的配置会启用那些特性。service分为两部分：

- extensions
- pipelines

extensions包含启用的扩展，如：

```yaml
    service:
      extensions: [health_check, pprof, zpages]
```

Pipelines有两类：

- metrics: 采集和处理metrics数据
- traces: 采集和处理trace数据

一个pipeline是一组 receivers, processors, 和exporters的集合。必须在service之外定义每个receiver/processor/exporter的配置，然后将其包含到pipeline中。

注：每个receiver/processor/exporter都可以用到多个pipeline中。当多个pipeline引用processor(s)时，每个pipeline都会获得该processor(s)的一个实例，这与多个pipeline中引用receiver(s)/exporter(s)的情况不同(所有pipelines仅能获得receiver/exporter的一个实例)。

下面给出了一个pipeline配置的例子，更多可以参见[pipeline文档](https://github.com/open-telemetry/opentelemetry-collector/blob/master/docs/pipelines.md)。

```yaml
service:
  pipelines:
    metrics:
      receivers: [opencensus, prometheus]
      exporters: [opencensus, prometheus]
    traces:
      receivers: [opencensus, jaeger]
      processors: [batch, queued_retry]
      exporters: [opencensus, zipkin]
```

#### Extensions

Extensions可以用于监控OpenTelemetry Collector的健康状态。Extensions是可选的，默认不会配置任何Extensions。

下面给出了所有可用的extensions的基本例子，更多参见[extensions文档](https://github.com/open-telemetry/opentelemetry-collector/blob/master/extension/README.md)。

```yaml
extensions:
  health_check: {}
  pprof: {}
  zpages: {}
```

#### 使用环境变量

collector配置中可以使用环境变量，如：

```yaml
processors:
  attributes/example:
    actions:
      - key: "$DB_KEY"
        action: "$OPERATION"
```

### Collector的使用

下面使用[官方demo](https://github.com/open-telemetry/opentelemetry-go/tree/master/example/otel-collector)来体验一下Collector的功能

本例展示如何从OpenTelemetry-Go SDK 中导出trace和metric数据，并将其导入OpenTelemetry Collector，最后通过Collector将trace数据传递给Jaeger，将metric数据传递给Prometheus。完整的流程为：

```
                                          -----> Jaeger (trace)
App + SDK ---> OpenTelemtry Collector ---|
                                          -----> Prometheus (metrics)
```

#### 部署到Kubernetes

[k8s](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/k8s)目录中包含本demo所需要的所有部署文件。为了简化方便，官方将部署目录集成到了一个[makefile](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/Makefile)文件中。在必要时可以手动执行Makefile中的命令。

#### 部署Prometheus operator

```shell
git clone https://github.com/coreos/kube-prometheus.git
cd kube-prometheus
kubectl create -f manifests/setup

# wait for namespaces and CRDs to become available, then
kubectl create -f manifests/
```

可以使用如下方式清理环境：

```shell
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
```

等待prometheus所有组件变为running状态

```shell
# kubectl get pod -n monitoring
NAME                                   READY   STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2     Running   0          16m
alertmanager-main-1                    2/2     Running   0          16m
alertmanager-main-2                    2/2     Running   0          16m
grafana-7f567cccfc-4pmhq               1/1     Running   0          16m
kube-state-metrics-85cb9cfd7c-x6kq6    3/3     Running   0          16m
node-exporter-c4svg                    2/2     Running   0          16m
node-exporter-n6tnv                    2/2     Running   0          16m
prometheus-adapter-557648f58c-vmzr8    1/1     Running   0          16m
prometheus-k8s-0                       3/3     Running   0          16m
prometheus-k8s-1                       3/3     Running   1          16m
prometheus-operator-5b469f4f66-qx2jc   2/2     Running   0          16m
```

#### 使用Makefile

下面使用[makefile](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/Makefile)部署Jaeger，Prometheus monitor和Collector，依次执行如下命令即可：

```shell
# Create the namespace
make namespace-k8s

# Deploy Jaeger operator
make jaeger-operator-k8s

# After the operator is deployed, create the Jaeger instance
make jaeger-k8s

# Then the Prometheus instance. Ensure you have enabled a Prometheus operator
# before executing (see above).
make prometheus-k8s

# Finally, deploy the OpenTelemetry Collector
make otel-collector-k8s
```

等待`observability`命名空间下的Jaeger和Collector的Pod变为running状态

```shell
# kubectl get pod -n observability
NAME                              READY   STATUS    RESTARTS   AGE
jaeger-7b868df4d6-w4tk8           1/1     Running   0          97s
jaeger-operator-9b4b7bb48-q6k59   1/1     Running   0          110s
otel-collector-7cfdcb7658-ttc8j   1/1     Running   0          14s
```

可以使用`make clean-k8s`命令来清理环境，但该命令不会移除命名空间，需要手动删除命名空间：

```shell
kubectl delete namespaces observability
```

#### 配置OpenTelemetry Collector

完成上述步骤之后，就部署好了所需要的所有资源。下面看一下Collector的[配置文件](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/k8s/otel-collector.yaml)：

为了使应用发送数据到OpenTelemetry Collector，首先需要配置`otlp`类型的receiver，它使用gRpc进行通信：

```yaml
...
  otel-collector-config: |
    receivers:
      # Make sure to add the otlp receiver.
      # This will open up the receiver on port 55680.
      otlp:
        endpoint: 0.0.0.0:55680
    processors:
...
```

上述配置会在Collector侧创建receiver，并打开`55680`端口，用于接收trace。剩下的配置都比较标准，唯一需要注意的是需要创建Jaeger和Prometheus exporters:

```yaml
...
    exporters:
      jaeger_grpc:
        endpoint: "jaeger-collector.observability.svc.cluster.local:14250"

      prometheus:
           endpoint: 0.0.0.0:8889
           namespace: "testapp"
...
```

##### OpenTelemetry Collector service

[配置](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/k8s/otel-collector.yaml)中另外一个值得注意的是用于访问OpenTelemetry Collector的NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
        ...
spec:
  ports:
  - name: otlp # Default endpoint for otlp receiver.
    port: 55680
    protocol: TCP
    targetPort: 55680
    nodePort: 30080
  - name: metrics # Endpoint for metrics from our app.
    port: 8889
    protocol: TCP
    targetPort: 8889
  selector:
    component: otel-collector
  type:
    NodePort
```

该service 会将用于访问otlp receiver的30080端口与cluster node的55680端口进行绑定，这样就可以通过静态地址`<node-ip>:30080`来访问Collector。

#### 运行代码

在[main.go](https://github.com/open-telemetry/opentelemetry-go/blob/master/example/otel-collector/main.go)文件中可以看到完整的示例代码。要运行该代码，需要满足Go的版本>=1.13

```shell
# go run main.go
2020/10/20 09:19:17 Waiting for connection...
2020/10/20 09:19:17 Doing really hard work (1 / 10)
2020/10/20 09:19:18 Doing really hard work (2 / 10)
2020/10/20 09:19:19 Doing really hard work (3 / 10)
2020/10/20 09:19:20 Doing really hard work (4 / 10)
2020/10/20 09:19:21 Doing really hard work (5 / 10)
2020/10/20 09:19:22 Doing really hard work (6 / 10)
2020/10/20 09:19:23 Doing really hard work (7 / 10)
2020/10/20 09:19:24 Doing really hard work (8 / 10)
2020/10/20 09:19:25 Doing really hard work (9 / 10)
2020/10/20 09:19:26 Doing really hard work (10 / 10)
2020/10/20 09:19:27 Done!
2020/10/20 09:19:27 exporter stopped
```

该示例模拟了一个正在运行应用程序，计算10秒之后结束。

#### 查看采集到的数据

运行`go run main.go`的数据流如下：![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201027103729559-733401015.png)

##### Jaeger UI

Jaeger上查询trace内容如下:

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201027103853848-1592674717.png)

##### Prometheus

运行main.go结束之后，可以在Prometheus中查看该metric。其对应的Prometheus target为`observability/otel-collector/0`

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201026171031896-459681643.png)

Prometheus上查询metric内容如下：

![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201027103044425-1333539720.png)

### FAQ：

- 在运行完部署命令之后，发现Prometheus没有注册如http://10.244.1.33:8889/metrics这样的target。可以查看Prometheus pod的日志，可能是因为Prometheus没有对应的role权限导致的，将Prometheus的clusterrole修改为如下内容即可：

  ```yaml
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: prometheus-k8s
    namespace: monitoring
  rules:
  - apiGroups: [""]
    resources: ["services","pods","endpoints","nodes/metrics"]
    verbs: ["get", "watch", "list"]
  - apiGroups: ["extensions"]
    resources: ["ingresses"]
    verbs: ["get", "watch", "list"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get", "watch", "list"]
  ```

- 在运行"go run main.go"时可能会遇到`rpc error: code = Internal desc = grpc: error unmarshalling request: unexpected EOF`这样的错误，通常因为client和server使用的proto不一致导致的。client端(即[main.go](https://github.com/open-telemetry/opentelemetry-go/tree/master/example/otel-collector))使用的proto文件目录为`go.opentelemetry.io/otel/exporters/otlp/internal/opentelemetry-proto-gen`，而[collector](https://github.com/open-telemetry/opentelemetry-collector)使用proto文件目录为`go.opentelemetry.io/collector/internal/data/opentelemetry-proto-gen`，需要比较这两个目录下的文件是否一致。如果不一致，则需要根据collector的版本为[main.go](https://github.com/open-telemetry/opentelemetry-go/tree/master/example/otel-collector)生成对应的proto文件(或者可以直接更换collector的镜像，注意使用的[otel/opentelemetry-collector](otel/opentelemetry-collector)的镜像版本)。在collector的[proto目录](https://github.com/open-telemetry/opentelemetry-collector/tree/v0.13.0/internal/data)下可以看到对应的注释和使用的proto版本，如下：

  ![](https://img2020.cnblogs.com/blog/1334952/202010/1334952-20201026173346122-994107466.png)

  collector使用的proto git库为[opentelemetry-proto](https://github.com/open-telemetry/opentelemetry-proto)。clone该库，切换到对应版本后，执行`make gen-go`即可生成对应的文件。

  | Component                    | Maturity |
  | ---------------------------- | -------- |
  | **Binary Protobuf Encoding** |          |
  | collector/metrics/*          | Alpha    |
  | collector/trace/*            | Stable   |
  | common/*                     | Stable   |
  | metrics/*                    | Alpha    |
  | resource/*                   | Stable   |
  | trace/trace.proto            | Stable   |
  | trace/trace_config.proto     | Alpha    |
  | **JSON encoding**            |          |
  | All messages                 | Alpha    |