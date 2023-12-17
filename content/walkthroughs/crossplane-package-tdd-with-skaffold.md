---
title: TDD For Crossplane Packages with Skaffold
date: 2022-01-14
description: 'Create a Test-Driven Development Environment for Crossplane Packages with https://skaffold.dev'
image: images/crossplane/crossplane-packages-tdd-with-skaffold/tdd-with-skaffold.png
aliases:
- /crossplane/crossplane-package-tdd-with-skaffold/
---

## Crossplane Package Development

Crossplane Packages, and specifically the Composition Engine, allow Platform
Teams to publish Composite Resources for use by Development Teams or Platform
Operators. The end result of a Crossplane Composition is always going to be a
set of valid YAML documents which define Crossplane Managed Resources.

Because the output of our compositions is this declarative set of YAML
documents, we have the ability to [write
tests](https://aaroneaton.com/crossplane/crossplane-package-testing-with-kuttl/)
for our configurations. And once we can write tests, we can do test-driven
development.

### Write the ~~Infrastructure~~ Test First

The Test-Driven workflow for creating new Composite Resources looks like this:

1. Create the POC Architecture as Managed Resources, deployed via Crossplane.
1. Translate the Managed Resource manifests into tests (using a framework such
   as [kuttl](https://kuttl.dev/)).
1. Create the XRD and Compositions for our new Composite Resource.
1. Create an Example manifest for the new Composite Resource.
1. Iteratively build out the new Composite Resource, using our tests to validate
   as we write.
1. Once the Composite Resource is finalized, commit the tests, package, and
   example to source control.

This article demonstrates how [skaffold](https://skaffold.dev) can speed up the
iterative development loop (step 5) and provide real-time feedback when writing
Crossplane Packages. While skaffold's primary purpose is to support building
containers and testing code its deploy functions are just as useful for working
with Packages.

## Let's get Started!

You can find a complete example of this tutorial in the [A Tour of
Crossplane](https://github.com/AaronME/ATourOfCrossplane/tree/main/crossplane-package-tdd-with-skaffold)
repo.

### Prerequisites

We're going to assume you have the following installed:
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [helm](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kuttl](https://kuttl.dev)

#### Installing skaffold

I am running Mac OS X, so I installed kuttl using homebrew:

```bash
brew install skaffold
```

You can find installation instructions for other platforms on
[skaffold's install page](https://skaffold.dev/docs/install/).

### Step One: Launch Kind

We're going to quickly spin-up a cluster for local development using kind. You
can call it anything, but I tend to call mine "crossplane-tour."

```bash
$ kind create cluster --name crossplane-tour
Creating cluster "crossplane-tour" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Waiting ≤ 5m0s for control-plane = Ready ⏳
 • Ready after 4m2s 💚
```

Now we're ready to set up skaffold and start our dev loop.

### Step Two: Use Skaffold to Helm Install Crossplane and Provider

We need Crossplane running on the cluster to provision Composite Resources and
render Compositions. Skaffold supports running helm install in its __deploy__
configuration. Let's initiate our Skaffold project by creating the skaffold
__Config__ file -- called __skaffold.yaml__ -- at the root of our package repo.

__skaffold.yaml__:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        version: 1.5.1-up.1
        wait: true
```

We use **universal-crossplane** at Upbound, but you can use vanilla
**crossplane**. They will both work for this demo.

For this demo we need
[provider-aws](https://github.com/crossplane/provider-aws). The helm chart for
crossplane supports passing in a list of provider packages during install. We'll
do this using a values file.

Using a values file will make it easier to keep the test dependencies up to date
with the actual dependencies, declared in a __crossplane.yaml__. We'll keep the
values file under the __tests__ folder.

__tests/uxp-values.yaml__:
```yaml
provider:
  packages:
    - crossplane/provider-aws:v0.19.0
```

We want to wait for the provider to be installed and ready before we attempt
to install any compositions. Skaffold supports both pre- and post-deploy
__hooks__. These hooks will run before and after _all_ helm releases are
installed. Skaffold does not currently support inserting hooks between helm
releases.

**skaffold.yaml**:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws",
              ]
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        valuesFiles:
          - tests/uxp-values.yaml
        version: 1.5.1-up.1
        wait: true
```

With the above, we are ready to run __skaffold dev__ for the first time.

### Step Three: Skaffold Dev

The key to our development environment is the [skaffold
dev](https://skaffold.dev/docs/references/cli/#skaffold-dev) command. This
command runs our pipeline in a loop, watching our local files and re-running the
pipeline whenever changes are saved to disk.

We can see it in action throughout the rest of the demo by starting it now.

One note: we're going to run with the flag __--cleanup=false__. This will ensure
our resources are not deleted if we hit an error. I have defaulted to using this
flag, as my cleanup amounts to deleting the kind cluster. There is no way to set
this in __skaffold.yaml__.

```bash
$ skaffold dev --cleanup=false
Listing files to watch...
Generating tags...
Checking cache...
Tags used in deployment:
Starting deploy...
Loading images into kind cluster nodes...
Images loaded in 232ns
Helm release universal-crossplane not installed. Installing...
NAME: universal-crossplane
LAST DEPLOYED: ...
NAMESPACE: upbound-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
 ...
Waiting for deployments to stabilize...
 - upbound-system:deployment/xgql is ready. [3/4 deployment(s) still pending]
 - upbound-system:deployment/crossplane is ready. [2/4 deployment(s) still pending]
 - upbound-system:deployment/crossplane-rbac-manager is ready. [1/4 deployment(s) still pending]
 - upbound-system:deployment/upbound-bootstrapper is ready.
Deployments stabilized in 2.61 seconds
Starting post-deploy hooks...
provider.pkg.crossplane.io/crossplane-provider-aws condition met
Completed post-deploy hooks
Waiting for deployments to stabilize...
Deployments stabilized in 322.442326ms
Press Ctrl+C to exit
Watching for changes...
```

Leave this terminal open and running __skaffold dev__.

### Step Four: Set Up Tests

Let's define our test cases before adding any compositions or claims.

Hint: This demo uses the
[platform-ref-aws](https://github.com/upbound/platform-ref-aws) Network
resource.

Create the __tests/compositions/compositenetwork__ folders and supply a test
assertion.

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
spec:
  forProvider:
    region: us-west-2
    cidrBlock: 192.168.128.0/18
    vpcIdSelector:
      matchControllerRef: true
    availabilityZone: us-west-2a
---
apiVersion: ec2.aws.crossplane.io/v1beta1
kind: Subnet
metadata:
  labels:
    access: public
    crossplane.io/claim-name: network
    networks.aws.platformref.crossplane.io/network-id: platform-ref-aws-network
    zone: us-west-2a
spec:
  forProvider:
    region: us-west-2
    mapPublicIPOnLaunch: true
    cidrBlock: 192.168.0.0/18
    vpcIdSelector:
      matchControllerRef: true
    availabilityZone: us-west-2a
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
    region: us-west-2
    cidrBlock: 192.168.0.0/16
    enableDnsSupport: true
    enableDnsHostNames: true
    tags:
      - key: crossplane-kind
        value: vpc.ec2.aws.crossplane.io
      - key: crossplane-name
      - key: crossplane-providerconfig
```

Next, add a kuttl __TestStep__ to install the resource.

__tests/compositions/compositenetwork/01-network.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestStep
commands:
  # Wait for XRD to become "established"
  - command: kubectl wait --for condition=established --timeout=20s xrd/compositenetworks.aws.platformref.crossplane.io
  # Create the XR/Claim
  - command: kubectl apply -f "${PWD}/examples/network.yaml"
```

And you'll need to setup a kuttl __TestSuite__ file. Ensure that __startKind__
and __skipClusterDelete__ are set to their proper values. Note, too, that
__kindContext__ must match the name of your kind cluster.

__kuttl-test.yaml__:
```yaml
apiVersion: kuttl.dev/v1beta1
kind: TestSuite
startKIND: false
testDirs:
- tests/compositions
kindContext: kuttl-test
skipClusterDelete: true
```

Before we can add this test to our pipeline, we need to configure the XRD and
example.

### Step Five: Add Composite Resources to the Control Plane

We've sourced these Composite Resources from the
[platform-ref-aws](https://github.com/upbound/platform-ref-aws) package on
github. Because we are demonstrating local development, we'll save files to disk
rather than installing from the web.

Create folders for __examples__ and __packages/network__. Then download files from the __platform-ref-aws__ repo.

```bash
curl https://raw.githubusercontent.com/upbound/platform-ref-aws/main/examples/network.yaml -o examples/network.yaml -s
curl https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/definition.yaml -o package/network/definition.yaml -s
curl https://raw.githubusercontent.com/upbound/platform-ref-aws/main/network/composition.yaml -o package/network/composition.yaml -s
```

#### Deploy Kubernetes Manifests with Skaffold

Earlier we mentioned that Skaffold does not support inserting hooks between helm
releases. But Skaffold _does_ allow multiple deployment methods, including
applying static kubernetes manifests. We are going to take advantage of this
feature to apply our XRD and Compositions after the post-deploy hook for
crossplane has completed.

This will only apply kubernetes manifests and cannot be used to run kubectl
plugins.

Add a __kubectl__ block to your configuration and add the XRD and Compositions.

__skaffold.yaml__:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws",
              ]
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        valuesFiles:
          - tests/uxp-values.yaml
        version: 1.5.1-up.1
        wait: true
  kubectl:
    manifests:
      - package/network/definition.yaml
      - package/network/composition.yaml
```

Once you've saved these entries to your file, you should notice __skaffold dev__
picking them up and applying them.

```bash
Listing files to watch...
Generating tags...
Checking cache...
Tags used in deployment:
Starting deploy...
Loading images into kind cluster nodes...
Images loaded in 224ns
Waiting for deployments to stabilize...
 - upbound-system:deployment/xgql is ready. [3/4 deployment(s) still pending]
 - upbound-system:deployment/crossplane is ready. [2/4 deployment(s) still pending]
 - upbound-system:deployment/upbound-bootstrapper is ready. [1/4 deployment(s) still pending]
 - upbound-system:deployment/crossplane-rbac-manager is ready.
Deployments stabilized in 17.76 seconds
Starting post-deploy hooks...
provider.pkg.crossplane.io/crossplane-provider-aws condition met
Completed post-deploy hooks
Loading images into kind cluster nodes...
Images loaded in 278ns
 - compositeresourcedefinition.apiextensions.crossplane.io/compositenetworks.aws.platformref.crossplane.io created
 - composition.apiextensions.crossplane.io/compositenetworks.aws.platformref.crossplane.io created
Waiting for deployments to stabilize...
Deployments stabilized in 151.939351ms
Press Ctrl+C to exit
Watching for changes...
```

 It will do this every time you change
__skaffold.yaml__ or any file referenced by __skaffold.yaml__.

### Step Six: Add Tests to Skaffold Pipeline

Now that our XRD is established and offered, we can add our kuttl test step.

The same pre- and post-deploy __hooks__ we used on helm are available for
kubectl. So we add a post-deploy hook for our resources.

__skaffold.yaml__:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws",
              ]
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        valuesFiles:
          - tests/uxp-values.yaml
        version: 1.5.1-up.1
        wait: true
  kubectl:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl kuttl test --test ./tests/compositions/compositenetwork/; exit 0",
              ]
    manifests:
      - package/network/definition.yaml
      - package/network/composition.yaml
```

We've added '; exit 0' to the end of our command. We want to stop
the test from exiting with its normal error code, as this will cause skaffold to
completely stop the dev loop.

#### Success!!

You will get a successful test run. 

```bash
...
--- PASS: kuttl (19.66s)
    --- PASS: kuttl/harness (0.00s)
        --- PASS: kuttl/harness/compositenetwork (12.11s)
PASS
Completed post-deploy hooks
Waiting for deployments to stabilize...
Deployments stabilized in 351.199244ms
Press Ctrl+C to exit
Watching for changes...
```

But there's one more catch.

#### Including the Composed Resource

We are applying our __examples/network.yaml__ file from the kuttl __TestStep__.
This means Skaffold is unaware of the file and not watching it for changes. (Go
ahead and edit the file, watch skaffold do nothing.)

To get Skaffold to update on changes to our example, we must add it to the list
of manifests.

__skaffold.yaml__:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws",
              ]
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        valuesFiles:
          - tests/uxp-values.yaml
        version: 1.5.1-up.1
        wait: true
  kubectl:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl kuttl test --test ./tests/compositions/compositenetwork/; exit 0",
              ]
    manifests:
      - package/network/definition.yaml
      - package/network/composition.yaml
      - examples/network.yaml
```

Be warned: This configuration will fail on a first run because the network
resource will not be offered when skaffold applies the example.

To deal with this, we comment out the example when committing to the repo. Only
after __skaffold dev__ is running and the resource is healthy do we un-comment
it.


## The Finished Product

In the end, our folder structure for a configuration package including skaffold
dev environment will look like the following:

```bash
.
├── examples # CompositeResource examples
│   └── network.yaml
├── kuttl-test.yaml # TestSuite configuration
├── package # XRDs and Composites
│   ├── crossplane.yaml # Configuration Package metadata file
│   └── network
│       ├── composition.yaml
│       └── definition.yaml
├── skaffold.yaml # Skaffold Configuration File
└── tests # TestSteps
    ├── compositions
    │   └── compositenetwork
    │       ├── 01-assert.yaml
    │       └── 01-network.yaml
    └── uxp-values.yaml # Values for Universal-Crossplane
```

And this is the complete __skaffold.yaml__:
```yaml
apiVersion: skaffold/v2beta24
kind: Config
deploy:
  helm:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl wait --for condition=healthy --timeout=300s provider/crossplane-provider-aws",
              ]
    releases:
      - name: universal-crossplane
        repo: https://charts.upbound.io/stable/
        remoteChart: universal-crossplane
        namespace: upbound-system
        createNamespace: true
        valuesFiles:
          - tests/uxp-values.yaml
        version: 1.5.1-up.1
        wait: true
  kubectl:
    hooks:
      after:
        - host:
            command:
              [
                "sh",
                "-c",
                "kubectl kuttl test --test ./tests/compositions/compositenetwork/; exit 0",
              ]
    manifests:
      - package/network/definition.yaml
      - package/network/composition.yaml
      # This file can only be un-commented once the XRD is healthy in the Control Plane
      # - examples/network.yaml
```

## Useful Links

- [skaffold.yaml configuration reference](https://skaffold.dev/docs/references/yaml/) 
- [skaffold cli reference](https://skaffold.dev/docs/references/cli/) 
- [Crossplane Terminology](https://crossplane.io/docs/v1.6/concepts/terminology.html#crossplane-terms)

