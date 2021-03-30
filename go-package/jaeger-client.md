### jaeger

```go
package main

import (
   "context"
   "fmt"
   "github.com/opentracing/opentracing-go"
   "github.com/uber/jaeger-client-go"
   "github.com/uber/jaeger-client-go/config"
   "io"
   "time"
)
//初始化tracer
func initJaeger(service string) (opentracing.Tracer, io.Closer) {
   cfg := &config.Configuration{
      ServiceName: service,
       //每次都菜鸡
      Sampler: &config.SamplerConfig{
         Type:  "const",
         Param: 1,
      },
       //上报地址
      Reporter: &config.ReporterConfig{
         LogSpans:           true,
         LocalAgentHostPort: "192.168.225.217:6831",
      },
   }
   tracer, closer, err := cfg.NewTracer(config.Logger(jaeger.StdLogger))
   if err != nil {
      panic(fmt.Sprintf("ERROR: cannot init Jaeger: %v\n", err))
   }
   return tracer, closer
}

func foo3(req string, ctx context.Context) (reply string) {
   //1.创建子span
   span, _ := opentracing.StartSpanFromContext(ctx, "span_foo3")
   defer func() {
      //4.接口调用完，在tag中设置request和reply
      span.SetTag("request", req)
      span.SetTag("reply", reply)
      span.Finish()
   }()

   println(req)
   //2.模拟处理耗时
   time.Sleep(time.Second / 2)
   //3.返回reply
   reply = "foo3Reply"
   return
}

//跟foo3一样逻辑
func foo4(req string, ctx context.Context) (reply string) {
   span, _ := opentracing.StartSpanFromContext(ctx, "span_foo4")
   defer func() {
      span.SetTag("request", req)
      span.SetTag("reply", reply)
      span.Finish()
   }()

   println(req)
   time.Sleep(time.Second / 2)
   reply = "foo4Reply"
   return
}

func main() {
   tracer, closer := initJaeger("jaeger-demo")
   defer closer.Close()
   opentracing.SetGlobalTracer(tracer) //StartspanFromContext创建新span时会用到

   span := tracer.StartSpan("span_root")
   ctx := opentracing.ContextWithSpan(context.Background(), span)
   r1 := foo3("Hello foo3", ctx)
   r2 := foo4("Hello foo4", ctx)
   fmt.Println(r1, r2)
   span.Finish()

}
```







