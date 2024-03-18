## Overview

This tutorial demonstrates a platform engineering approach to DataPower
development and deployment. It demonstrates continuous integration, continuous
deployment, GitOps, Infrastructure as Code and DevOps using containers,
Kubernetes and a set of popular cloud native tools such as ArgoCD and Tekton.

In this tutorial, you will:

1. Create a Kubernetes cluster and image registry, if required.
2. Create an operational repository to store DataPower resources that are
   deployed to the Kubernetes cluster.
3. Install ArgoCD to manage the continuous deployment of DataPower-related
   resources to the cluster.
4. Create a source Git repository that holds the DataPower development artifacts
   for a virtual DataPower appliance.
5. Install Tekton to provide continuous integration of the source DataPower
   artifacts. These pipeline ensures that all changes to these artifacts are
   successful built, packaged, versioned and tested before they are delivered
   into the operational repository, read for deployment.
6. Gain experience with the IBM-supplied DataPower operator and container.

By the end of the tutorial, you will have practical experience and knowledge of
platform engineering with DataPower in a Kubernetes environment.

---

## Introduction

The following diagram shows a CICD pipeline for DataPower:

![diagram1](./docs/images/diagram1.drawio.png)

Notice:

- The git repository `dp01-src` holds the source development artifacts for a
  virtual DataPower appliance `dp01`.
- A Tekton pipeline uses the `dp01-src` repository to build, package, test,
  version and deliver resources that define the `dp01` DataPower appliance.
- If the pipeline is successful, then the YAMLs that define `dp01` are stored in
  the operational repository `dp01-ops` and the container image for `dp01` is
  stored in an image registry.
- Shortly after the changes are committed to the git repository, an ArgoCD
  application detects the updated YAMLs. It applies them to the cluster to create or
  update a running `dp01` DataPower virtual appliance.

This tutorial will walk you through the process of setting up this
configuration:
- Step 1: Follow the instructions in this README to set up your cluster, ArgoCD,
  the `dp01-ops` repository, and Tekton. Continue to step 2.
