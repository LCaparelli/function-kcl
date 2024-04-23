# Crossplane Composition Functions using KCL

[![Go Report Card](https://goreportcard.com/badge/github.com/crossplane-contrib/function-kcl)](https://goreportcard.com/report/github.com/crossplane-contrib/function-kcl)
[![GoDoc](https://godoc.org/github.com/crossplane-contrib/function-kcl?status.svg)](https://godoc.org/github.com/crossplane-contrib/function-kcl)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/crossplane-contrib/function-kcl/blob/main/LICENSE)

## Introduction

Crossplane KCL function allows developers to use [KCL](https://kcl-lang.io/) (a DSL) to write composite logic without the need for repeated packaging of crossplane functions, and we support package management and the [KRM KCL specification](https://github.com/kcl-lang/krm-kcl), which allows for OCI/Git source and the reuse of [KCL's module ecosystem](https://artifacthub.io/packages/search?org=kcl&sort=relevance&page=1).

Check out this [blog](https://blog.crossplane.io/function-kcl/) to learn more. Here's an simple example:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: example
spec:
  compositeTypeRef:
    apiVersion: example.crossplane.io/v1beta1
    kind: XR
  mode: Pipeline
  pipeline:
    - step: create-a-bucket
      functionRef:
        name: function-go-templating
      input:
        apiVersion: krm.kcl.dev/v1alpha1
        kind: KCLRun
        source: |
          # Read the XR
          oxr = option("params").oxr
          # Patch the XR with the status field
          dxr = {
              **oxr
              status.dummy = "cool-status"
          }
          # Construct a bucket
          bucket = {
              apiVersion = "s3.aws.upbound.io/v1beta1"
              kind = "Bucket"
              metadata.annotations: {
                  "krm.kcl.dev/composition-resource-name" = "bucket"
              }
              spec.forProvider.region = option("oxr").spec.region
          }
          # Return the bucket and patched XR
          items = [bucket, dxr]
    - step: automatically-detect-ready-composed-resources
      functionRef:
        name: function-auto-ready
```

## Install the KCL Function to Cluster

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: kcl-function
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-kcl:latest
EOF
```

## Using this Function

### Source Support

The function can load KCL codes from inline source, OCI source, Git source and FileSystem source.

+ OCI source example

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  source: oci://ghcr.io/kcl-lang/crossplane-xnetwork-kcl-function
```

+ Git source example

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  source: github.com/kcl-lang/modules/crossplane-xnetwork-kcl-function
```

+ FileSystem source example

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  target: Resources
  source: ./path/to/kcl/file.k
```

### Read the Function Requests through the `option` Function

+ Return an error using `assert {condition}, {error_message}`.
+ Read the `ObservedCompositeResource` from `option("params").oxr`.
+ Read the `ObservedComposedResources` from `option("params").ocds`.
+ Read the `DesiredCompositeResource` from `option("params").dxr`.
+ Read the `DesiredComposedResources` from `option("params").dcds`.
+ Read the function pipeline's context from `option("params").ctx`.

You can define your custom parameters in the `params` field and use `option("params").custom_key` to get the `custom_value`.

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  params:
     custom_key: custom_value
  source: oci://ghcr.io/kcl-lang/crossplane-xnetwork-kcl-function
```

### Expect Output

A KRM YAML list means that each document must have an `apiVersion`, `kind` through the `items` field or a single YAML output.

+ Using the `items` field

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  source: |
    items = [{
        apiVersion: "ec2.aws.upbound.io/v1beta1"
        kind: "Instance"
        metadata.name = "instance1"
        spec.forProvider.region: "us-east-2"
    }, {
        apiVersion: "ec2.aws.upbound.io/v1beta1"
        kind: "Instance"
        metadata.name = "instance2"
        spec.forProvider.region: "us-east-2"
    }]
```

+ Single YAML output

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: basic
spec:
  source: |
    {
        apiVersion: "ec2.aws.upbound.io/v1beta1"
        kind: "Instance"
        metadata.name = "instance"
        spec.forProvider.ami: "ami-0d9858aa3c6322f73"
        spec.forProvider.instanceType: "t2.micro"
        spec.forProvider.region: "us-east-2"
    }
```

### Composite Resource Connection Details

To return desired composite resource connection details, include a KCL dict that produces the special CompositeConnectionDetails resource like [the example](./examples/default/connection_details/composition.yaml):

```kcl
details = {
    apiVersion: "meta.krm.kcl.dev/v1alpha1"
    kind: "CompositeConnectionDetails"
    data: {
        "connection-secret-key": "connection-secret-value"
    }
}
```

> Note: The value of the connection secret value must be base64 encoded. This is already the case if you are referencing a key from a managed resource's connectionDetails field. However, if you want to include a connection secret value from somewhere else, you will need to use the `base64.encode` function:

```kcl
import base64

ocds = option("params").ocds
details = {
    apiVersion: "meta.krm.kcl.dev/v1alpha1"
    kind: "CompositeConnectionDetails"
    data: {
        "server-endpoint" = base64.encode(ocds["my-server"].Resource.status.atProvider.endpoint)
    }
}
```

To mark a desired composed resource as ready, use the `krm.kcl.dev/ready` annotation:

```kcl
user = {
    apiVersion: "iam.aws.upbound.io/v1beta1"
    kind: "User"
    metadata.name = "test-user"
    metadata.annotations: {
        "krm.kcl.dev/ready": "True"
    }
}
```

### Patching the XR status field

You can read the XR and patch it with the status field and returns the new patched XR in the `item` result like this

```
# Read the XR
oxr = option("params").oxr
# Patch the XR with the status field
dxr = {
    **oxr
    status.dummy = "cool-status"
}
items = [dxr] # Omit other resources
```

## Library

You can directly use [KCL standard libraries](https://kcl-lang.io/docs/reference/model/overview) such as `regex.match`, `math.log`.

## Tutorial

See [here](https://kcl-lang.io/docs/reference/lang/tour) to study more features such as conditions and loops in KCL.

## Examples

More examples can found [here](./examples/)

## Debugging the KCL Function in Cluster

Logs are emitted to the Function's pod logs. Look for the Function pod in `crossplane-system`.

### Levels

```shell
Info   # default
Debug  # run with --debug flag
```

## Developing

```shell
# Run code generation - see input/generate.go
$ go generate ./...

# Run tests - see fn_test.go
$ go test ./...

# Build the function's runtime image - see Dockerfile
$ docker build . --tag=kcllang/crossplane-kcl

# Build a function package - see package/crossplane.yaml
$ crossplane xpkg build -f package --embed-runtime-image=kcllang/crossplane-kcl

# Push a function package to the registry
$ crossplane --verbose xpkg push -f package/*.xpkg xpkg.upbound.io/crossplane-contrib/function-kcl:latest
```
