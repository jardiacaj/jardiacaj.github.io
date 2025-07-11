---
layout: post
title:  "Passing computed values to Helm subcharts"
date:   2025-07-11
categories: tech
published: true
title_image: "/assets/helm-logo.png"
---

I recently found myself in a situation where I needed to pass computed values to a subchart in Helm. This does not seem to be possible directly, but there are ways to accomplish a similar effect.

{:refdef: style="text-align: center;"}
![Helm logo]({{ page.title_image }})
{: refdef}

## The problem

When using subcharts in Helm, values to be passed to them must be set in the parent chart's values file. There are situations where a subchart's configuration will depend on the parent chart's configuration. In some of these cases, it is useful to compute values for the subchart based on the parent chart's values.

The values files are static. Templating does not apply to values files. Thus, there's no direct way to compute values to be passed to a subchart.

## A workable solution: Passing template and values to subchart

**An important caveat: This solution requires the subchart to support it. It will not allow to pass computed values to arbitrary charts.**

It is not possible to pass computed values to a subchart, but a similar effect can be achieved by passing a template (as a string) and the required values to the subchart.

The subchart can then render the final configuration using the [tpl function](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-the-tpl-function).

Example subchart template:

{% raw %}
```yaml
kind: ConfigMap
metadata:
  name: example
data:
  foo: {{- tpl .Values.configMap.template $ | quote }}
```
{% endraw %}

Usage in parent chart `values.yaml` file:

{% raw %}
```yaml
subchart:
  configmap:
    template: |
      Value where we can render what we wish, let's try {{ .Release.Name }}.
```
{% endraw %}

Note that as the template is rendered in the context of the subchart, only values passed to the subchart will be usable. This may force providing configuration to the subchart that it does not actually need, but that the provided template will need to use for rendering.

## A bad idea: Overriding named templates

While exploring solutions to the original problem, I considered a different approach. Helm [named templates](https://helm.sh/docs/chart_template_guide/named_templates/) in a subchart can be overridden in a parent chart. This can be leveraged to pass a template, similarly to the previously proposed solution.

However, there are several reasons why this is not a great approach. The key argument that made me ultimately discard it is that template names are global. This means that when the same chart is used more than once as a dependency, it will not be possible to pass a different template to each of them, since the template to override has the same global name in each instance.

## Handling simple cases with YAML Anchors, and their caveats

In simple cases where the same configuration must be passed in several locations, DRY can be achieved with [YAML Anchors](https://helm.sh/docs/chart_template_guide/yaml_techniques/#yaml-anchors).

This is useful but one must be aware: the references will be resolved the first time the YAML is consumed. The references are then discarded. This means that later overriding of values (with `-f`, `--set` or in ArgoCD values) will not contemplate YAML anchors. This can lead to unexpected behaviour.

Example:

```yaml
localNetworkRange: &localNetworkRange 192.168.111.0/24
allowedNetworkRanges:
- *localNetworkRange
- 10.0.0.0/8
```

This will yield:

```yaml
localNetworkRange: 192.168.111.0/24
allowedNetworkRanges:
- 192.168.111.0/24
- 10.0.0.0/8
```

Overriding the referenced value with `--set localNetworkRange=192.168.234.0/24` will not lead to the alias using the new value.

```yaml
localNetworkRange: 192.168.234.0/24
allowedNetworkRanges:
- 192.168.111.0/24  # Not updated!
- 10.0.0.0/8
```

YAML anchors must be used carefully to avoid this potential confusion.

Additionally, YAML anchors only allow the same value to be used in multiple locations, but won't help when more complex logic is needed.

## My wish

I don't consider myself a Helm expert, and I suspect there are architectural reasons why the following is not easily possible. But if I could have a wish:

I wish values passed to subcharts could be templated just the same way Kubernetes manifests are.

Intuitively (and possibly naively), subcharts and Kubernetes manifests don't seem that different.

## Acknowledgements

I wrote this in part because neither traditional search nor AI tools provided me this answer without digging deeper. But I am definitely not the first one to figure this out.

Austin Dewey writes [here](https://austindewey.com/2021/02/22/using-the-helm-tpl-function-to-refer-values-in-values-files/) about the suggested solution. There are charts in the wild using this approach; [Grafana](https://github.com/grafana/helm-charts/tree/main/charts/grafana) is an example.

Big thanks to my dear colleague Madi, whom I asked for help when stuck with this issue and who gave me the key suggestion to consider the solution presented here.
