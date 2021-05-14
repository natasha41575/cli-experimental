---
title: "replacements"
linkTitle: "replacements"
type: docs
weight: 18
description: >
    Substitute field(s) in N target(s) with a field from a source.
---

Replacements are used to copy fields from one source into any
number of specified targets. The `replacements` field in the kustomization file can support either a path to 
a replacement or an inline replacement. 

`kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

replacements:
  - path: replacement.yaml
  - source:
      kind: Deployment
      fieldPath: metadata.name
     targets:
       - select: 
           name: my-resource
```

### Syntax

The full schema of `replacements` is as follows:

```yaml
replacements:
- source:
    group: string
    version: string
    kind: string
    name: string
    namespace: string
    fieldPath: string
    options:
      delimiter: string
      index: int
      create: bool
  targets: 
  - select:
      group: string
      version: string
      kind: string
      name: string
      namespace: string
    reject:
    - group: string
      version: string
      kind: string
      name: string
      namespace: string
    fieldPaths: 
    - string
    options:
      delimiter: string
      index: int
      create: bool
```

### Field Descriptions

| Field       | Description |
| ----------- | ----------- |
| source [required] | The source of the value |
| target [required] | The N fields to write the value to |
| group [optional]  | The group of the referent |
| version [optional] | The version of the referent
|kind [optional] |The kind of the referent
|name [optional] |The name of the referent
|namespace [optional] |The namespace of the referent
|select [required] |Include objects that match this
|reject [optional] |Exclude objects that match this
|fieldPath [optional] |The structured path to the source value (defaults to metadata.name)
|fieldPaths [optional] |The structured path(s) to the target nodes (defaults to [metadata.name])
|options [optional] |Options used to refine interpretation of the field
|delimiter [optional] |Used to split/join the field
|index [optional] |Which position in the split to consider (defaults to 0)
|create [optional] |If target field is missing, add it (defaults to false)

### Source and Targets
The source selector must resolve to a single source. The targets can be ambiguous, and replacements will be applied to all matching targets.

#### Reject
Reject is a list of selectors that determine what objects should not be matched by this target. It being a list allows more flexibility. 

For example, if we wanted to reject all Deployments named my-deploy:

```yaml
reject:
- kind: Deployment
  name: my-deploy
```

This is distinct from the following:

```yaml
reject:
- kind: Deployment
- name: my-deploy
```

The first case would only reject resources that are both of kind Deployment and named my-deploy. The second case would reject all Deployments, and all resources named my-deploy.

Allowing a list here also allows rejection of more than one kind, name, etc. For example:

```yaml
reject:
- kind: Deployment
- kind: StatefulSet
```

####Delimiter and Index

These fields are intended for partial string replacement. If the delimiter is not specified, the index does nothing and the field is not split nor joined. If the delimiter is specified, the index will default to 0 unless otherwise specified. 

For a target, if the index is less than 0, the value will be prepended as a prefix. If it is beyond the length of the split list, the value will be appended as a suffix. For a source, an index out of bounds will throw an error.

If delimiter and/or index are specified when the source or target value is a mapping node or list (rather than a scalar value), the delimiter and index fields will do nothing. 

#### Field Path format
The fieldPath and fieldPaths fields support a format of a '.'-separated path to a value. For example, the default:

`metadata.name`

Strings are used for mapping nodes. For sequence nodes, we support two options:

1. Index by number:

`spec.template.spec.containers.1.image`

2. Index by key-value pair:

`spec.template.spec.containers.[name=nginx].image`


### Example

For example, suppose one specifies the name of a k8s Secret object in a container's 
environment variable as follows:

`job.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
        - image: myimage
          name: hello
          env:
            - name: SECRET_TOKEN
              value: SOME_SECRET_NAME
```

Suppose you have the following resources:

`resources.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - image: busybox
    name: myapp-container
  restartPolicy: OnFailure
---
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
```

To (1) replace the value of SOME_SECRET_NAME with the name of my-secret, and (2) to add
a restartPolicy copied from my-pod, you can do the following: 

`kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- resources.yaml
- job.yaml

replacements:
- path: my-replacement.yaml
- source:
    kind: Secret
    name: my-secret
  targets:
  - select:
      name: hello
      kind: Job
    fieldPaths:
    - spec.template.spec.containers.[name=hello].env.[name=SECRET_TOKEN].value
```

`my-replacement.yaml`
```yaml
source: 
  kind: Pod
  name: my-pod
  fieldPath: spec.restartPolicy
targets:
- select:
    name: hello
    kind: Job
  fieldPaths: 
  - spec.template.spec.restartPolicy
  options:
    create: true
```

The output of `kustomize build` will be:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
---
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    spec:
      containers:
      - env:
        - name: SECRET_TOKEN
          value: my-secret # this value is copied from my-secret
        image: myimage
        name: hello
      restartPolicy: OnFailure # this value is copied from my-pod
---
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - image: busybox
    name: myapp-container
  restartPolicy: OnFailure
```