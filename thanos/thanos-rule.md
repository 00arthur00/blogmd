title: thanos规则
tags:
  - 监控
  - thanos
categories:
  - 监控
date: 2019-03-26 15:22:00
---
[原文](https://github.com/improbable-eng/thanos/blob/master/docs/components/rule.md)

# 规则

_**注意:** 规则组件是实验性的，因为它具有可能对大多数用例不利的概念权衡。 建议将规则部署到相关的Prometheus服务器。_

_规则组件尤其不能被用于在配置管理级别规避正确解决规则部署。_

规则组件针对其群集中的随机查询节点计算Prometheus记录和警报规则。规则结果以Prometheus 2.0存储格式写回磁盘。规则节点同时作为源存储节点参与集群本身，并将其生成的TSDB块上载到对象存储库。

可以标记每个规则节点的数据以满足集群标记方案。高可用性对可以并行运行，并且应该通过指定的副本标签进行区分，就像常规的Prometheus服务器一样。

```
$ thanos rule \
    --data-dir          "/path/to/data" \
    --eval-interval     "30s" \
    --rule-file         "/path/to/rules/*.rules.yaml" \
    --alert.query-url   "http://0.0.0.0:9090" \
    --alertmanagers.url "alert.thanos.io" \
    --cluster.peers     "thanos-cluster.example.org" \
    --objstore.config-file "bucket.yml"
```

`bucket.yml`内容:

```yaml
type: GCS
config:
  bucket: example-bucket
```
由于规则节点将查询处理外包给查询节点，因此它们通常应该承受很少的负载。如有必要，可以通过在HA对之间拆分规则集来应用功能分片。

根据在查询节点上配置的副本标签，使用经过去重数据处理规则。


## 部署

## Flags

[embedmd]:# (flags/rule.txt $)
```$
usage: thanos rule [<flags>]

ruler evaluating Prometheus rules against given Query nodes, exposing Store API
and storing old blocks in bucket

Flags:
  -h, --help                     Show context-sensitive help (also try
                                 --help-long and --help-man).
      --version                  Show application version.
      --log.level=info           Log filtering level.
      --log.format=logfmt        Log format to use.
      --gcloudtrace.project=GCLOUDTRACE.PROJECT
                                 GCP project to send Google Cloud Trace tracings
                                 to. If empty, tracing will be disabled.
      --gcloudtrace.sample-factor=1
                                 How often we send traces (1/<sample-factor>).
                                 If 0 no trace will be sent periodically, unless
                                 forced by baggage item. See
                                 `pkg/tracing/tracing.go` for details.
      --http-address="0.0.0.0:10902"
                                 Listen host:port for HTTP endpoints.
      --grpc-address="0.0.0.0:10901"
                                 Listen ip:port address for gRPC endpoints
                                 (StoreAPI). Make sure this address is routable
                                 from other components if you use gossip,
                                 'grpc-advertise-address' is empty and you
                                 require cross-node connection.
      --grpc-server-tls-cert=""  TLS Certificate for gRPC server, leave blank to
                                 disable TLS
      --grpc-server-tls-key=""   TLS Key for the gRPC server, leave blank to
                                 disable TLS
      --grpc-server-tls-client-ca=""
                                 TLS CA to verify clients against. If no client
                                 CA is specified, there is no client
                                 verification on server side. (tls.NoClientCert)
      --grpc-advertise-address=GRPC-ADVERTISE-ADDRESS
                                 Explicit (external) host:port address to
                                 advertise for gRPC StoreAPI in gossip cluster.
                                 If empty, 'grpc-address' will be used.
      --cluster.address="0.0.0.0:10900"
                                 Listen ip:port address for gossip cluster.
      --cluster.advertise-address=CLUSTER.ADVERTISE-ADDRESS
                                 Explicit (external) ip:port address to
                                 advertise for gossip in gossip cluster. Used
                                 internally for membership only.
      --cluster.peers=CLUSTER.PEERS ...
                                 Initial peers to join the cluster. It can be
                                 either <ip:port>, or <domain:port>. A lookup
                                 resolution is done only at the startup.
      --cluster.gossip-interval=<gossip interval>
                                 Interval between sending gossip messages. By
                                 lowering this value (more frequent) gossip
                                 messages are propagated across the cluster more
                                 quickly at the expense of increased bandwidth.
                                 Default is used from a specified network-type.
      --cluster.pushpull-interval=<push-pull interval>
                                 Interval for gossip state syncs. Setting this
                                 interval lower (more frequent) will increase
                                 convergence speeds across larger clusters at
                                 the expense of increased bandwidth usage.
                                 Default is used from a specified network-type.
      --cluster.refresh-interval=1m
                                 Interval for membership to refresh
                                 cluster.peers state, 0 disables refresh.
      --cluster.secret-key=CLUSTER.SECRET-KEY
                                 Initial secret key to encrypt cluster gossip.
                                 Can be one of AES-128, AES-192, or AES-256 in
                                 hexadecimal format.
      --cluster.network-type=lan
                                 Network type with predefined peers
                                 configurations. Sets of configurations
                                 accounting the latency differences between
                                 network types: local, lan, wan.
      --cluster.disable          If true gossip will be disabled and no cluster
                                 related server will be started.
      --label=<name>="<value>" ...
                                 Labels to be applied to all generated metrics
                                 (repeated). Similar to external labels for
                                 Prometheus, used to identify ruler and its
                                 blocks as unique source.
      --data-dir="data/"         data directory
      --rule-file=rules/ ...     Rule files that should be used by rule manager.
                                 Can be in glob format (repeated).
      --eval-interval=30s        The default evaluation interval to use.
      --tsdb.block-duration=2h   Block duration for TSDB block.
      --tsdb.retention=48h       Block retention time on local disk.
      --alertmanagers.url=ALERTMANAGERS.URL ...
                                 Alertmanager replica URLs to push firing
                                 alerts. Ruler claims success if push to at
                                 least one alertmanager from discovered
                                 succeeds. The scheme may be prefixed with
                                 'dns+' or 'dnssrv+' to detect Alertmanager IPs
                                 through respective DNS lookups. The port
                                 defaults to 9093 or the SRV record's value. The
                                 URL path is used as a prefix for the regular
                                 Alertmanager API path.
      --alertmanagers.send-timeout=10s
                                 Timeout for sending alerts to alertmanager
      --alert.query-url=ALERT.QUERY-URL
                                 The external Thanos Query URL that would be set
                                 in all alerts 'Source' field
      --alert.label-drop=ALERT.LABEL-DROP ...
                                 Labels by name to drop before sending to
                                 alertmanager. This allows alert to be
                                 deduplicated on replica label (repeated).
                                 Similar Prometheus alert relabelling
      --web.route-prefix=""      Prefix for API and UI endpoints. This allows
                                 thanos UI to be served on a sub-path. This
                                 option is analogous to --web.route-prefix of
                                 Promethus.
      --web.external-prefix=""   Static prefix for all HTML links and redirect
                                 URLs in the UI query web interface. Actual
                                 endpoints are still served on / or the
                                 web.route-prefix. This allows thanos UI to be
                                 served behind a reverse proxy that strips a URL
                                 sub-path.
      --web.prefix-header=""     Name of HTTP request header used for dynamic
                                 prefixing of UI links and redirects. This
                                 option is ignored if web.external-prefix
                                 argument is set. Security risk: enable this
                                 option only if a reverse proxy in front of
                                 thanos is resetting the header. The
                                 --web.prefix-header=X-Forwarded-Prefix option
                                 can be useful, for example, if Thanos UI is
                                 served via Traefik reverse proxy with
                                 PathPrefixStrip option enabled, which sends the
                                 stripped prefix value in X-Forwarded-Prefix
                                 header. This allows thanos UI to be served on a
                                 sub-path.
      --objstore.config-file=<bucket.config-yaml-path>
                                 Path to YAML file that contains object store
                                 configuration.
      --objstore.config=<bucket.config-yaml>
                                 Alternative to 'objstore.config-file' flag.
                                 Object store configuration in YAML.
      --query=<query> ...        Addresses of statically configured query API
                                 servers (repeatable). The scheme may be
                                 prefixed with 'dns+' or 'dnssrv+' to detect
                                 query API servers through respective DNS
                                 lookups.
      --query.sd-files=<path> ...
                                 Path to file that contain addresses of query
                                 peers. The path can be a glob pattern
                                 (repeatable).
      --query.sd-interval=5m     Refresh interval to re-read file SD files.
                                 (used as a fallback)
      --query.sd-dns-interval=30s
                                 Interval between DNS resolutions.

```