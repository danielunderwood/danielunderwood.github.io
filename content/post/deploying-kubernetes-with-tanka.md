---
title: "Deploying Kubernetes with Grafana Tanka"
date: 2022-05-16T17:05:59-04:00
draft: false
tags: [kubernetes, tanka, jsonnet]
---

This weekend, I have been playing around with yet another Kubernetes deployment method: Grafana's [tanka](https://tanka.dev). I have seen Tanka in the past, but usually ignored it as a needless complication and going with [Helm](https://helm.sh) and [Flux](https://fluxcd.io/).

While Tanka's purpose is entirely different from Flux (I probably plan to try out Tanka + Argo), it does overlap with Helm quite a bit, allowing parametrization of manifests. It also allows merging and patching of manifests, much like [Kustomize](https://kustomize.io). But I believe Tanka takes a unique approach at solving the main headache of Kubernetes: the mess of redundant YAML used to describe resources.

## Figuring Things Out

It was easy to get started with Tanka's tutorial, but moving past their tutorial (such as attaching PVCs) proved to be a bit difficult. It turns out that that information is largely within the [k8s-libsonnet documentation](https://jsonnet-libs.github.io/k8s-libsonnet/), which is what actually allows the creation of Kubernetes resources within jsonnet; Tanka is just a nice wrapper to apply the jsonnet files to your cluster.

## Using Helm

While Tanka itself provides an alternative to Helm, many common projects still use Helm as the easiest way to get going. Tanka [has Helm support](https://tanka.dev/helm). It is marked as experimental, though I suspect it's mainly just calling `helm template` and importing the output in Jsonnet.

To get started with [ingress-nginx](https://github.com/kubernetes/ingress-nginx), you can run

```shell
jb install github.com/grafana/jsonnet-libs/tanka-util
tk tool charts init
tk tool charts add-repo ingress-nginx https://kubernetes.github.io/ingress-nginx
tk tool charts add ingress-nginx/ingress-nginx@4.1.0
```

which will initialize a `chartfile.yaml` describing repos and charts and vendor the ingress-nginx version 4.1.0 chart in the `charts/` directory. With that, you should be able to create an environment with a `main.jsonnet` containing

```jsonnet
local k = import "k.libsonnet";
local tanka = import "github.com/grafana/jsonnet-libs/tanka-util/main.libsonnet";
local helm = tanka.helm.new(std.thisFile);
local namespace = "ingress-nginx";
{
    ingressNginxNamespace: k.core.v1.namespace.new(namespace),
    // Best practice here is to move everything to a jsonnet library,
    // but this is good enough for an initial try
    ingressNginx: helm.template("ingress-nginx", "../../charts/ingress-nginx", {
        namespace: namespace,
        values: {}
    })
}
```

One may wonder what is going on with `tanka.helm.new(std.thisFile)`, but I'll just brush it away as magic (I suspect it's a clever way of replacing contents in the file with an import of `helm template` results).

## CRDs

Many cloud-native applications, such as cert-manager, create custom resources in addition to the kubernetes APIs. In many common cases, Jsonnet libraries have been creates for these resources (specifically, they're typically generated from API definitions), so using them with Tanka is just a matter of finding the correct dependencies. Many of these libraries can be found under the [jsonnet-libs](https://github.com/jsonnet-libs) GitHub organization.

## Secrets

At some point or another, we need secrets for any type of useful kubernetes deployment. Common techniques are SOPS and sealed-secrets, but I can't seem to find any information on using those with Tanka. sealed-secrets should be pretty straightforward, but it has been an annoyance when I've used it in the past, so I would prefer to manage secret encryption myself and just give k8s plain Secret resources. Of course I could leave them in plaintext in the repo, but you never know where a repo may end up.

My preferred method would likely be injecting some environment variables from a secure location through a script like with `sops exec-env`, but Jsonnet does not appear to provide a way to read environment variables. This basically leaves me with the option to use a file like `sops exec-file` (which can then be imported into Jsonnet), use encryption on the repository itself like `git-crypt`, or manage the secrets entirely out of band with something like `kubectl create secret`.

I have been using [gopass](https://www.gopass.pw/) for the management of some credentials already. At first glance, I was excited to find a [kubectl gopass plugin](https://github.com/gopasspw/kubectl-gopass) from the authors of gopass, but it looked like it didn't quite meet my needs since it requires that you store the secret resource itself in gopass while I would rather just the secret contents.

I ended up creating a really dumb shell script to use until I find a better solution:

```shell
#!/bin/bash

set +e

github_oauth() {
    client_id_b64="$(gopass show github-oauth | yq -r '.client_id' | base64)"
    client_secret_b64="$(gopass show github-oauth | yq -r '.client_secret' | base64)"
    kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
type: Opaque
metadata:
    name: github-oauth
    namespace: stats
data:
        client_id: "$client_id_b64"
        client_secret: "$client_secret_b64"
EOF
}

github_oauth
```

While I can't say it's the best thing I've ever seen, it should be pretty flexible until I figure out a better solution.

## Gotchas
- The structure of plain resources is usually checked, but you can easily supply wrong values to Helm and wander around for half an hour because `extraSecretsMounts` isn't `extraSecretMounts`

## Further Reading

If you want to know more about using Tanka or Jsonnet with kubernetes, then you may like some of the following:
- [Grafana: Introducing Tanka, Our Way of Deploying to Kubernetes](https://grafana.com/blog/2020/01/09/introducing-tanka-our-way-of-deploying-to-kubernetes/)
- [GitHub: grafana/tanka](https://github.com/grafana/tanka)
- [Official Tanka Tutorial](https://tanka.dev/tutorial/overview)
- [Argo: The State of Kubernetes Configuration Management: An Unsolved Problem](https://blog.argoproj.io/the-state-of-kubernetes-configuration-management-d8b06c1205)
- [databricks: Declarative Infrastructure with the Jsonnet Templating Language](https://databricks.com/blog/2017/06/26/declarative-infrastructure-jsonnet-templating-language.html)

Additionally, an example based on this blog post is available [on GitHub](https://github.com/danielunderwood/tanka-example).