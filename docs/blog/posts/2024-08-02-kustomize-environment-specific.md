---
date: 2024-08-02
title: Kustomize - Using environment variables for configuration
authors:
  - sulochan
description: >
  Kustomize - Using environment variables for configuration
categories:
  - Kubernetes
  - Kustomize
---
# Kustomize: Using environment variables for configuration

[Kustomize](https://kustomize.io) is a widely used tool for Kuberenetes config management that provides a template free way to change your manifests during application deployment. It uses a kustomization.yaml file to define the actions taken during the build process. The file itself can be seen a collection of optional ordered processes: resources, generators, transformers, validators, configMapGenerator, patches and so on.

We wont dive too deep into these but look at a specific way of using the transformer property to use configuration values (as environment variables) from a file to drive your configuration.

<!-- more -->

Lets look at a typical layout of the kustomization files from the genstack repo:

```shell

$ ls genestack/base-kustomize/

.
..
argocd/base
bacups/etcd
barbican
...
```

and

```shell

$ ls genestack/base-kustomize/glance

..
aio
base
```

We are familiar with this style of providing variance. A base configuration and different varient to go with base configuration folders.

The problem that we face in this scenario if you would like to deploy different values for your base configuration let’s say depending on the region you are on or host you are on is difficult since it’s not templated.

One way to handle this is to use configuration file or environment variables to drive the values. This is possible to do since configMapGenerator takes in both envs and literal values. For that to work we need to adopt a slightly different way of doing things:

Take this deployment for example :

```yaml title="deployment.yaml"
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: flex-gateway
  namespace: nginx-gateway
  annotations: # This is the name of the ClusterIssuer created in the previous step
    cert-manager.io/cluster-issuer: flex-gateway-issuer
    acme.cert-manager.io/http01-edit-in-place: "true"
spec:
  gatewayClassName: $(CLASSNAME)
  listeners:
  - name: cluster-http
    port: 80
    protocol: HTTP
    hostname: $(HOST1)
    allowedRoutes:
      namespaces:
        from: All
  - name: cluster-tls
    port: 443
    protocol: HTTPS
    hostname: $(HOST2)
    allowedRoutes:
      namespaces:
        from: All
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: wildcard-cluster-tls-secret
```

Instead of hardcoding the gatewayClassName and hostname for the listners we have defined variables `$(CLASSNAME)` and `$(HOST1)` and so on.
We then modify our kustomization to include these variables in the config.

```yaml title="kustomization.yaml"

resources:
  - deployment.yaml

configurations:
  - env-var-transformer.yaml

configMapGenerator:
- name: environment-variables
  envs: [environment-properties.env]
  behavior: create

vars:
- name: HOST1
  objref:
    kind: ConfigMap
    name: environment-variables
    apiVersion: v1
  fieldref:
    fieldpath: data.HOST1
- name: HOST2
  objref:
    kind: ConfigMap
    name: environment-variables
    apiVersion: v1
  fieldref:
    fieldpath: data.HOST2
- name: CLASSNAME
  objref:
    kind: ConfigMap
    name: environment-variables
    apiVersion: v1
  fieldref:
    fieldpath: data.CLASSNAME
```

and the corresponding configuration file look like so

```yaml title="env-var-transformer.yaml"
varReference:
  - kind: Gateway
    path: spec/gatewayClassName
  - kind: Gateway
    path: spec/listeners/hostname
```

Finally, we want these valus to come from our config files, so we define these values in a file (keep it separate from the repo in a separate secure place and pull during deploy only)

```shell title="environment-properties.env"

HOST1=host1.cluster.local
HOST2=host2.cluster.local
CLASSNAME=nginx
```

Now, a `kubectl kustomize .` will result in the expected values being replaced with our config values. What this allows us to do is to have a set of base config be applied with different context specific variables.

!!! note "When the variable has no value assigned"

	One drawback of this is that if for some reason the value is undefined there is no default value replaced and the build will have empty or $var, so make sure you check your kustomize build to ensure smooth operation.
