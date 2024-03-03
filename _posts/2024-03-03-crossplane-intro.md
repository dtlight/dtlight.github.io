---
layout: post
title: Using Crossplane to provision a GKE cluster
subtitle: A basic example
cover-img: /assets/img/abstract-geometrical-shapes.jpg
thumbnail-img: /assets/img/cloud-icons.jpg
gh-repo: dtlight/crossplane-gke-demo
gh-badge: [star, fork, follow]
tags: [crossplane, GCP, GKE]
comments: false
author: David Light
---

I've previously written about provisioning infrastructure using terraform so I will first do a brief comparison to crossplane. A major difference between the two tools is the pull vs push approach. Crossplane leverages the kubernetes api using a control loop mechanism that allows it to monitor the state of infrastructure and bring it into sync if configuration drift is found. It proactively probes the state of infrastructure every hour by default (this value can be changed). Terraform on the other hand updates its understanding of the state of the infrastructure when told to do so in response to an event (such as a terraform plan command). If after infrastructure changes are applied, configuration drift occurs, this will not be captured until the next 'push event'.

## Getting Started

{: .box-note}
You'll need to install helm as well as some form of kubernetes (minikube, microk8s etc) and kubectl.  You'll also need Google Cloud SDK.

```
$ helm repo add crossplane-stable https://charts.crossplane.io/stable 
$ helm install crossplane \
--namespace crossplane-system \
--create-namespace \
--version 1.15.0 \
crossplane-stable/crossplane
```

Resulting with:
```
NAME: crossplane
LAST DEPLOYED: Sat Mar  2 17:04:38 2024
NAMESPACE: crossplane-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Release: crossplane

Chart Name: crossplane
Chart Description: Crossplane is an open source Kubernetes add-on that enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume.
Chart Version: 1.15.0
Chart Application Version: 1.15.0

Kube Version: v1.28.3
```

Let's verify Crossplane is up and running:
```
$ kubectl get pods -n crossplane-system
```

There are some init containers that will install custom resource definitions in the cluster:
```
$ kubectl get crds | grep crossplane.io
```

Once the project is created in GCP, billing is enabled and the SDK is configured on your machine, configure the following environment variables:

```
# set this to the PROJECT_ID
export GCP_PROJECT=crossplane-demo-416016
# set this to whatever you want the service account name to be
export SA=crossplane-demo  
export GCP_SVC_ACCT="$SA@$GCP_PROJECT.iam.gserviceaccount.com"  
# set this to the path where you want to store Crossplane's authentication key file (home directory by default). Otherwise, you can specify a path and filename
export KEY_FILE="$SA-keyfile.json"         
# set this to the namespace in your **Crossplane Kubernetes Cluster** 
export PROVIDER_SECRET_NAMESPACE=crossplane-system
```

We need to create a Service Account in order to allow Crossplane to communicate with the GKE cluster. The Service Account must be created with the following Roles: 
- Computer Network Admin
- Kubernetes Engine Admin
- Service Account User

To do this we need to enable these APIs:

```
$ gcloud services enable --project $GCP_PROJECT \
  compute.googleapis.com \
  container.googleapis.com \
  servicenetworking.googleapis.com
  
  Operation "operations/acf.p2-XXXXXX-XXXX-XXXXXXXX" finished successfully.
```

#### Now we need to create a service account for Crossplane...

```
$ gcloud iam service-accounts create --project $GCP_PROJECT $SA

Created service account [crossplane-demo]
```
...and then download a JSON key for the account so Crossplane can authenticate with it:

```
$ gcloud iam service-accounts keys create --iam-account $GCP_SVC_ACCT --project $GCP_PROJECT $KEY_FILE

created key [XXXXXXXXXXXXXXXXXXXXXXX] of type [json] as [crossplane-demo-keyfile.json] for [crossplane-demo@crossplane-demo-416016.iam.gserviceaccount.com]
```

Now to store this key in a secret in Crossplane’s Kubernetes namespace so that Crossplane can use the key to authenticate to GCP. In authentication.yaml the following can be  applied:

First convert the key file into base64:
```
$ export BASE64KEYFILE=$(base64 crossplane-demo-keyfile.json | tr -d "\n")
```

