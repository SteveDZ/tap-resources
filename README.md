# tap-resources
Repository containing all resources and instructions to install TAP on a cloud distribution

## Installing TAP 1.1.0 on GKE

The instructions followed can be found here: https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-install-intro.html

1. Create a k8s cluster on Google Cloud.
Assign a nodepool of 4 nodes. E2-standard-2 (2Cpu, 8Gb ram).
--> RAN OUT OF CPU WITH THIS SETUP! Experiment with 4 nodes high cpu, cpu4 mem 4
2. Connect kubectl to that context using the gcloud command which you can get from the UI displaying your cluster.

### Install Cluster essentials to your cluster

Follow the instructions from: https://docs.vmware.com/en/Cluster-Essentials-for-VMware-Tanzu/1.1/cluster-essentials/GUID-deploy.html

Cluster essentials installs the kapp-controller which is heavily used by the tanzu cli.
It also installs a secretgen-controller which allows us to create secrets that are automatically exported to all namespaces. 
Specifically secrets used to pull and push images from tanzu harbor and to our preferred tap registry. 
Secrets are also used by TBS to push workload images to internal registries.

```
export INSTALL_BUNDLE=registry.tanzu.vmware.com/tanzu-cluster-essentials/cluster-essentials-bundle@sha256:ab0a3539da241a6ea59c75c0743e9058511d7c56312ea3906178ec0f3491f51d
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=TANZU-NET-USER
export INSTALL_REGISTRY_PASSWORD=TANZU-NET-PASSWORD
cd $HOME/tanzu-cluster-essentials
./install.sh --yes
```

Instructions on how the uninstall cluster essentials can be found on the same page!

### Install Tanzu CLI

Follow the instructions from: https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-install-tanzu-cli.html

If you have an older version of the CLI already installed, make sure to uninstall that version first!

After installing the CLI, we need to install a couple of plugins necessary to install TAP to our cluster.
For more information, see: https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-install-tanzu-cli.html#cli-plugin-install

### Installing the Tanzu Application Platform package and profiles

Follow the instructions found here: https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-install.html

For experimental purposes, we chose NOT to relocate the tap-packages to our own repository but to continue relying on tanzu harbor registy.

Once we installed the secret and package repository to our tap-install namespace, we're able to install a tanzu application platform profile.

We will chose to install the full profile so we can experiment with all TAP components.

### Installing a Tanzu Application Platform profile

When installing a TAP profile, you'll have to make use of a tap-values file that contains the configuration for all the TAP components you're installing.
You also indicate which profile you wish to install.
When installing the components, often times, there's a need to configure usernames and passwords to certain container registries.
We do not want to hardcode these values, so we'll use tools such as ytt to be able to configure these values on our local machines and create a templatized tap-values file.

Check the tap-values.yaml which will be used when installing tap and the values.yaml file which contains our variables.

We will be installing the full profile. 
This includes grype for image scanning.
Grype needs access to the GCR registry to pull images and scan them.
Therefore, we'll create a secret that allows just that:
```
tanzu secret registry add gcr-secret --server="eu.gcr.io" --username="_json_key" --password="$(cat ~/.ssh/sdezitter-labs-build-service.json)" --export-to-all-namespaces --yes --namespace tap-install
```

#### Configure Pod Security Policies so that Tanzu Application Platform controller pods can run as root.
```
kubectl create clusterrolebinding tap-psp-rolebinding --group=system:authenticated --clusterrole=gce:podsecuritypolicy:privileged
```

#### Create cnrs-network-config secret

We create a CNRS network config secret, which we can later refer to.
We will annotate all packageinstalls from tap with a carvel annotation that refers to that config secret.
This annotation will cause the output if application URL to use https instead of http.

```
kapp deploy \
  --app tap-overlay-cnrs-network \
  --namespace tap-install \
  --file <(\
    kubectl create secret generic tap-pkgi-overlay-0-cnrs-network-config \
      --namespace tap-install \
      --from-file="tap-pkgi-overlay-0-cnrs-network-config.yaml=${script_dir}/overlays/cnrs/tap-pkgi-overlay-0-cnrs-network-config.yaml" \
      --dry-run=client \
      --output=yaml \
      --save-config \
  ) \
  --yes
```

${script_dir}: script_dir="$(cd $(dirname "$BASH_SOURCE[0]") && pwd)"

#### Create the development namespace

