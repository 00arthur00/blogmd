title: prometheus exporter golang
date: 2019-08-13 14:00:34
tags:
  - prometheus
  - 监控
categories:
  - 监控
---
## 导出默认指标

``` golang
package main

import (
	"flag"
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var addr = flag.String("listen-address", ":8080", "The address to listen on for HTTP requests.")

func main() {
	flag.Parse()
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(*addr, nil))
}
```
核心代码只有一行:
``` golang
http.Handle("/metrics", promhttp.Handler())
```
`promhttp.Handler()`返回prometheus client中默认注册的handler,其导出默认的metrics,如go routine的数量等.

## 自定义指标
golang的prometheus client提供了一些方便的接口可以注册到默认的promhttp的handler中.

指标类型有:
* Counter: 一个单调递增值,重启的时候reset为0,如http请求数量
* Gauge: 一个变化值,可变大变小,如温度
* Histogram: 一个历史值,prometheus服务器端可以通过[histogram_quantile()函数](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)计算percentile
* Summary: 一个历史值,在客户端计算

通过New函数定义相应的metrics并用Set/Add/Observe等函数设置相应的值,并在访问exporter端口时将指标暴露出去.
如:
``` golang
package main

import (
	"flag"
	"log"
	"math"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
	addr              = flag.String("listen-address", ":8080", "The address to listen on for HTTP requests.")
	uniformDomain     = flag.Float64("uniform.domain", 0.0002, "The domain for the uniform distribution.")
	normDomain        = flag.Float64("normal.domain", 0.0002, "The domain for the normal distribution.")
	normMean          = flag.Float64("normal.mean", 0.00001, "The mean for the normal distribution.")
	oscillationPeriod = flag.Duration("oscillation-period", 10*time.Minute, "The duration of the rate oscillation period.")
)

var (
	//第一步:创建Summary类型和histogram类型的collector
    //此处为New*Vec形式的,定义的labelkey为service,其labelvalue可以有多个
	rpcDurations = prometheus.NewSummaryVec(
		prometheus.SummaryOpts{
			Name:       "rpc_durations_seconds",
			Help:       "RPC latency distributions.",
			Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
		},
		[]string{"service"},
	)
    //此处为New*形式的此处不带labelkey及labelvalue
	rpcDurationsHistogram = prometheus.NewHistogram(prometheus.HistogramOpts{
		Name:    "rpc_durations_histogram_seconds",
		Help:    "RPC latency distributions.",
		Buckets: prometheus.LinearBuckets(*normMean-5**normDomain, .5**normDomain, 20),
	})
)

func init() {
	//第二步:使用默认的注册器注册定义的指标
    prometheus.MustRegister(rpcDurations)
	prometheus.MustRegister(rpcDurationsHistogram)
	//注册build信息仅适用于支持module的go版本
	prometheus.MustRegister(prometheus.NewBuildInfoCollector())
}

func main() {
	flag.Parse()

	start := time.Now()

	oscillationFactor := func() float64 {
		return 2 + math.Sin(math.Sin(2*math.Pi*float64(time.Since(start))/float64(*oscillationPeriod)))
	}

	//第三步:创建数据
	go func() {
		for {
			v := rand.Float64() * *uniformDomain
			//带*Vec形式的collector需要WithLabelValues函数填写lablevalue
            rpcDurations.WithLabelValues("uniform").Observe(v)
			time.Sleep(time.Duration(100*oscillationFactor()) * time.Millisecond)
		}
	}()

	go func() {
		for {
			v := (rand.NormFloat64() * *normDomain) + *normMean
            //带*Vec形式的collector需要WithLabelValues函数填写lablevalue
			rpcDurations.WithLabelValues("normal").Observe(v)
            //只有New不带*Vec形式的collector直接设置值
			rpcDurationsHistogram.Observe(v)
			time.Sleep(time.Duration(75*oscillationFactor()) * time.Millisecond)
		}
	}()

	go func() {
		for {
			v := rand.ExpFloat64() / 1e6
            //带*Vec形式的collector需要WithLabelValues函数填写lablevalue
			rpcDurations.WithLabelValues("exponential").Observe(v)
			time.Sleep(time.Duration(50*oscillationFactor()) * time.Millisecond)
		}
	}()

	//第四步:暴露指标接口,第三步和第四步无先后顺序
	http.Handle("/metrics", promhttp.Handler())
	log.Fatal(http.ListenAndServe(*addr, nil))
```
定义的collector的形式为`New[type][Vec]`,type指定类型,Vec指定labelkey,指标名字在函数参数中定义.
**自定义的指标会如果不更新会一直export原来的值,而且通过vec定义的labelvalue也会一直在**