- Step 2: Move to [these
  instructions](https://github.com/dp-auto/dpxx-src#readme) to create the
  `dp01-src` repository, run a Tekton pipeline to populate the `dp01-ops`
  repository, and interact with the new or updated DataPower appliance `dp01`.

---

## Install Kubernetes

At the moment, this Tutorial requires OpenShift. It will be updated to support
Minikube.

---

## Install Kubernetes CLI

To interact with your cluster from your local machine, you will need to use the 
`kubectl` or `oc` command line interface.

Add instructions to install `kubectl` or `oc` CLI.

---

## Login to the cluster

From the your OpenShift web console console, select `Copy login command`, and
copy the login command.

Login to cluster using this command, for example:

```bash
oc login --token=sha256~noTAVal1dSHA --server=https://example.cloud.com:31428
```

which should return a response like this:

```bash
Logged into "https://example.cloud.com:31428" as "odowda@example.com" using the token provided.

You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

indicating that you are now logged into the Kubernetes cluster.

---

## Create a GitHub organization for your repositories

This tutorial requires you to create the `dp01-src` and `dp01-ops` repositories.
It's a good idea to create them in a separate organization because it makes it
easy to collaborate with your colleagues later on.

<br>

Click on the following URL: [https://github.com/settings/organizations](https://github.com/settings/organizations)
to create a new organization:


<br> You will see a list of your existing GitHub organizations:

<img src="./docs/images/diagram6.png" alt="drawing" width="800"/>

You're going to create a new organization for this tutorial.

<br> Click on `New organization`:

<br> This shows you the list of available plans:

<img src="./docs/images/diagram7.png" alt="drawing" width="800"/>

The `Free` plan is sufficient for this tutorial.

<br> Click on `Create a free organization`:

<br> This shows you the properties for the new organization.

<img src="./docs/images/diagram8.png" alt="drawing" width="800"/>

<br> Complete the details for your new organization.

* Specify `Organization account name` of the form `dporg-xxxxx` where `xxxxx` is
  your GitHub user name.
* Specify `Contact mail` e.g. `odowda@example.com`
* Select `My personal account`.

<br> Once you've complete this page, click `Next`:

<br> Your new organization `dporg-xxxxx` has now been created:
<img src="./docs/images/diagram9.png" alt="drawing" width="800"/>

You can add colleagues to this organization each with a particular role. For
now, we can use the organization as-is.

<br> Click on `Complete setup` to complete the organization creation process.

<br> Although you may see a few more screens, such as a usage survey, your
organization has been now been created. We will use it to host the `dp01-src`
and `dp01-ops` repositories in this tutorial.

---

##  Useful environment variables

We now define some environment variables that are used by the commands in this
tutorial to make them easy to use.

Define your GitHub user name in the `GITUSER` variable using the name you
supplied above, e.g. `odowdaibm`. Your GitHub user name will be used to 

Open a new Terminal window and type:

```bash
export GITUSER=odowdaibm
export GITORG=dporg-$GITUSER
```

Let's use this environment variable to examine your new organization in GitHub.

Enter the following command:

```bash
echo https://github.com/orgs/$GITORG/repositories
```

which will respond with a URL of this form:

```bash
https://github.com/orgs/dporg-odowdaibm/repositories
```

Navigate to this URL in your browser:

<img src="./docs/images/diagram10.png" alt="drawing" width="800"/>

You can see that your new organization doesn't yet have any repositories in it.
Let's start by adding the `dp01-ops` repository to it.

---

## Creating the `dp01-ops` repository

We use a [template repository](https://github.com/dp-auto/dpxx-ops) to
create `dp01-ops` in our new organization. Forking a template creates a
repository with a clean git history, allowing us to track the history of changes
to our cluster every time we update `dp01-ops`.

<br> Click on [this URL](https://github.com/dp-auto/dpxx-ops/generate) to fork from
the `dpxx-ops` template repository:

<img src="./docs/images/diagram11.png" alt="drawing" width="800"/>

This screen allows you to define the properties for your copy of the `dp01-ops`
repository.

Specifically:

* In the `Owner` dropdown, select your newly created organization, e.g. `dporg-xxxxx`
* In the `Repository name` field, specify `dp01-ops`.
* In the `Description` field, specify `Operational repository for DataPower`.
* Select `Public` for the repository visibility.

<br> Click on `Create repository from template`:

<br> This repository will be cloned to the specified GitHub account:
<img src="./docs/images/diagram12.png" alt="drawing" width="800"/>

<br> You have successfully created a copy of the `dp01-ops` repository in your
organization.

---

## Enable Personal Access Tokens for your new organization

To allow sufficient, but limited, access the repositories in our new Git organization, we use a [Personal Access Token ](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) (**PAT**). First, we must enable this feature using the GitHub web.

Issue the following command:

```bash
echo https://github.com/organizations/$GITORG/settings/personal-access-tokens-onboarding
```

Navigate to this URL in your browser and complete the workflow:

<img src="./docs/images/diagram14.png" alt="drawing" width="800"/>

Select the following options via their radio buttons:

1. Select `Allow access via fine-grained personal access tokens` and hit `Continue`
2. Select `Do not require administrator approval` and hit `Continue`
3. Select `Allow access via personal access tokens (classic)` and hit `Continue` 
4. Complete the process and hit `Enroll` to enable PATs for your organization 

Personal Access Tokens are now enabled for your GitHub organization.

## Create Personal Access Token for GitHub access

We now create a PAT to limit access to only repositories in the new GitHub organization. 
This token will be used by the tutorial CLI and Tekton service account, thereby limiting 
exposure of your other GitHub resources; this is a nice example of **Zero Trust** security, 
limiting access to only what's required.

Navigate to https://github.com/settings/tokens?type=beta in your Browser:

<img src="./docs/images/diagram15.png" alt="drawing" width="800"/>

Complete the page as follows:

1. `Token Name`: `DataPower tutorial access`
2. `Description`: `This token allows a user to clone repositories and create/merge PRs`
3. `Resource Owner` drop-down:  Select your organization e.g. `dporg-odowdaibm`
4. Select the `All repositories` radio button
5. Under `Repository permissions` select:
   * `Contents`: `Read and write`
   * `Pull requests` : `Read and write`
6. (No changes under `Organization permissions`)   

Click on `Generate token` to create a PAT which has the above access encoded within it.

<img src="./docs/images/diagram16.png" alt="drawing" width="800"/>

---

## Store PAT for later use

Copy the PAT token and store in a file on your local machine; we'll use it later.

In the meantime, we're going to store in an environment variable.
```bash
export GITTOKEN=<PAT copied from GitHub>
```

Let's now use this token to create our own copies of the `dp01-src` and `dp01-ops` 
repositories.

---

## Clone repository to your local machine

We're going to use the contents of this repository to configure our cluster. To
do this we need to clone this repository to our local machine.

It's best practice to store cloned git repositories under a folder called `git`,
with subfolders that correspond to your projects.

Issue the following commands to create this folder structure and clone the
`dp01-ops` repository from GitHub to your local machine.

```bash
mkdir -p $HOME/git/$GITORG-tutorial
cd $HOME/git/$GITORG-tutorial
git clone https://$GITTOKEN@github.com/$GITORG/dp01-ops.git
cd dp01-ops
```

---

## Create DataPower development namespace

Let's use some YAML in `dp01-ops` to define two namespaces in our cluster:

Issue the following command:

```bash
oc apply -f setup/namespaces.yaml
```

which will confirm the following namespaces are created in the cluster:

```bash
namespace/dp01-ci created
namespace/dp01-dev created
```

As the tutorial proceeds, we'll see how the YAMLs in `dp01-ops` **fully** define
the DataPower related resources deployed to the cluster. In fact, we're going to
set up the cluster such that it is **automatically** updated whenever the `dp01-ops`
repository is updated. This concept is called **continuous deployment** and we'll
use ArgoCD to achieve it.

---

## Explore the `dp01-ops` repository

If you'd like to understand a little bit more about how the namespaces were
created, you can explore the contents of the `dp01-ops` repository.

Issue the following command:

```bash
cat setup/namespaces.yaml
```

which shows the following namespace definitions:

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: dp01-ci
  labels:
    name: dp01-ci
---
kind: Namespace
apiVersion: v1
metadata:
  name: dp01-dev
  labels:
    name: dp01-dev
```

Issue the following command to show these namespaces in the cluster

```bash
oc get namespace dp01-ci
oc get namespace dp01-dev
```

which will shows these namespaces and their age, for example:

```bash
NAME      STATUS   AGE
dp01-ci   Active   18s
NAME       STATUS   AGE
dp01-dev   Active   18s
```

During this tutorial, we'll see how:

- the `dp01-ci` namespace is used to store specific Kubernetes resources to
  build, package, version and test `dp01`.
- the `dp01-dev` namespace is used to store specific Kubernetes resources
  relating to a running DataPower virtual appliance, `dp01`.

---

## Create ArgoCD subscription

Let's install ArgoCD to enable continuous deployment.

Use the following command to create a subscription for ArgoCD:

```bash
oc apply -f setup/argocd-operator-sub.yaml
```

which will create a subscription for ArgoCD:

```bash
subscription.operators.coreos.com/openshift-gitops-operator created
```

Explore the subscription using the following command:

```bash
cat setup/argocd-operator-sub.yaml
```

which details the subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

See if you can understand each YAML node, referring to
[subscriptions](https://olm.operatorframework.io/docs/concepts/crds/subscription/)
if you need to learn more.

## Approve ArgoCD install plan

This subscription enables the cluster to keep up-to-date with new releases of
ArgoCD. Each release has an [install
plan](https://olm.operatorframework.io/docs/concepts/olm-architecture/) that is
used to maintain it. Our install plan requires manual approval; we'll see why a
little later.

Let's find our install plan and approve it.

Issue the following command:

```bash
oc get installplan -n openshift-operators | grep "openshift-gitops-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan `install-xxxxx` for ArgoCD.

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

ArgoCD will now install; this may take a few minutes.

## Verify ArgoCD installation

A
[ClusterServiceVersion](https://olm.operatorframework.io/docs/concepts/crds/clusterserviceversion/)
(CSV) is created for each release of the ArgoCD operator installed in the
cluster. This tells Operator Lifecycle Manager how to deploy and run the operator.

We can verify that the installation has completed successfully by examining the
CSV for ArgoCD.

Issue the following command:

```bash
oc get clusterserviceversion openshift-gitops-operator.v1.5.10 -n openshift-gitops
```

```bash
NAME                               DISPLAY                    VERSION   REPLACES                                          PHASE
openshift-gitops-operator.v1.5.10  Red Hat OpenShift GitOps   1.5.10    openshift-gitops-operator.v1.5.6-0.1664915551.p   Succeeded
```

See how the operator has been successfully installed at version 1.5.10.

Feel free to explore this CSV:

```bash
oc describe csv openshift-gitops-operator.v1.5.10 -n openshift-operators
```

The output provides an extensive amount of information not listed
here; feel free to examine it.

## Minor modifications to ArgoCD

ArgoCD will deploy `dp01` and its related resources to the cluster. These
resources are labelled by ArgoCD with a specific `applicationInstanceLabelKey`
so that they can be tracked for configuration drift. The default label used by ArgoCD collides with DataPower operator, so we need to change it.

Issue the following command to change the `applicationInstanceLabelKey`used by
ArgoCD:

```bash
oc patch argocd openshift-gitops  \
 --namespace openshift-gitops \
 --type merge \
 --patch '{"spec":{"applicationInstanceLabelKey":"argocd.argoproj.io/instance"}}'
```

which should respond with:

```bash
argocd.argoproj.io/openshift-gitops patched
```

which confirms that the ArgoCD operator has been patched and will now add this
label to every resource it deploys to the cluster.

---

## Role and role binding

ArgoCD requires permission to create resources in the `dp01-dev` namespace. We
use a role to define the resources required to deploy a DataPower virtual
appliance, and a role binding to associate this role with the `serviceaccount`
associated with ArgoCD.

Issue the following command to create this `role`:

```bash
oc apply -f setup/dp-role.yaml
```

which confirms that the `dp-deployer` role has been created:

```bash
role.rbac.authorization.k8s.io/dp-deployer created
```

Issue the following command to create the corresponding `rolebinding`:

```bash
oc apply -f setup/dp-rolebinding.yaml
```

which confirms that the `dp-deployer` role binding  has been created:

```bash
rolebinding.rbac.authorization.k8s.io/dp-deployer
```

We can see which resources ArgoCD can create in the cluster by examining the
`dp-deployer` role:

```bash
oc describe role dp-deployer -n dp01-dev
```

which returns:

```bash
Name:         dp-deployer
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources                            Non-Resource URLs  Resource Names  Verbs
  ---------                            -----------------  --------------  -----
  secrets                              []                 []              [*]
  services                             []                 []              [*]
  datapowerservices.datapower.ibm.com  []                 []              [*]
  ingresses.networking.k8s.io          []                 []              [*]
```

See how ArgoCD can now control `secrets`, `services`, `datapowerservices` and
`ingresses` with all operations such as create, read, update and delete (i.e.
`Verbs[*]`).

---

## Add IBM catalog sources

Like ArgoCD, there is a dedicated operator that manages DataPower virtual appliances in
the cluster. Unlike ArgoCD, its definition is held in the IBM [catalog
source](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/), so
we need to add this catalog source to the cluster before we can install it.

Issue the following command:

```bash
oc apply -f setup/catalog-sources.yaml
```

which will add the catalog sources defined in this YAML to the cluster:

```bash
catalogsource.operators.coreos.com/opencloud-operators created
catalogsource.operators.coreos.com/ibm-operator-catalog created
```

Notice, that there actually **two** new catalog sources added; feel free to
examine the catalog source YAML:

```bash
cat setup/catalog-sources.yaml
```

which shows you the detailed YAML for these catalog sources:

```bash
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: opencloud-operators
  namespace: openshift-marketplace
spec:
  displayName: IBMCS Operators
  publisher: IBM
  sourceType: grpc
  image: docker.io/ibmcom/ibm-common-service-catalog:latest
  updateStrategy:
    registryPoll:
      interval: 45m
---
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: IBM Operator Catalog
  image: 'icr.io/cpopen/ibm-operator-catalog:latest'
  publisher: IBM
  sourceType: grpc
  updateStrategy:
    registryPoll:
      interval: 45m
```

Examine these YAMLs to see if you can understand [how they
work](https://olm.operatorframework.io/docs/concepts/crds/catalogsource/).

---

## Install DataPower operator

We can now install the DataPower operator; using the same process as we used with ArgoCD.

Issue the following command:

```bash
oc apply -f setup/dp-operator-sub.yaml
```

which will create the DataPower operator subscription:

```bash
subscription.operators.coreos.com/datapower-operator created
```

Explore the subscription using the following command:

```bash
cat setup/dp-operator-sub.yaml
```

which details the subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/datapower-operator.dp01-ns: ''
  name: datapower-operator
  namespace: openshift-operators
spec:
  channel: v1.6
  installPlanApproval: Manual
  name: datapower-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
  startingCSV: datapower-operator.v1.6.8
```

Notice how this operator is installed in the `openshift-operators` namespace.
Note also the use of `channel` and `startingCSV` to be precise about the exact
version of the DataPower operator to be installed.

## Approve and verify DataPower install plan

Let's find our DataPower install plan and approve it.

```bash
oc get installplan -n openshift-operators | grep "datapower-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan:

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

where `install-xxxxx` is the name of the DataPower install plan.

Again, feel free to verify the DataPower installation with the following
commands:

```bash
oc get clusterserviceversion datapower-operator.v1.6.8 -n openshift-operators
```

```bash
NAME                                     DISPLAY                       VERSION   REPLACES                                          PHASE
datapower-operator.v1.6.8                IBM DataPower Gateway         1.6.8                                                       Succeeded
```

which shows that the 1.6.8 version of the operator has been successfully installed.

```bash
oc describe csv datapower-operator.v1.6.8 -n openshift-operators
```

The output provides an extensive amount of information not listed
here; feel free to examine it.

---

## Install Tekton pipelines

Our final task is to install Tekton.  With it, we can create pipelines that
populate the operational repository `dp01-ops` using the DataPower configuration
and development artifacts stored in `dp01-src`. Once populated by Tekton, ArgoCD
will then synchronize these artifacts with the cluster to ensure the cluster is
running the most up-to-date version of `dp01`.

Issue the following command to create a subscription for Tekton:

```bash
oc apply -f setup/tekton-operator-sub.yaml
```

which will create a subscription:

```bash
subscription.operators.coreos.com/openshift-pipelines-operator created
```

Again, this subscription enables the cluster to keep up-to-date with new version
of Tekton.

Explore the subscription using the following command:

```bash
cat setup/tekton-operator-sub.yaml
```

which details the subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel:  stable
  installPlanApproval: Manual
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Manual Tekton install:
```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.16.3/release.yaml)
```
---

## Approve and verify Tekton install plan

Let's find our install plan and approve it.

```bash
oc get installplan -n openshift-operators | grep "openshift-pipelines-operator" | awk '{print $1}' | \
xargs oc patch installplan \
 --namespace openshift-operators \
 --type merge \
 --patch '{"spec":{"approved":true}}'
```

which will approve the install plan

```bash
installplan.operators.coreos.com/install-xxxxx patched
```

where `install-xxxxx` is the name of the Tekton install plan.

Again, feel free to verify the Tekton installation with the following commands:

```bash
oc get clusterserviceversion -n openshift-operators
```

(replacing `x.y.z` with the installed version of Tekton)

```bash
oc describe csv openshift-pipelines-operator-rh.vx.y.z -n openshift-operators
```

---

## Create a secret to contain the PAT for use by Tekton.

The PAT we created earlier is now stored as a secret in the `dp01-ci` namespace and used by the pipeline whenever it needs to access the `dp01-src` and `dp01-ops` repositories.

Issue the following command to create a secret containing the PAT:

```bash
export GITCONFIG=$(printf "[credential \"https://github.com\"]\n  helper = store")
oc create secret generic dp01-git-credentials -n dp01-ci \
  --from-literal=.gitconfig="$GITCONFIG" \
  --from-literal=.git-credentials="https://$GITUSER:$GITTOKEN@github.com" \
  --type=Opaque \
  --dry-run=client -o yaml > .ssh/dp01-git-credentials.yaml
```

Issue the following command to create this secret in the cluster:

```bash
oc apply -f .ssh/dp01-git-credentials.yaml
```

Finally, add this secret to the `pipeline` service account to allow it to use
`dp-1-ssh-credentials` secret to access GitHub.

```bash
oc patch serviceaccount pipeline \
    --namespace dp01-ci \
    --type merge \
    --patch '{"secrets":[{"name":"dp01-git-credentials"}]}'
```

---

## Image registry

Allow Tekton to write to image registry

```bash
// oc adm policy  add-cluster-role-to-user edit system:serviceaccount:dp01-ci:pipeline // not sure we need this?
oc policy add-role-to-user system:image-puller system:serviceaccount:dp01-dev:dp01-datapower-pod-service-account --namespace=dp01-ci
```
---

## An ArgoCD application to manage `dp01`

Finally, we're going to create an ArgoCD application to manage the virtual
appliance `dp01`. The YAMLs for `dp01` will be created by its Tekton pipeline in
`dp01-ops`. Every time this repository is updated, our ArgoCD application will
ensure that the latest version of `dp01` is deployed to the cluster.

Let's have a quick look at our ArgoCD application.

Issue the following command:

```bash
cat environments/dev/argocd/dp01.yaml
```

which will show its YAML:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dp01-argo
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "100"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    namespace: dp01-dev
    server: https://kubernetes.default.svc
  project: default
  source:
    path: environments/dev/dp01/
    repoURL: https://github.com/$GITORG/dp01-ops.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - Replace=true
```

Notice how the Argo application monitors a specific GitHub location for
resources to deploy to the cluster:

```yaml
  source:
    path: environments/dev/dp01/
    repoURL: https://github.com/$GITORG/dp01-ops.git
    targetRevision: main
```

See how:
  - `repoURL: https://github.com/$GITORG/dp01-ops.git` identifies the
    repository where the YAMLs are located ($GITORG will be replaced with your GitHub organisation)
  - `targetRevision: main` identifies the branch within the repository
  - `path: environments/dev/dp01/` identifies the folder within the repository

## Deploy `dp01-argo` to the cluster

Let's deploy this ArgoCD application to the cluster. We use the `envsubst`
command to replace $GITORG with your GitHub organisation.

Issue the following command:

```bash
envsubst < environments/dev/argocd/dp01.yaml > environments/dev/argocd/dp01.yaml
oc apply -f environments/dev/argocd/dp01.yaml
```

which will complete with:

```bash
application.argoproj.io/dp01-argo created
```

We now have an ArgoCD application monitoring our repository.

## View `dp01-argo` in ArgoCD UI

We can use the ArgoCD UI to look at the `dp01-argo` application and the
resources it is managing:

Issue the following command to identify the URL for the ArgoCD login page:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{"https://"}{.spec.host}{"\n"}'
```

which will return a URL similar to this:

```bash
https://openshift-gitops-server-openshift-gitops.apps.sno-ajo-1.snoajo1.com
```

We will use this URL to log into the ArgoCD admin console to view our deployments. 

Issue the following command to determine the ArgoCD `password` for the `admin`
user:

```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --keys="admin.password" --to=-
```

Login to ArgoCD with `admin` and `password`.

> **Note**<br>
> 
> If DNS name resolution has not been set up for your cluster hostname, you will need
> to add the URL hostname to your local machine  `/etc/hosts` file to identify the IP address of the ArgoCD server, e.g.:
>
> `141.125.162.227 openshift-gitops-server-openshift-gitops.apps.sno-ajo-1.snoajo1.com`

Upon successful login, you will see the following screen:

![diagram4](./docs/images/diagram4.png)

Notice how the ArgoCD application `dp01-argo` is monitoring the
`https://github.com/dporg-odowdaibm/dp01-ops` repository for YAMLs in the
`environments/dev/dp01` folder.

In the next step we will run the Tekton pipeline that populates this repository
folder with the YAMLs for the `dp01` virtual appliance.

---

## Congratulations

You've configured your cluster for DataPower. Let's run a pipeline to populate
the `dp01-ops` repository. This pipeline is held in the source repository
`dp01-src`; it also holds the configuration for the `dp01` virtual DataPower
appliance.

Continue [here](https://github.com/datapower-virtual-appliance-demo/dpxx-src#introduction) to fork
your copy of the `dp01-src` repository.

---