Be sure to create the developer namespace before installing TAP!
This is normally being done after installing a profile, but certain components (grype) seem to rely on this namespace being created during packageinstall.

```
kapp deploy \
  --app "tap-dev-ns-wine-cellar" \
  --namespace tap-install \
  --file <(\
    kubectl create namespace wine-cellar \
      --dry-run=client \
      --output=yaml \
      --save-config \
    ) \
  --yes
```

#### Creating the actual profile

First, create a temp values.yaml file that will be used during installation.
Our goal is to use ytt to overlay the values.yaml with the values from tap-values.yaml.

```
ytt -f "values-template.yaml" -f "values.yaml" --ignore-unknown-comments > "tap-values.yaml"
```

The above command results in a tap-values.yaml

We can now install TAP to our cluster, running:

```
tanzu package install tap \
  --namespace tap-install \
  --package-name tap.tanzu.vmware.com \
  --version 1.1.0 \
  --values-file "tap-values.yaml"
```

Updating tap using a changed tap-values.yaml file can be done using:

```
tanzu package installed update tap --package-name tap.tanzu.vmware.com --version 1.1.0 --values-file tap-values.yaml -n tap-install
```

#### Create developer namespace

https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-install-components.html#setup

Create a developer namespace. 
This involves creating secrets to be able to interact with the registry we've setup. 
And where TAP and TBS will push their images to.

```
tanzu secret registry add registry-credentials --server eu.gcr.io --username _json_key --password $(cat ~/.ssh/YOUR_JSON_KEY) --namespace YOUR-NAMESPACE
```

In my case:

```
tanzu secret registry add registry-credentials --server eu.gcr.io --username _json_key --password "$(cat ~/.ssh/sdezitter-labs-build-service.json)" --namespace wine-cellar
```

Additional developer namespaces can be provisioned using the ```configure-dev-space.sh``` script.

```
./configure-dev-space.sh wine-cellar
```

#### Create a workload and have the testing-scanning supply chain build it for us!

Installing the full profile means we get a couple of supply chains out of the box.
Specifically, we chose to have the testing-scanning supply chain installed.
However, this DOESNT WORK OUT OF THE BOX!

See this getting started section to learn what we need to do to get this to work: https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-getting-started.html#section-3-add-testing-and-security-scanning-to-your-application-17

More specifically, we need to provide the tekton pipeline that will ultimately run our tests!

Apply below yaml config () to have a tekton pipeline that will be picked up by the supply chain (See vmware.com/pipeline: test label) and will execute the tests in our maven project.

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: developer-defined-tekton-pipeline
  labels:
    apps.tanzu.vmware.com/pipeline: test     # (!) required
spec:
  params:
    - name: source-url                       # (!) required
    - name: source-revision                  # (!) required
  tasks:
    - name: test
      params:
        - name: source-url
          value: $(params.source-url)
        - name: source-revision
          value: $(params.source-revision)
      taskSpec:
        params:
          - name: source-url
          - name: source-revision
        steps:
          - name: test
            image: gradle
            script: |-
              cd `mktemp -d`

              wget -qO- $(params.source-url) | tar xvz -m
              ./mvnw test
```

```
k apply -f supply-chain/tekton/pipeline-test.yaml -n wine-cellar
```

Scanning is another capability that needs a resource in the developer namespace to work, a ScanPolicy:

```
kubectl apply -f supply-chain/scanning/scan-policy.yaml -n wine-cellar
```

#### Create workload

We can now create cartohgrapher workloads. 
This will cause our supply-chains to be triggered and our app to be built, containerized and deployed.
We started with the spring-java-web-app application accelerator to create a standard workload.
applying small changes to the Tiltfile allows us to deploy this Workload using the k apply command.

```
apiVersion: carto.run/v1alpha1
kind: Workload
metadata:
  name: testje
  labels:
    apps.tanzu.vmware.com/workload-type: web
    app.kubernetes.io/part-of: testje
    apps.tanzu.vmware.com/has-tests: true
spec:
  params:
  - name: annotations
    value:
      autoscaling.knative.dev/minScale: "1"
  source:
    git:
      url: https://github.com/sample-accelerators/testje
      ref:
        branch: main
```

## General

### kapp deploy

We use kapp deploy often because it is repeatable.
We create our resources using kapp and kapp in turn applies our k8s config. 
The k8s config is created using imperative commands.
Using kapp, the next time we run this command (or a newer version of it), it will perform a diff compared with the previous resource and determine what needs to be done (how to update or delete the resource).
