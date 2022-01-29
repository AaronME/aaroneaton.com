---
title: Conformance Testing Crossplane Configuration Packages with Sonobuoy
date: 2022-01-28
description: 'Conformance Testing Crossplane Configuration Packages with https://sonobuoy.io'
image: images/crossplane/crossplane-package-testing-with-sonobuoy/sonobuoy_and_crossplane.svg
---

## Conformance Tests for Crossplane Configuration Packages

When crossplane
[graduated](https://www.cncf.io/blog/2021/09/14/crossplane-moves-from-sandbox-to-cncf-incubator)
to Incubation status we published a
[conformance](https://github.com/crossplane/conformance) for crossplane and its
providers. I started to wonder if we could use the same tools and framework for
testing Configuration Packages, as well as Provider Packages. 

A unified testing strategy for all crossplane packages could simplify our
build and test pipelines. And it would provide a method for testing and
validating external packages. Especially if we adopted a tool and framework
which were already community standards.

### Sonobuoy

[sonobuoy](https://sonobuoy.io) is "a diagnostic tool that makes it easier to
understand the state of a Kubernetes cluster by running a choice of
configuration tests in an accessible and non-destructive manner."

As we mentioned
[before](http://localhost:1313/crossplane/crossplane-package-testing-with-kuttl/),
testing a crossplane configuration package will always be an e2e test of a
kubernetes cluster running crossplane. And sonobuoy was built around the
Kubernetes e2e-framework.

We developed a demo of sonobuoy and the e2e-framework for our internal teams.
And along the way we found a solution to writing golang tests against untyped
Resources in the cluster. This is how we did it.

## Let's get Started!

You can find a complete example of this tutorial in the [A Tour of Crossplane]()
repo.

### Prerequisites

We're going to assume you have the following installed:
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [helm](https://helm.sh/docs/intro/install/)
- [up cli](http://cloud.upbound.io/docs/cli)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

#### Installing sonobuoy

I am running Mac OS X, so I installed sonobuoy using homebrew:

```bash
brew install sonobuoy
```

You can find installation instructions for other platforms on
[sonobuoy's install page](https://sonobuoy.io/docs/v0.56.0/#installation).

### Step One: Launch Kind

Let's get started in a fresh repo. First, bring up a kind cluster and get 
crossplane and your providers running on it:

1. Create a 'uxp-values.yaml' file that installs the platform-ref-aws package.
   (provider-aws will install automatically as a dependency.)

	__uxp-values.yaml__:
	```yaml
	---
	configuration:
	packages:
		- registry.upbound.io/upbound/platform-ref-aws:v0.2.1
	
1. Stat the cluster and install crossplane.

	```bash
	kind create cluster --name sonobuoy-test
	up uxp install -f uxp-values.yaml

With our development cluster in place, we can move on to the coding.

#### Do I need to clone platform-ref-aws?

No. Sonobuoy, and the kubernetes e2e-framework, are built on top of the golang
_testing_ library. As such, all of the test components -- including test cases
and test resoruces -- will be defined in the code. At the end, the entire suite
will be built into a docker image and pushed to a repo.

This creates a test tool which can be run on a local cluster, a remote environment,
or even made available to the public to confirm your package compatibily with
other systems.

### Step Two: Download the e2e-example Plugin Starter

Sonobuoy offers several plugin [examples](https://github.com/vmware-tanzu/sonobuoy-plugins)
we can use for starters. We're going to use the 'e2e-skeleton.' (If you want to
know more, you can check out [sonobuoy's blog](https://sonobuoy.io/plugin-starter/) on it.) 

Run the following to get the starter plugin:

```bash
git clone https://github.com/vmware-tanzu/sonobuoy-plugins
cp -r sonobuoy-plugins/examples/e2e-skeleton/ ./plugin
rm -rf sonobuoy-plugins
# The next line will update the name of your module
export MODULE_HEAD="module <your-module-git-url-here>"
sed -i '' "1 s,.*,${MODULE_HEAD}," plugin/go.mod
pushd plugin/pkg
```

You are now in the root of your code pkg.

#### Running from the command line

The plugin is designed run within a cluster. If we try to run the code now, we
would see:

```bash
envconfig: client failed: unable to load in-cluster configuration, KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined
FAIL    github.com/aaronme/ATourOfCrossplane/crossplane-package-testing-with-sonobuoy/pkg       0.312s
FAIL
```

Let's enable our pluging to use a kubeconfig so we can run it from the
commandline.

1. Add an ENV_VAR for the kubeconfig:
	__main_test.go__:
	```golang
	const (
		ProgressReporterCtxKey    = "SONOBUOY_PROGRESS_REPORTER"
		NamespacePrefixKey        = "NS_PREFIX"
		ExternalClusterKubeconfig = "EXTERNAL_KUBECONFIG" // Accept a Kubeconfig
	)
1. Replace the _config.NewClient()_ call on line 53.
	__main_test.go__:
	```golang
		// Try and create the client; doing it before all the tests allows the tests to assume
		// it can be created without error and they can just use config.Client().

 		// If EXTERNAL_KUBECONFIG is set, create a client based on the kubeconfig
		externalKubeConfig := os.Getenv(ExternalClusterKubeconfig)

			var err error
			if externalKubeConfig != "" {
				_, err = config.WithKubeconfigFile(externalKubeConfig).NewClient()
			} else {
				_, err = config.NewClient()
			}
1. Export a kubeconfig from kind and setup the ENV_VARs:
	```bash
	kind export kubeconfig --name sonobuoy-test --kubeconfig kubeconfig
	export EXTERNAL_KUBECONFIG=kubeconfig
	export NS_PREFIX=tour # We also want set a prefix for our namespace names
1. Run the tests
	```bash
	go test ./...

Success! You should be looking at your first test result:

```bash
ok      github.com/aaronme/ATourOfCrossplane/crossplane-package-testing-with-sonobuoy/pkg       25.337s
```

Let's go ahead and create our own test file now. Because we are using the go
'testing' framework the file name must end with '_test.go'.

```bash
touch platform_ref_aws_test.go
```
### Step Three: Our First Test

This is what our code needs to do:
1. Create a Composite Resource Claim
1. Confirm the claim generated a Composite Resource (XR)
1. Confirm that we got all the Managed Resources (MRs) form the XR that we
   expect

#### Our Big Problem

Here's how __custom_test.go__ accesses resources in the cluster:
```golang
			var pods corev1.PodList
			err := cfg.Client().Resources("kube-system").List(context.TODO(), &pods)
			if err != nil {
				t.Fatal(err)
			}
```

They create a _corev1.PodList_ object, then use a client to populate that
object with data from the api. The object type is imported from _k8s.io/api/core/v1_.

Unfortunately, we cannot import types from our configuration packages.

#### XRDs, CRDs, and Types

A Crossplane Composite Resource Definition (XRD) defines the schema for a
Composite Resource. Crossplane creates a CustomResourceDefinition based on this 
schema, and installs it on the cluster.

Unlike managed resources in an open-source provider, we do not have access to
the type library for a given package. Because it was never defined in go.

If we want to test Composites and Claims in our cluster, we're going to need
some way to access objects without knowing their types.

#### Learning about the Dynamic Client

After spending a week with _runtime.Object{}_ I was able to teach myself that
interfaces are not as magical as I thought they were. Interfaces are a way to
use multiple types in a common method, but the types must be _written to_ the
interface.

We aren't trying to work with multiple types. We're trying to work with _no_
types. We'll never have them.

Finally, I found [The Kubernetes Dynamic
Client](https://caiorcferreira.github.io/post/the-kubernetes-dynamic-client/).
And then I found [Unstructured Kubernetes Components with client-go's Dynamic
Client](https://trstringer.com/kubernetes-dynamic-client-go/).

We are _not_ the first to encounter this problem.

### Step Four: Implement the Dynamic Client

We're not going to switch out the current client e2e uses, because this is just
a demo. We don't want to fall down the rabbit hole of needing to re-write all of
the functions that are currently working. So we'll write a quick function to
get a Dynamic Client when we need one:

*Note*: We are also going to support the external kubeconfig with this client.

__main_test.go__:
```golang
...
// Add these to your list of imports
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
...
// Add this to the end of your file
func newDynamicClient() (dynamic.Interface, error) {
	externalKubeConfig := os.Getenv(ExternalClusterKubeconfig)

	if externalKubeConfig != "" {
		config, err := clientcmd.BuildConfigFromFlags("", "./kubeconfig")
		if err != nil {
			fmt.Printf("error getting Kubernetes config: %v\n", err)
			return nil, err
		}

		dynClient, err := dynamic.NewForConfig(config)
		if err != nil {
			return nil, err
		}

		return dynClient, nil

	} else {
		config, err := rest.InClusterConfig()
		if err != nil {
			return nil, err
		}

		dynClient, err := dynamic.NewForConfig(config)
		if err != nil {
			return nil, err
		}

		return dynClient, nil
	}
}
```

*NOTE*: I'm trusting you to run __go mod download__ as needed when we add
modules. 

### Step ~~Three~~ Five: Let's write that test!

Declare package and imports.

__platform_ref_aws_test.go__:
```golang
package pkg

import (
	"context"
	"fmt"
	"testing"
	"time"

	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/e2e-framework/pkg/envconf"
	"sigs.k8s.io/e2e-framework/pkg/features"
	"sigs.k8s.io/yaml"
)
```

#### Define GroupVersionResource and Claim for Test

The Dynamic Client works like the regular, typed client in _custom_test.go_. But
instead of passing an object of the Kind we want, we pass it an explicit
definition of the Group, Version, and Resource.

We are going to be working with both a Claim and XR, so we'll define
GroupVersionResources for both. And we'll define the Claim we want to test:

__platform_ref_aws_test.go__:
```golang
func TestPlatformRefAWS(t *testing.T) {
	// We need to declare the GVR for both our XRs and Claims
	// This is the XR GroupVersionResource
	compositeResource := schema.GroupVersionResource{
		Group:    "aws.platformref.crossplane.io",
		Version:  "v1alpha1",
		Resource: "compositenetworks", // remember to use the plural
	}

	// This is the Claim GroupVersionResource
	claimResource := schema.GroupVersionResource{
		Group:    "aws.platformref.crossplane.io",
		Version:  "v1alpha1",
		Resource: "networks", // remember to use the plural
	}

	// This is a claim from https://github.com/upbound/platform-ref-aws/blob/main/examples/network.yaml
	claim := `---
apiVersion: aws.platformref.crossplane.io/v1alpha1
kind: Network
metadata:
  name: network
spec:
  id: platform-ref-aws-network
  clusterRef:
  	id: platform-ref-aws-cluster
`
	// Unmarshal the Claim yaml into an Unstructured Resource.
	// This requires going through Json.
	unstructuredClaim := unstructured.Unstructured{}
	json, err := yaml.YAMLToJSON([]byte(claim))
	if err != nil {
		t.Fatal(err)
	}
	err = unstructuredClaim.UnmarshalJSON(json)
	if err != nil {
		t.Fatal(err)
	}
```

#### Test Step 1: Create the Claim

Claims are namespace-scoped, and our namespaces are being dynamically generated
by the e2e-framework. We can access the namespace name in the context, but we
must grab the lookup key before we run our first Feature test. This is because
the name of each test is a '/' joined concatenation of the Test Suite Name,
Feature Name, and Test Step Name.

So, first we add this:

```golang
	// We need to capture the namespace key name here because the test name
	// changes inside Features and Assess methods
	namespaceKey := nsKey(t)
```

And then we can set the name of our resource in the Test Step:

```golang
	f := features.New("Rendered").
		Assess("Managed Resources", func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
			// We will name our claim after the namespace for the parent test.
			ns := fmt.Sprint(ctx.Value(namespaceKey))
			claimName := fmt.Sprintf("%s-claim", ns)

			unstructuredClaim.SetName(claimName)
			unstructuredClaim.SetNamespace(ns)
```

Now we get a Dynamic Client and create the Claim. We will wait three seconds
before confirming the claim was created:

```golang
			// The e2e-skeleton comes with a client based on their klient type.
			// We want to use a dynamic client, which enables working with
			// unstructured resources
			dynClient, err := newDynamicClient()
			if err != nil {
				t.Fatal(err)
			}

			// Create the Claim
			createClaim, err := dynClient.Resource(claimResource).Namespace(ns).Create(context.TODO(), &unstructuredClaim, v1.CreateOptions{})
			if err != nil {
				t.Logf("creating Claim failed: +%v", createClaim)
				t.Fatal(err)
			}

			time.Sleep(3 * time.Second)

			// Retrieve the claim. This confirms our resource created and is also
			// necessary to lookup the XR name.
			getClaim, err := dynClient.Resource(claimResource).Namespace(ns).Get(context.TODO(), claimName, v1.GetOptions{})
			if err != nil {
				t.Logf("getting Claim failed: +%v", getClaim)
				t.Fatal(err)
			}
```

#### Test Step 2: Confirm we Generated an XR

After our claim is created, we want to confirm we got a Composite Resource (XR)
for it. If we do not find an XR, we will throw an error:

```golang
			// Get the XR
			compositeName, exists, err := unstructured.NestedString(getClaim.UnstructuredContent(), "spec", "resourceRef", "name")
			if err != nil {
				t.Fatal(err)
			}

			if exists != true {
				t.Log(getClaim.UnstructuredContent())
				t.Fatal("No composite name found.")
			}

			// Our first test: confirm that an XR was created.
			// This failure will indicate whether a successful composition template
			// was selected or not
			t.Run("Did create XR", func(t *testing.T) {
				t.Logf("Fetching XR %s", compositeName)
				getXR, err := dynClient.Resource(compositeResource).Get(context.TODO(), compositeName, v1.GetOptions{})
				if err != nil {
					t.Logf("getting XR failed: +%v", getXR)
					t.Fatal(err)
				}
			})
```

#### Test Step 3: Confirm we Generated Expected Managed Resources

Now that we are dealing with managed resources, we can import the types library
from [provider-aws
types](https://github.com/crossplane/provider-aws/tree/master/apis). However,
for the sake of clarity, we'll continue working with unstructured objects in
this demo.

##### Table-Driven Tests

Crossplane [Contributing
guidelines](https://github.com/crossplane/crossplane/blob/master/CONTRIBUTING.md)
make it clear: "Crossplane encourages the use
of table driven unit tests." At Upbound, we maintain our open source principles
in-house, so let's make sure our tests honor the guidelines.

I found
[this](https://www.gopherguides.com/articles/table-driven-testing-in-parallel)
article was a helpful reference.

Create a table of test cases:

```golang
			// MRs we expect to be created by a Claim.
			// For the purpose of this demo, we only verify we have the correct number
			// of each resource type.
			var mrs = []struct {
				name  string
				gvr   schema.GroupVersionResource
				count int
			}{
				{"VPC", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "vpcs"}, 1},
				{"InternetGateway", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "internetgateways"}, 1},
				{"SecurityGroup", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "securitygroups"}, 1},
				{"Subnet", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "subnets"}, 4},
				{"RouteTable", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "routetables"}, 1},
			}
```

And loop through them:

```golang
			// MR Test Case Runs
			for _, mr := range mrs {
				mr := mr // rebind mr into this lexical scope
				t.Run(mr.name, func(t *testing.T) {

					got, err := dynClient.Resource(mr.gvr).List(context.TODO(), v1.ListOptions{})
					if err != nil {
						t.Errorf("error retrieving %q: %q", mr.name, err)
					}

					count := len(got.Items)

					if count != mr.count {
						t.Errorf("resource %q count is wrong.", mr.name)
					}
				})

			}
```

*NOTE*: This is a terrible test. Do not do this in actual practice.

Finally, return and run:

```golang
			return ctx
		})
	testenv.Test(t, f.Feature())
}
```

### Step Five: Test the Test

We can delete __custom_test.go__ now. We're not going to need it.

```bash
rm -rf custom_test.go
```

Run the suite:

```bash
go test ./... -v
```

And you should see:

```bash
...
--- PASS: TestPlatformRefAWS (3.14s)
    --- PASS: TestPlatformRefAWS/Rendered (3.10s)
        --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources (3.10s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/Did_create_XR (0.00s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/VPC (0.00s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/InternetGateway (0.00s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/SecurityGroup (0.00s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/Subnet (0.00s)
            --- PASS: TestPlatformRefAWS/Rendered/Managed_Resources/RouteTable (0.01s)
PASS
ok      github.com/aaronme/ATourOfCrossplane/crossplane-package-testing-with-sonobuoy/pkg       28.767s
```

### Step Six: Sonobuoy Run

Building and running the container requires enabling _buildx_ on your docker
daemon, which we won't cover here. But you can change to the __plugin/__ folder
and run the following to test the containerized solution.

```bash
export REPO_NAME=<your repo here>
export CONTAINER_TAG=<your tag here>
export CONTAINER_NAME=${REPO_NAME}:${CONTAINER_TAG}

cat <<EOF > plugin.yaml
sonobuoy-config:
  driver: Job
  plugin-name: custom-e2e
  result-format: gojson
  source_url: https://raw.githubusercontent.com/aaronme/ATourOfCrossplane/main/crossplane-package-testing-with-sonobuoy/plugin/plugin.yaml
  description: e2e test of the crossplane configuration package platform-ref-aws.
spec:
  command:
  - bash
  args: ["-c","go tool test2json ./custom.test -test.v | tee \${SONOBUOY_RESULTS_DIR}/out.json ; echo \${SONOBUOY_RESULTS_DIR}/out.json > \${SONOBUOY_RESULTS_DIR}/done"]
  image: ${CONTAINER_NAME}
  env:
  - name: NS_PREFIX
    value: custom
  - name: SONOBUOY_PROGRESS_PORT
    value: "8099"
  name: plugin
  resources: {}
  volumeMounts:
  - mountPath: /tmp/sonobuoy/results
    name: results
EOF

docker build -t ${CONTAINER_NAME} ./
kind load docker-image ${CONTAINER_NAME} --name sonobuoy-test
sonobuoy run --plugin plugin.yaml
sonobuoy wait
sonobuoy retrieve -f results.tar.gz
sonobuoy results results.tar.gz --mode dump
```

Your results should look something like this:
```yaml
name: custom-e2e
status: passed
meta:
  type: summary
items:
- name: out.json
  status: passed
  meta:
    file: results/global/out.json
    type: file
  items:
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/Did_create_XR
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/VPC
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/InternetGateway
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/SecurityGroup
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/Subnet
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources/RouteTable
    status: passed
  - name: TestPlatformRefAWS/Rendered/Managed_Resources
    status: passed
  - name: TestPlatformRefAWS/Rendered
    status: passed
  - name: TestPlatformRefAWS
    status: passed
```

## The Finished Product

In the end, our folder structure for a configuration package including kuttl
tests will look like the following:

```bash
.
├── plugin
│   ├── Dockerfile
│   ├── README.md
│   ├── build.sh
│   ├── go.mod
│   ├── go.sum
│   ├── pkg
│   │   ├── kubeconfig
│   │   ├── main_test.go
│   │   └── platform_ref_aws_test.go
│   └── plugin.yaml
└── uxp-values.yaml
```

And your complete test file looks like this:

__platform_ref_aws_test.go__:
```golang
package pkg

import (
	"context"
	"fmt"
	"testing"
	"time"

	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/e2e-framework/pkg/envconf"
	"sigs.k8s.io/e2e-framework/pkg/features"
	"sigs.k8s.io/yaml"
)

func TestPlatformRefAWS(t *testing.T) {
	// We need to declare the GVR for both our XRs and Claims
	// This is the XR GroupVersionResource
	compositeResource := schema.GroupVersionResource{
		Group:    "aws.platformref.crossplane.io",
		Version:  "v1alpha1",
		Resource: "compositenetworks", // remember to use the plural
	}

	// This is the Claim GroupVersionResource
	claimResource := schema.GroupVersionResource{
		Group:    "aws.platformref.crossplane.io",
		Version:  "v1alpha1",
		Resource: "networks", // remember to use the plural
	}

	// This is a claim from https://github.com/upbound/platform-ref-aws/blob/main/examples/network.yaml
	claim := `---
apiVersion: aws.platformref.crossplane.io/v1alpha1
kind: Network
metadata:
  name: network
spec:
  id: platform-ref-aws-network
  clusterRef:
    id: platform-ref-aws-cluster
`

	// Unmarshal the Yaml into an Unstructured Resource.
	// This requires going through Json.
	unstructuredClaim := unstructured.Unstructured{}
	json, err := yaml.YAMLToJSON([]byte(claim))
	if err != nil {
		t.Fatal(err)
	}
	err = unstructuredClaim.UnmarshalJSON(json)
	if err != nil {
		t.Fatal(err)
	}

	// We need to capture the namespace key name here because the test name
	// changes inside Features and Assess methods
	namespaceKey := nsKey(t)

	f := features.New("Rendered").
		Assess("Managed Resources", func(ctx context.Context, t *testing.T, cfg *envconf.Config) context.Context {
			// We will name our claim after the namespace for the parent test.
			ns := fmt.Sprint(ctx.Value(namespaceKey))
			claimName := fmt.Sprintf("%s-claim", ns)

			unstructuredClaim.SetName(claimName)
			unstructuredClaim.SetNamespace(ns)

			// The e2e-skeleton comes with a client based on their klient type.
			// We want to use a dynamic client, which enables working with
			// unstructured resources
			dynClient, err := newDynamicClient()
			if err != nil {
				t.Fatal(err)
			}

			// Create the Claim
			createClaim, err := dynClient.Resource(claimResource).Namespace(ns).Create(context.TODO(), &unstructuredClaim, v1.CreateOptions{})
			if err != nil {
				t.Logf("creating Claim failed: +%v", createClaim)
				t.Fatal(err)
			}

			time.Sleep(3 * time.Second)

			// Retrieve the claim. This confirms our resource created and is also
			// necessary to lookup the XR name.
			getClaim, err := dynClient.Resource(claimResource).Namespace(ns).Get(context.TODO(), claimName, v1.GetOptions{})
			if err != nil {
				t.Logf("getting Claim failed: +%v", getClaim)
				t.Fatal(err)
			}

			// Get the XR
			compositeName, exists, err := unstructured.NestedString(getClaim.UnstructuredContent(), "spec", "resourceRef", "name")
			if err != nil {
				t.Fatal(err)
			}

			if exists != true {
				t.Log(getClaim.UnstructuredContent())
				t.Fatal("No composite name found.")
			}

			// Our first test: confirm that an XR was created.
			// This failure will indicate whether a successful composition template
			// was selected or not
			t.Run("Did create XR", func(t *testing.T) {
				t.Logf("Fetching XR %s", compositeName)
				getXR, err := dynClient.Resource(compositeResource).Get(context.TODO(), compositeName, v1.GetOptions{})
				if err != nil {
					t.Logf("getting XR failed: +%v", getXR)
					t.Fatal(err)
				}
			})

			// MRs we expect to be created by a Claim.
			// For the purpose of this demo, we only verify we have the correct number
			// of each resource type.
			var mrs = []struct {
				name  string
				gvr   schema.GroupVersionResource
				count int
			}{
				{"VPC", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "vpcs"}, 1},
				{"InternetGateway", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "internetgateways"}, 1},
				{"SecurityGroup", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "securitygroups"}, 1},
				{"Subnet", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "subnets"}, 4},
				{"RouteTable", schema.GroupVersionResource{Group: "ec2.aws.crossplane.io", Version: "v1beta1", Resource: "routetables"}, 1},
			}

			// MR Test Case Runs
			for _, mr := range mrs {
				mr := mr // rebind mr into this lexical scope
				t.Run(mr.name, func(t *testing.T) {

					got, err := dynClient.Resource(mr.gvr).List(context.TODO(), v1.ListOptions{})
					if err != nil {
						t.Errorf("error retrieving %q: %q", mr.name, err)
					}

					count := len(got.Items)

					if count != mr.count {
						t.Errorf("resource %q count is wrong.", mr.name)
					}
				})

			}

			return ctx
		})
	testenv.Test(t, f.Feature())
}

```

## Useful Links

- [Skip the Boilerplate and Start Testing](https://sonobuoy.io/plugin-starter/)
- [Sonobouy-plugins Examples Repo](https://github.com/vmware-tanzu/sonobuoy-plugins/tree/main/examples)
- [The Kubernetes Dynamic Client](https://caiorcferreira.github.io/post/the-kubernetes-dynamic-client/)
- [Unstructured Kubernetes Components with client-go's Dynamic Client](https://trstringer.com/kubernetes-dynamic-client-go/)
- [Table Driven Testing in Parallel](https://www.gopherguides.com/articles/table-driven-testing-in-parallel)
- [Crossplane Terminology](https://crossplane.io/docs/v1.6/concepts/terminology.html#crossplane-terms)