```go
// Configuration configures and creates Jaeger Tracer
 可以通方法config.FromEnv()
type Configuration struct {
   // ServiceName说明一个tracer的服务名，环境变量JAEGER_SERVICE_NAME
   ServiceName string `yaml:"serviceName"`

   // JAEGER_DISABLED
   Disabled bool `yaml:"disabled"`

   // JAEGER_RPC_METRICS
   RPCMetrics bool `yaml:"rpc_metrics"`

   // JAEGER_TAGS
   Tags []opentracing.Tag `yaml:"tags"`
	// 采集器
   Sampler             *SamplerConfig             `yaml:"sampler"`
    // 上报配置
   Reporter            *ReporterConfig            `yaml:"reporter"`
    // 头配置
   Headers             *jaeger.HeadersConfig      `yaml:"headers"`
    // 上游携带到下游的配置
   BaggageRestrictions *BaggageRestrictionsConfig `yaml:"baggage_restrictions"`
    // ThrottlerConfig配置可用于限制，客户端可以发送调试请求的速率。
   Throttler           *ThrottlerConfig           `yaml:"throttler"`
}


// 初始化tracer,closer可以用来在关闭服务前冲刷缓冲区
func (c Configuration) NewTracer(options ...Option) (opentracing.Tracer, io.Closer, error) {
	// 禁用
    if c.Disabled {
		return &opentracing.NoopTracer{}, &nullCloser{}, nil
	}
	
	if c.ServiceName == "" {
		return nil, nil, errors.New("no service name provided")
	}
	// 加载选项
	opts := applyOptions(options...)
	tracerMetrics := jaeger.NewMetrics(opts.metrics, nil)
    // 打开标量
	if c.RPCMetrics {
		Observer(
			rpcmetrics.NewObserver(
				opts.metrics.Namespace(metrics.NSOptions{Name: "jaeger-rpc", Tags: map[string]string{"component": "jaeger"}}),
				rpcmetrics.DefaultNameNormalizer,
			),
		)(&opts) // adds to c.observers
	}
    // 默认采样0.01概率
	if c.Sampler == nil {
		c.Sampler = &SamplerConfig{
			Type:  jaeger.SamplerTypeRemote,
			Param: defaultSamplingProbability,
		}
	}
	if c.Reporter == nil {
		c.Reporter = &ReporterConfig{}
	}

	sampler := opts.sampler
	if sampler == nil {
        // 创建sampler
		s, err := c.Sampler.NewSampler(c.ServiceName, tracerMetrics)
		if err != nil {
			return nil, nil, err
		}
		sampler = s
	}

	reporter := opts.reporter
	if reporter == nil {
        // 创建reporter
		r, err := c.Reporter.NewReporter(c.ServiceName, tracerMetrics, opts.logger)
		if err != nil {
			return nil, nil, err
		}
		reporter = r
	}
	
	tracerOptions := []jaeger.TracerOption{
		jaeger.TracerOptions.Metrics(tracerMetrics),
		jaeger.TracerOptions.Logger(opts.logger),
		jaeger.TracerOptions.CustomHeaderKeys(c.Headers),
		jaeger.TracerOptions.Gen128Bit(opts.gen128Bit),
		jaeger.TracerOptions.PoolSpans(opts.poolSpans),
		jaeger.TracerOptions.ZipkinSharedRPCSpan(opts.zipkinSharedRPCSpan),
		jaeger.TracerOptions.MaxTagValueLength(opts.maxTagValueLength),
		jaeger.TracerOptions.NoDebugFlagOnForcedSampling(opts.noDebugFlagOnForcedSampling),
	}
	// 处理tag
	for _, tag := range opts.tags {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.Tag(tag.Key, tag.Value))
	}

	for _, tag := range c.Tags {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.Tag(tag.Key, tag.Value))
	}
	
	for _, obs := range opts.observers {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.Observer(obs))
	}

	for _, cobs := range opts.contribObservers {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.ContribObserver(cobs))
	}

	for format, injector := range opts.injectors {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.Injector(format, injector))
	}

	for format, extractor := range opts.extractors {
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.Extractor(format, extractor))
	}

	if c.BaggageRestrictions != nil {
		mgr := remote.NewRestrictionManager(
			c.ServiceName,
			remote.Options.Metrics(tracerMetrics),
			remote.Options.Logger(opts.logger),
			remote.Options.HostPort(c.BaggageRestrictions.HostPort),
			remote.Options.RefreshInterval(c.BaggageRestrictions.RefreshInterval),
			remote.Options.DenyBaggageOnInitializationFailure(
				c.BaggageRestrictions.DenyBaggageOnInitializationFailure,
			),
		)
		tracerOptions = append(tracerOptions, jaeger.TracerOptions.BaggageRestrictionManager(mgr))
	}

	if c.Throttler != nil {
		debugThrottler := throttler.NewThrottler(
			c.ServiceName,
			throttler.Options.Metrics(tracerMetrics),
			throttler.Options.Logger(opts.logger),
			throttler.Options.HostPort(c.Throttler.HostPort),
			throttler.Options.RefreshInterval(c.Throttler.RefreshInterval),
			throttler.Options.SynchronousInitialization(
				c.Throttler.SynchronousInitialization,
			),
		)

		tracerOptions = append(tracerOptions, jaeger.TracerOptions.DebugThrottler(debugThrottler))
	}
	// 处理所有的选项
	tracer, closer := jaeger.NewTracer(
		c.ServiceName,
		sampler,
		reporter,
		tracerOptions...,
	)

	return tracer, closer, nil
}
```





```go
// StartSpanFromContext starts and returns a Span with `operationName`, using
// any Span found within `ctx` as a ChildOfRef. If no such parent could be
// found, StartSpanFromContext creates a root (parentless) Span.
//
// The second return value is a context.Context object built around the
// returned Span.
//
// Example usage:
//
//    SomeFunction(ctx context.Context, ...) {
//        sp, ctx := opentracing.StartSpanFromContext(ctx, "SomeFunction")
//        defer sp.Finish()
//        ...
//    }
func StartSpanFromContext(ctx context.Context, operationName string, opts ...StartSpanOption) (Span, context.Context) {
   return StartSpanFromContextWithTracer(ctx, GlobalTracer(), operationName, opts...)
}

// StartSpanFromContextWithTracer starts and returns a span with `operationName`
// using  a span found within the context as a ChildOfRef. If that doesn't exist
// it creates a root span. It also returns a context.Context object built
// around the returned span.
//
// It's behavior is identical to StartSpanFromContext except that it takes an explicit
// tracer as opposed to using the global tracer.
func StartSpanFromContextWithTracer(ctx context.Context, tracer Tracer, operationName string, opts ...StartSpanOption) (Span, context.Context) {
    // 从context中提取parent信息
	if parentSpan := SpanFromContext(ctx); parentSpan != nil {
		opts = append(opts, ChildOf(parentSpan.Context()))
	}
	span := tracer.StartSpan(operationName, opts...)
	return span, ContextWithSpan(ctx, span)
}


// SpanFromContext returns the `Span` previously associated with `ctx`, or
// `nil` if no such `Span` could be found.
//
// NOTE: context.Context != SpanContext: the former is Go's intra-process
// context propagation mechanism, and the latter houses OpenTracing's per-Span
// identity and baggage information.
var activeSpanKey = contextKey{}
func SpanFromContext(ctx context.Context) Span {
	val := ctx.Value(activeSpanKey)
	if sp, ok := val.(Span); ok {
		return sp
	}
	return nil
}
```