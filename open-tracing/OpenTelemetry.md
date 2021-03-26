OpenTelemetry

https://www.mmbyte.com/article/124104.html

https://github.com/open-telemetry

https://opentelemetry.io/docs/

如果刚接触Opentelemetry，那么需要了解如下术语：

- Traces：记录经过分布式系统的请求活动，一个trace是spans的有**向无环图**
- Spans：一个trace中表示一个命名的，基于时间的操作。Spans**嵌套形成trace树**。每个trace包含一个根span，描述了端到端的延迟，其子操作也可能拥有一个或多个子spans。
- Metrics：在运行时捕获的关于服务的原始度量数据。Opentelemetry定义的metric instruments(指标工具)如下。Observer支持通过异步API来采集数据，每个采集间隔采集一个数



- | ame               | Synchronous | Adding | Monotonic |
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

通过将OpenTracing 和OpenCensus **合并为一个开放的标准**，OpenTelemetry提供了如下便利：

- 选择简单：不必在两个标准之间进行选择，OpenTelemetry可以同时**兼容** OpenTracing和OpenCensus。
- 跨平台：OpenTelemetry 支持各种语言和后端。它代表了一种厂商中立的方式，可以在不改变现有工具的情况下捕获并将遥测数据传输到后端。
- 简化可观测性：正如OpenTelemetry所说的"高质量的观测下要求高质量的遥测"。希望看到更多的厂商转向OpenTelemetry，因为它更方便，且仅需测试单一标准。