## 自定义exporter
如果生成的指标时每次采集时生成且labelvalue是动态的,则需要自定义exporter.

自定义exporter只需要满足Collector接口即可:
``` golang
type Collector interface {
	Describe(chan<- *Desc)
	Collect(chan<- Metric)
}
```
Describe:将此Collector收集的度量标准的所有可能描述符的超集发送到提供的通道，并在最后一个描述符发送后返回。发送的描述符满足了Desc文档中描述的一致性和唯一性要求。如果同一个收集器发送重复的描述符，则它是有效的。这些重复项被简单地忽略了。但是，两个不同的收集器不得发送重复的描述符。完全不发送描述符将收集器标记为“unchecked”，即在注册时不执行任何检查，并且收集器可以在其Collect方法中产生它认为合适的任何指标。此方法在收集器的整个生命周期中有意识地发送相同的描述符。**它可以同时调用，因此必须以并发安全的方式实现**。如果收集器在执行此方法时遇到错误，则它必须发送无效的描述符（使用NewInvalidDesc创建）以向registry发送错误信号。

Collect:收集指标时，Prometheus registry会调用Collect。实现通过提供的通道发送每个收集的度量标准，并在最后一个度量标准发送后返回。每个发送的度量标准的描述符是Describe返回的度量标准之一（除非收集器是unchecked的）。**共享相同描述符的返回指标的变量标签值必须不同。此方法可以同时调用，因此必须以并发安全的方式实现。**Block发生的代价是呈现所有已注册指标的总体性能。**理想情况下，Collector实现支持并发读者。**
1. 定义指标
``` golang
// 指标结构体
type Metrics struct {
    metrics map[string]*prometheus.Desc
    mutex   sync.Mutex
}

/**
 * 函数：newGlobalMetric
 * 功能：创建指标描述符
 */
func newGlobalMetric(namespace string, metricName string, docString string, labels []string) *prometheus.Desc {
    return prometheus.NewDesc(namespace+"_"+metricName, docString, labels, nil)
}


/**
 * 工厂方法：NewMetrics
 * 功能：初始化指标信息，即Metrics结构体
 */
func NewMetrics(namespace string) *Metrics {
    return &Metrics{
        metrics: map[string]*prometheus.Desc{
            "my_counter_metric": newGlobalMetric(namespace, "my_counter_metric", "The description of my_counter_metric", []string{"host"}),
            "my_gauge_metric": newGlobalMetric(namespace, "my_gauge_metric","The description of my_gauge_metric", []string{"host"}),
        },
    }
```

2. 注册指标
``` golang
metrics := collector.NewMetrics(*metricsNamespace)    // 创建指标结构体实例
registry := prometheus.NewRegistry()
registry.MustRegister(metrics)                        // 注册指标
```

3. 数据采集

数据采集需要实现collector的两个接口：
```golang
/**
 * 接口：Describe
 * 功能：传递结构体中的指标描述符到channel
 */
func (c *Metrics) Describe(ch chan<- *prometheus.Desc) {
    for _, m := range c.metrics {
        ch <- m
    }
}

/**
 * 接口：Collect
 * 功能：抓取最新的数据，传递给channel
 */
func (c *Metrics) Collect(ch chan<- prometheus.Metric) {
    c.mutex.Lock()  // 加锁
    defer c.mutex.Unlock()

    mockCounterMetricData, mockGaugeMetricData := c.GenerateMockData()
    for host, currentValue := range mockCounterMetricData {
        ch <-prometheus.MustNewConstMetric(c.metrics["my_counter_metric"], prometheus.CounterValue, float64(currentValue), host)
    }
    for host, currentValue := range mockGaugeMetricData {
        ch <-prometheus.MustNewConstMetric(c.metrics["my_gauge_metric"], prometheus.GaugeValue, float64(currentValue), host)
    }
}
```