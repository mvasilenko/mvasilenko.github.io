---
title: "k8s operator with kubebuilder - domain, group, kind, version"
date: 2023-06-07T15:05+01:00
draft: false
---

While trying to close my k8s knowledge gaps, I've followed [titorual](https://github.com/syntasso/instruqt-kubecon)
on how to create k8s operator with kubebuilder. After executing few magic commands, 
like

```
kubebuilder init --domain tutorial.kubebuilder.io --repo tutorial.kubebuilder.io/project
kubebuilder create api --group batch --version v1 --kind CronJob
```

I've keen to know more - what are domain, group, kind, why they needed at all and can we omit them?

There is some information on that topic in kubebuilder tutorial - [Groups and Versions and Kinds, oh my!](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#groups-and-versions-and-kinds-oh-my)
Also, there is a [question on stackoverflow](https://stackoverflow.com/questions/72337882/what-is-domain-in-kubebuilder-scaffolding)

It turns out that those abstractions serve namespacing purposes, and you cannot omit domain.
Below are few quotes from StackOverflow


A `group` along with a `version` and a `kind` uniquely identify a type of resource
that the Kubernetes API server knows about.

Functionally, the `domain/group` along with the version form the `apiVersion` field
of any resources you provision against

`group` is a required field on the `CustomResourceDefinition`

Also, `group` can be used to have single objects with the same `Kind: Bucket` but different
implementations

```
apiVersion: aws.amazon.com/v1
kind: Bucket
```

is different from

```
apiVersion: gcp.google.com/v1
kind: Bucket
```

Those CRDS are provisioned in fresh k3d k8s cluster:

```
$ kubectl get crds
NAME                                     CREATED AT
addons.k3s.cattle.io                     2023-06-13T22:08:22Z
helmcharts.helm.cattle.io                2023-06-13T22:08:23Z
helmchartconfigs.helm.cattle.io          2023-06-13T22:08:23Z
middlewaretcps.traefik.containo.us       2023-06-13T22:08:52Z
serverstransports.traefik.containo.us    2023-06-13T22:08:52Z
ingressrouteudps.traefik.containo.us     2023-06-13T22:08:52Z
ingressroutetcps.traefik.containo.us     2023-06-13T22:08:52Z
ingressroutes.traefik.containo.us        2023-06-13T22:08:52Z
tlsstores.traefik.containo.us            2023-06-13T22:08:52Z
tlsoptions.traefik.containo.us           2023-06-13T22:08:52Z
middlewares.traefik.containo.us          2023-06-13T22:08:52Z
traefikservices.traefik.containo.us      2023-06-13T22:08:52Z
```

Let's see what is inside last one - `traefikservices.traefik.containo.us`,
some lines were stripped for clarity

It means, k8s API server have CRD with `group: traefik.containo.us`

```
kubectl get crd traefikservices.traefik.containo.us -o yaml|kubectl neat

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us
spec:
  group: traefik.containo.us
  names:
    kind: TraefikService
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
      [skip]    
 ```

Quote from [k8s docs on CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#create-a-customresourcedefinition)

For example, if you save the following CustomResourceDefinition to `resourcedefinition.yaml`

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
 ```

and create it with

```
kubectl apply -f resourcedefinition.yaml
```

Then a new namespaced RESTful API endpoint is created at:

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

This endpoint URL can then be used to create and manage custom objects.
The kind of these objects will be CronTab from the spec of the CustomResourceDefinition
object you created above.

Hope, this helps with the understanding what are `domain`, `group`, `kind` and `version` for CRDs.
