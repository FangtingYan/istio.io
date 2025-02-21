---
title: 自定义 Istio 指标
description: 此任务向您展示如何自定义 Istio 指标。
weight: 25
keywords: [telemetry,metrics,customize]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

此任务向您展示如何自定义 Istio 生成的指标。

Istio 生成各种 dashboard 使用的遥测数据，以帮助您可视化您的网格。
例如，支持 Istio 的 dashboard 包括：

* [Grafana](/zh/docs/tasks/observability/metrics/using-istio-dashboard/)
* [Kiali](/zh/docs/tasks/observability/kiali/)
* [Prometheus](/zh/docs/tasks/observability/metrics/querying-metrics/)

默认情况下，Istio 定义并生成一组标准指标（例如： `requests_total`），但您也可以自定义它们并创建新指标。

## 自定义统计配置{#custom-statistics-configuration}

Istio 使用 Envoy 代理生成指标并在
[manifests/charts/istio-control/istio-discovery/templates/telemetryv2_{{< istio_version >}}.yaml]({{<github_blob>}}/manifests/charts/istio-control/istio-discovery/templates/telemetryv2_{{< istio_version >}}.yaml) 文件的 `EnvoyFilter` 中提供配置。

配置自定义统计信息涉及 `EnvoyFilter` 的两个部分： `definitions` 和 `metrics`。
在 `definitions` 部分支持用名字、期望值表达式和指标类型（`counter` 、 `gauge` 和 `histogram` ）创建新的指标。
在 `metrics` 的部分以表达式的形式提供指标维度的值，并允许您删除或覆盖现有的指标维度。
您可以调整标准指标定义，
利用 `tags_to_remove` 或重新定义维度。
这些配置设置也用 istioctl 安装选项公开，
允许您为网关和边车以及入站或出站方向自定义不同的指标。

有关详细信息，请参阅[统计配置参考](/zh/docs/reference/config/proxy_extensions/stats/)。

## 开始之前{#before-you-begin}

在集群中[安装 Istio](/zh/docs/setup/)并部署应用程序。
或者，您可以设置自定义统计作为 Istio 安装的一部分。

[Bookinfo示例](/zh/docs/examples/bookinfo/)应用程序在整个任务中用作示例应用程序。

## 启用自定义指标{#enable-custom-metrics}

1. 默认遥测 v2 `EnvoyFilter`配置等效于以下安装选项：

    {{< text yaml >}}
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
      values:
        telemetry:
          v2:
            prometheus:
              configOverride:
                inboundSidecar:
                  disable_host_header_fallback: false
                outboundSidecar:
                  disable_host_header_fallback: false
                gateway:
                  disable_host_header_fallback: true
    {{< /text >}}

    要自定义遥测 v2 指标，例如，添加`request_host`
    和 `destination_port` 维度到两者发出的 `requests_total` 指标
    入站和出站方向的网关和边车，请更改安装选项，如下所示：

    {{< tip >}}
    您只需为要自定义的设置指定配置。
    例如，仅自定义边车入站 `requests_count` 指标，您可以省略
    配置中的 `outboundSidecar` 和 `gateway` 部分。
    未指定设置将保留默认配置，相当于上面显示的显式设置。
    {{< /tip >}}

    {{< text yaml >}}
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
      values:
        telemetry:
          v2:
            prometheus:
              configOverride:
                inboundSidecar:
                  metrics:
                    - name: requests_total
                      dimensions:
                        destination_port: string(destination.port)
                        request_host: request.host
                outboundSidecar:
                  metrics:
                    - name: requests_total
                      dimensions:
                        destination_port: string(destination.port)
                        request_host: request.host
                gateway:
                  metrics:
                    - name: requests_total
                      dimensions:
                        destination_port: string(destination.port)
                        request_host: request.host
    {{< /text >}}

1. 使用以下命令将以下注释应用到所有注入的 Pod 以及要提取到 Prometheus [时间序列](https://en.wikipedia.org/wiki/Time_series)中的维度列表：

    {{< tip >}}
    仅当您的维度不在
    [DefaultStatTags 列表]({{<github_blob>}}/pkg/bootstrap/config.go) 才需要此步骤
    {{< /tip >}}

    {{< text yaml >}}
    apiVersion: apps/v1
    kind: Deployment
    spec:
      template: # pod template
        metadata:
          annotations:
            sidecar.istio.io/extraStatTags: destination_port,request_host
    {{< /text >}}

    要在网格范围内启用额外的标签，您可以添加 `extraStatTags` 到网格配置中：

    {{< text yaml >}}
    meshConfig:
      defaultConfig:
        extraStatTags:
         - destination_port
         - request_host
    {{< /text >}}

## 验证结果{#verify-the-results}

将流量发送到网格。对于 Bookinfo 示例，请 `http://$GATEWAY_URL/productpage` 在您的网络浏览器中访问或发出以下命令：

{{< text bash >}}
$ curl "http://$GATEWAY_URL/productpage"
{{< /text >}}

{{< tip >}}
`$GATEWAY_URL` 是 Bookinfo 示例中设置的值。
{{< /tip >}}

使用以下命令验证 Istio 是否为您的新维度或修改后的维度生成数据：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -- curl -sS 'localhost:15000/stats/prometheus' | grep istio_requests_total
{{< /text >}}

例如，在输出中，找到指标 `istio_requests_total` 并验证它是否包含您的新维度。

{{< tip >}}
代理开始应用配置可能需要很短的时间。如果未收到该指标，您可以在稍等片刻后重试发送请求，然后再次查找该指标。
{{< /tip >}}

## 对值使用表达式{#use-expressions-for-values}

指标配置中的值是常用表达式，这意味着您
JSON 中的字符必须双引号（例如："'string value'"）。
与 Mixer 表达式语言不同，不支持 pipe (`|`) 运算符，但您
可以使用 `has` 或 `in` 操作符来模拟它，例如：

{{< text plain >}}
has(request.host) ? request.host : "unknown"
{{< /text >}}

有关详细信息，请参阅[通用表达式语言](https://opensource.google/projects/cel)。

Istio 公开了所有标准 [Envoy 属性](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes)。
对等元数据可用作出站属性 `upstream_peer` 和入站属性 `downstream_peer` ，具有以下字段：

|字段  | 类型  | 值 |
|---|---|---|
| `name` | `string` | Pod 的名子。 |
| `namespace` | `string` | Pod 运行的名子空间。 |
| `labels` | `map` | 工作负载标签。 |
| `owner` | `string` | 工作负载所有者。|
| `workload_name` | `string` | 工作负载名称。 |
| `platform_metadata` | `map` | 带有前缀键的平台元数据。 |
| `istio_version` | `string` | 代理的版本标识符。 |
| `mesh_id` | `string` | 网格的唯一标识符。 |
| `app_containers` | `list<string>` | 应用程序容器的短名称列表。 |
| `cluster_id` | `string` | 此工作负载所属的集群的标识符。 |

例如，要在出站配置中使用的对等 `app` 标签的表达式是
`upstream_peer.labels['app'].value`。

有关详细信息[配置参考](/zh/docs/reference/config/proxy_extensions/stats/)。