```
cat > authentication.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gcp-account-creds
  namespace: ${PROVIDER_SECRET_NAMESPACE}
type: Opaque
data:
  credentials: ${BASE64KEYFILE}
EOF
```

```
$ kubectl apply -f authentication.yaml

secret/gcp-account-creds created
```

#### Grant the service account the permissions to do its work:

```
$ gcloud projects add-iam-policy-binding $GCP_PROJECT --member "serviceAccount:$GCP_SVC_ACCT" --role="roles/iam.serviceAccountUser"
$ gcloud projects add-iam-policy-binding $GCP_PROJECT --member "serviceAccount:$GCP_SVC_ACCT" --role="roles/container.admin"
$ gcloud projects add-iam-policy-binding $GCP_PROJECT --member "serviceAccount:$GCP_SVC_ACCT" --role="roles/compute.networkAdmin"
```
#### Set up the GCP provider
First to install the GCP provider:

```
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-gcp:v0.22.0
```

Then we need to configure the Crossplane GCP provider to tell it where to get the authentication information we created. Information about this provider can be found using [marketplace.upbound.io/providers](https://marketplace.upbound.io/providers/crossplane-contrib/provider-gcp/v0.22.0).

```
cat > provider.yaml <<EOF
apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: crossplane-provider-gcp
spec:
  projectID: ${GCP_PROJECT}
  credentials:
    source: Secret
    secretRef:
      namespace: ${PROVIDER_SECRET_NAMESPACE}
      name: gcp-account-creds
      key: credentials
EOF
```

We can check the credentials in the ProviderConfig spec are as expected by running

```
$ kubectl describe providerconfig.gcp.upbound.io/crossplane-provider-gcp  -n crossplane-system
```

to which we should see something along the lines of:

```
  Credentials:
    Impersonate Service Account:
      Name:  
    Secret Ref:
      Key:        credentials
      Name:       gcp-account-creds
      Namespace:  crossplane-system
    Source:       Secret
```

## Setting Up A Network and Cluster
Before we build a cluster, we first need to define the network the cluster will use. We need a network and a subnet. 

```
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  name: crossplane-demo-network
spec:
  forProvider:
    description: 'This is a network built by crossplane' 
    routingConfig:
      routingMode: 'GLOBAL'
  providerConfigRef:
    name: crossplane-provider-gcp
```
```
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Subnetwork
metadata:
  name: crossplane-demo-subnet
spec:
  forProvider:
    ipCidrRange: 10.154.0.0/20
    privateIpGoogleAccess: true
    networkRef: 
      # ensure this matches the network name you defined in Network
      name: crossplane-demo-network
    region: europe-west2
  providerConfigRef:
    name: crossplane-provider-gcp

```

Add both to the same or different files and apply.

Cluster and Node pool creation

```
apiVersion: container.gcp.crossplane.io/v1beta2
kind: Cluster
metadata:
  name: example-cluster
spec:
  forProvider:
    addonsConfig:
      httpLoadBalancing:
    location: europe-west2
    network: "crossplane-demo-network" 
    networkPolicy:
      enabled: true
      provider: CALICO
    subnetwork: crossplane-demo-subnet
  providerConfigRef:
    name: crossplane-provider-gcp
```
```
apiVersion: container.gcp.crossplane.io/v1beta1
kind: NodePool
metadata:
  name: crossplane-nodepool
spec:
  forProvider:
    clusterRef:
      name: example-cluster
    config:
      machineType: n1-standard-1
    locations:
      - "europe-west2"
  providerConfigRef:
    name: crossplane-provider-gcp

```

Lastly to check on the status of the cluster
```
$ kubectl get cluster

NAME              READY   SYNCED   STATE          ENDPOINT         LOCATION       AGE
example-cluster   False   True     PROVISIONING   34.147.184.108   europe-west2   100s
```
After some time (roughly 6 minutes in my case) the cluster is created
```
$ kubectl get cluster

NAME              READY   SYNCED   STATE          ENDPOINT         LOCATION       AGE
example-cluster   True    True     RUNNING   34.147.184.108   europe-west2   6m37s
```

Don't forget to delete the cluster (it may take some time and your terminal will appear to hang).

```
$ kubectl delete cluster example-cluster

cluster.container.gcp.crossplane.io "example-cluster" deleted
```
