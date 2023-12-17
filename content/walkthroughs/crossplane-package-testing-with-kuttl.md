---
title: Testing Crossplane Packages with Kuttl
date: 2022-01-07
description: 'Test and validate crossplane package with https://kuttl.dev'
image: images/crossplane/crossplane-package-testing-with-kuttl/test_loop_standard.png
aliases:
- /crossplane/crossplane-package-testing-with-kuttl/
---

## Crossplane Package Test Loop

The test loop for Crossplane Packages is always going to be an end-2-end test in
a working kubernetes cluster. In order to validate that a package behaves as
desired, we must:

1. Access a Kubernetes Cluster
1. Install Crossplane and required providers on that cluster
1. Configure our Composite Resources (XRs) on that cluster
1. Submit a Composite Resource or Claim
1. Validate that we got the desired Resources in response
1. Repeat the last 3 steps for each Resource

### Kuttl

[kuttl](https://kuttl.dev) (The KUbernetes Test TooL) is a toolkit for writing
tests against Kubernetes. Using kuttl, engineers can apply manifests, helm
charts, or run kubectl commands against a kubernetes cluster, then compare the
cluster state against a yaml file to validate the result.

This is exactly what we need when testing our configuration packages. Unlike
custom operators, crossplane compositions will always output declarative yaml
objects. So we can validate the output of our Composite Resources with simple
yaml comparisons.

We've been able to satisfy all of the phases of the test loop above with kuttl.
In this post I will show you how we did it.

## Let's get Started!

You can find a complete example of this tutorial in the [A Tour of Crossplane](https://github.com/AaronME/ATourOfCrossplane/tree/main/crossplane-package-testing-with-kuttl) repo.

### Prerequisites

We're going to assume you have the following installed:
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [helm](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

#### Installing kuttl

I am running Mac OS X, so I installed kuttl using homebrew:

```bash
brew tap kudobuilder/tap
brew install kuttl-cli
```

You can find installation instructions for other platforms on
[kuttl's install page](https://kuttl.dev/docs/#install-kuttl-cli).

### Step One: Launch Kind

We're going to create a kuttl __TestSuite__ and use it to start a kind cluster.
Create the following file in the root of your workspace.

**kuttl-test.yaml**:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
startKIND: true
kindContext: kuttl-test
```

You have now completed Step One. That was easy!

At the start of each test run kuttl will create a kind cluster with the name
defined in **kindContext**.

If you don't care about the name of the cluster, you can omit this. Kuttl will
just create a cluster with the default name "kind".

### Step Two: Install Crossplane

We need Crossplane running on the cluster to provision Composite Resources and
render the Compositions.  Kuttl supports a __commands__ argument which accepts a
list of commands to run. Each command must start with valid binary. If you want
to use shell built-ins or scripting functionality you can wrap those in a script
called by __command__ or you can use the __script__ command type.

__commands__ is supported in both TestSuites and TestSteps.

The __commands__ argument is really powerful. We can use it to run any command or
script required to setup our tests. Because we will need Crossplane to keep
running for all tests, we're going to set it up in the TestSuite, rather than
any individual TestStep.

Let's add __commands__ to install Crossplane to our Test Suite.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace
startKIND: true
kindContext: kuttl-test
```

We use **universal-crossplane** at Upbound, but you can use vanilla
**crossplane**. They will both work for this demo.

### Step Three: Add Composite Resources to the Control Plane

We're going to source the Composite Resource configuration from the
[platform-ref-aws](https://github.com/upbound/platform-ref-aws) package on
github. When testing our local providers, we cannot depend on crossplane's
native dependency resolution. We must install our package dependencies
ourselves.

For this demo we need
[provider-aws](https://github.com/crossplane/provider-aws). The helm chart for
crossplane supports passing in a list of provider packages during install. We'll
do this using a values file.

Using a values file will make it easier to keep the test dependencies up to date
with the actual dependencies, declared in a __crossplane.yaml__. We'll keep the
values file under a new folder called __tests__, which will be the home for our
test cases as well.

__tests/uxp-values.yaml__:
```yaml
provider:
  packages:
    - crossplane/provider-aws:v0.19.0
```

Now update the command in your TestSuite.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
startKIND: true
kindContext: kuttl-test
```

We're going to want to make sure we delay any testing until the provider is
healthy. So let's add a __kubectl wait__ command.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
  # Wait for provider-aws to become healthy.
  - command: kubectl wait --for condition=healthy  --timeout=300s provider/crossplane-provider-aws
startKIND: true
kindContext: kuttl-test
```

#### Creating a TestStep

Now that we're working with specific resources, we're ready to define Test
Cases. First, we'll create a folder for our composition tests and a folder for
our test case (we'll be using the __CompositeNetwork__ Resource from
__platform-ref-aws__).

```bash
mkdir -p tests/compositions/compositenetwork
```

We'll add __tests/compositions/__ to the list of __testDirs__ in our TestSuite.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
  # Wait for provider-aws to become healthy.
  - command: kubectl wait --for condition=healthy  --timeout=300s provider/crossplane-provider-aws
testDirs:
  - tests/compositions/
startKIND: true
kindContext: kuttl-test
```

#### Composite Resource Order of Operations

This is not a tutorial on writing compositions, or a technical walkthrough of
how compositions work. But I want to clearly establish the order of operations
required to get a new XR installed on a cluster.

A Crossplane Composite Resource (XR) defines a new Type in the Kubernetes API.
Composite Resources are defined by two other types: Composite Resource 
Definitions (XRD), and Compositions.

To install a new XR, you must apply both an XRD _and_ a valid Composition that
satisfies that XRD.

When an XRD has successfully added the new type to Kubernetes, it will have the
status "established=true".

You can only submit an XR or a Claim for one if its XRD is "established."

So the first step our test case will install the XRD and Composite.

__tests/compositions/compositenetwork/00-install.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestStep
commands:
  # Install the XRD
  - command: kubectl apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/definition.yaml
  # Install the Composition
  - command: kubectl apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/composition.yaml
```

We're using the same __commands__ argument that we saw in the TestSuite. If we 
were working with a local package these commands would reference files in our
__package/__ folder. But for the sake of this demo, we are applying them
directly from github.

(If you want to play around with local files, feel free to copy all of those down
to your workspace and apply them from a local __package/__ folder.)

#### A brief detour: The No Money Down ProviderConfig

If you ran the test at this point, you most likely got an error on the XR. This
is because Crossplane cannot find the ProviderConfig required by one of the
resources in our Composition. Without that ProviderConfig, Crossplane will not
render composition.

We don't need to actually create any AWS infrastructure to run these tests. We
only want to validate that our Composition renders properly. So we'll create a
"no money down" ProviderConfig for AWS in an __init__ folder under __tests__.

__tests/init/providerConfig.yaml__:
```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-creds
  namespace: upbound-system
type: Opaque
stringData:
  key: nocreds
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: upbound-system
      name: aws-creds
      key: key
```

And we'll make sure every test case can use this by adding it ot the TestSuite.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
  # Wait for provider-aws to become healthy.
  - command: kubectl wait --for condition=healthy  --timeout=300s provider/crossplane-provider-aws
  # Install ProviderConfig for test.
  - command: kubectl apply -f tests/init/
testDirs:
  - tests/compositions/
startKIND: true
kindContext: kuttl-test
```

### Step Four: Create the CompositeResource

We are now going to submit a CompositeResource for Crossplane to render. Note
that we are making this a new Step by incrementing the numerical prefix. Kuttl
will run all of the __00-\*__ test steps before proceeding to __01-\*__.)

We will add a __kubectl wait__ command to make sure the XRD is established
before applying our resource.

__tests/compositions/compositenetwork/01-network.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestStep
commands:
  # Wait for XRD to become "established"
  - command: kubectl wait --for condition=established --timeout=20s xrd/compositenetworks.aws.platformref.crossplane.io
  # Create the XR/Claim
  - command: kubectly apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/examples/network.yaml
```

### Step Five: Validate Managed Resources

At this point, Crossplane should be rendering our Network Resource. We
will now validate that using an __*-assert.yaml__ file. Warning,
__CompositeNetwork__ creates 8 managed resources. We've limited this test to
checking only for the existence of two subnets and some tags on the VPC.

__tests/compositions/compositenetwork/01-assert.yaml__:
```yaml
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  labels:
    access: private
    crossplane.io/claim-name: network
    networks.aws.platformref.crossplane.io/network-id: platform-ref-aws-network
    zone: us-west-2a
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  labels:
    access: public
    crossplane.io/claim-name: network
    networks.aws.platformref.crossplane.io/network-id: platform-ref-aws-network
    zone: us-west-2a
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: VPC
metadata:
  labels:
    crossplane.io/claim-name: network
    networks.aws.platformref.crossplane.io/network-id: platform-ref-aws-network
spec:
  deletionPolicy: Delete
  forProvider:
    tags:
    - key: crossplane-kind
      value: vpc.ec2.aws.crossplane.io
    - key: crossplane-name
    - key: crossplane-providerconfig
```

#### Parallelism

Kuttl will run your Test Cases in Parallel. This can cause problems if you have
multiple resources that depend on one another, or on a common resource. (For
example, both PostgresDB and Cluster rely on Network in platform-ref-aws). This
could create conditions where cleanup of one test case destroys a dependency for
another.

Keep this in mind when designing your Test Suite. You may wish to apply _all_ of
the Composite Resources in your __package/__ folder at the beginning of the Test
Suite to ensure all dependencies are in place.

#### Running the test

Now that everything is setup, run the tests from the root of your workspace.

```bash
kubectl kuttle test

=== RUN   kuttl
    harness.go:457: starting setup
    harness.go:245: running tests with KIND.
    harness.go:156: Starting KIND cluster
    kind.go:67: Adding Containers to KIND...
    harness.go:285: Successful connection to cluster at: https://127.0.0.1:50478
    logger.go:42: 20:44:36 |  | running command: [helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml]
    ...
    logger.go:42: 20:46:21 |  | running command: [kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws]
    logger.go:42: 20:47:30 |  | provider.pkg.crossplane.io/crossplane-provider-aws condition met
    logger.go:42: 20:47:30 |  | running command: [kubectl apply -f tests/init/]
    logger.go:42: 20:47:39 |  | secret/aws-creds created
    logger.go:42: 20:48:00 |  | providerconfig.aws.crossplane.io/default created
    harness.go:353: running tests
    harness.go:74: going to run test suite with timeout of 30 seconds for each step
    harness.go:365: testsuite: tests/compositions/ has 1 tests
=== RUN   kuttl/harness
=== RUN   kuttl/harness/compositenetwork
=== PAUSE kuttl/harness/compositenetwork
=== CONT  kuttl/harness/compositenetwork
    logger.go:42: 20:48:00 | compositenetwork | Creating namespace: kuttl-test-enormous-albacore
    logger.go:42: 20:48:00 | compositenetwork/0-install | starting test step 0-install
    logger.go:42: 20:48:00 | compositenetwork/0-install | running command: [kubectl apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/definition.yaml]
    logger.go:42: 20:48:02 | compositenetwork/0-install | compositeresourcedefinition.apiextensions.crossplane.io/compositenetworks.aws.platformref.crossplane.io created
    logger.go:42: 20:48:02 | compositenetwork/0-install | running command: [kubectl apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/composition.yaml]
    logger.go:42: 20:48:04 | compositenetwork/0-install | composition.apiextensions.crossplane.io/compositenetworks.aws.platformref.crossplane.io created
    logger.go:42: 20:48:05 | compositenetwork/0-install | test step completed 0-install
    logger.go:42: 20:48:05 | compositenetwork/1-network | starting test step 1-network
    logger.go:42: 20:48:05 | compositenetwork/1-network | running command: [kubectl wait --for condition=established --timeout=20s xrd/compositenetworks.aws.platformref.crossplane.io]
    logger.go:42: 20:48:06 | compositenetwork/1-network | compositeresourcedefinition.apiextensions.crossplane.io/compositenetworks.aws.platformref.crossplane.io condition met
    logger.go:42: 20:48:06 | compositenetwork/1-network | running command: [kubectl apply -f https://raw.githubusercontent.com/upbound/platform-ref-aws/main/examples/network.yaml]
    logger.go:42: 20:48:12 | compositenetwork/1-network | network.aws.platformref.crossplane.io/network created
    logger.go:42: 20:48:23 | compositenetwork/1-network | test step completed 1-network
    logger.go:42: 20:48:23 | compositenetwork | compositenetwork events from ns kuttl-test-enormous-albacore:
    logger.go:42: 20:48:23 | compositenetwork | Deleting namespace: kuttl-test-enormous-albacore
=== CONT  kuttl
    harness.go:399: run tests finished
    harness.go:508: cleaning up
    harness.go:569: tearing down kind cluster
--- PASS: kuttl (362.88s)
    --- PASS: kuttl/harness (0.00s)
        --- PASS: kuttl/harness/compositenetwork (23.70s)
PASS
```

## The Finished Product

In the end, our folder structure for a configuration package including kuttl
tests will look like the following:

```bash
.
├── examples # CompositeResource examples (not demonstrated)
├── kuttl-test.yaml # TestSuite configuration
├── package # XRDs and Composites (not demonstrated)
│   └── crossplane.yaml # Configuration Package metadata file
└── tests # TestSteps
    ├── compositions
    │   └── compositenetwork
    │       ├── 00-install.yaml
    │       ├── 01-assert.yaml
    │       └── 01-network.yaml
    ├── init # Manifests to initialize test environment
    │   └── providerConfig.yaml
    └── uxp-values.yaml # Values for Universal-Crossplane
```

And this is the complete __kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
commands:
  # Install Univerasal Crossplane (uxp).
  - command: helm install universal-crossplane https://charts.upbound.io/stable/universal-crossplane-1.5.1-up.1.tgz --namespace upbound-system --wait --create-namespace --values tests/uxp-values.yaml
  # Wait for provider-aws to become healthy.
  - command: kubectl wait --for condition=healthy  --timeout=300s provider/crossplane-provider-aws
  # Install ProviderConfig for test.
  - command: kubectl apply -f tests/init/
testDirs:
  # Test Cases
  - tests/compositions/
startKIND: true
kindContext: kuttl-test
```

## Useful Links

- [KUTTL: Level up your Kubernetes E2E Testing - Ken Sipe (D2iQ)](https://www.youtube.com/watch?v=K9i-vleJggo)
- [kuttl configuration reference](https://kuttl.dev/docs/testing/reference.html) 
- [Crossplane Terminology](https://crossplane.io/docs/v1.6/concepts/terminology.html#crossplane-terms)



