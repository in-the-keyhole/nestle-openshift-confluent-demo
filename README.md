# Nestle Openshift | Confluent Kafka configuration

This repository contains a [VS Code Remote Container](https://code.visualstudio.com/docs/remote/containers) configuration that provides the following:

* Openshift CLI
* Helm CLI
* Flux CLI

To use it, follow the [Getting Started](https://code.visualstudio.com/docs/remote/containers#_getting-started) guide, and then you must provide a .devcontainer/.env file that sets the following environment variables:
> A template has been provided in .devcontainer/.env-template that you can copy and modify
* OPENSHIFT_SERVER - The OpenShift server URL 
* OPENSHIFT_USER - The Openshift username with admin priveleges
* OPENSHIFT_PASSWORD - The password for the Openshift user
* GITHUB_ORGANIZATION - A valid GitHub organization
* GITHUB_TOKEN - A valid GitHub personal access token (PAT)
* FLUX_NS=flux-system - The namespace in which to install Flux

### Authenticate with Openshift
```
oc login --server=$OPENSHIFT_SERVER --username=$OPENSHIFT_USER --password=$OPENSHIFT_PASSWORD
```

### Install Flux via OperatorHub
Browse to OperatorHub in the Openshift console - find Flux, and choose Install accepting defaults

> Aside on Operators

Operators allow for "extensions" of the core k8s capability.
Replacing a "human operator" they manage applications, automatically ensuring that the current state matches the desired state (control loop).
They use "custom resource definitions" (CRDs) to manage application components.

### Bootstrap Flux
This is the ONLY manual command you will need to apply to manage workloads in Openshift/k8s.  It *bootstraps* the GitOps engine (FluxCD), by creating a [GitRepository](https://fluxcd.io/docs/components/source/gitrepositories/) resource containing the resources that need to be provisioned/maintained in OpenShift.

```
flux bootstrap github \
 --owner=$GITHUB_ORGANIZATION \
 --repository=nestle-openshift-confluent-demo \
 --path=clusters/production \
 --components-extra=image-reflector-controller,image-automation-controller \
 --read-write-key
```

### Control Center Dashboard

To expose network access to the Confluent Control Center, you can either:

* Forward port for Control Center Dashboard
```
oc port-forward controlcenter-0 9021:9021
```
> You can then browse to http://localhost:9021

* Create a route to controlcenter service on port 9021
```
oc create route edge --service controlcenter -n confluent
```
> This will create a permanant route URL that can be used/bookmarked etc.

### Review the producer-example application deployed in the nestle namespace
The GitOps defined resources in apps/production create a namspace named *nestle*, and within it, a deployment of an application named *producer_example* that includes the provisioning of a *KafkaTopic* in Confluent. The application produces messages on a timer, pushing then onto a topic named 'producer-0'