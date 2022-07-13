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


### Steps to Onboard a new team

1. Setup (or have the team setup) a GitOps repo that will contain their k8s manifests, kustomizations, and/or Helm releases

2. Create a new manifest under /clusters/production/teams folder _(an existing file in that folder can be copied and modified)_

    The file will need to contain the following:

    - A Sealed Secret to be used for git access to the team GitOps repository by Flux, To create a Sealed Secrete, follow these steps:
    
        1. Create git secret for the teams gitops repository, use:
            ```
            flux create secret git teamx-git \
                --namespace production \
                --url=ssh://git@github.com/in-the-keyhole/gitops-teamx \
                --export > temp-git-secret.yaml
            ```
    
        2. Copy the geneated deployKey from the temp-git-secret.yaml and add it to the git repo

        3. Generate a Sealed Secret by pressing Cmd+Shft+P and choosing >Seal Kubernetes Secret - File, when prompted:

            - choose 'strict'
            - give the secret a name
            - use the tls.crt from kubesystem/secrets/sealed-secrets-keyXXXX

        4. Copy the generated SealedSecret resource into the manifest file created in Step 1. (replace the existing one if copied from another file)
    
        5. Delete temp-git-secret.yaml

    - A GitRepository resource pointed to the teams gitops repo and using the secret name created above

    - A Kustomization for production, staging, develop

    - Optionally, an ImageUpdateAutomation resource



    #### Where I left off --

    Established an ImageUpdateAutomation plan for Production - which automatically creates a branch in the Team GitOps repo with the updated image tag.

    Things to do:

    1. Git a 'develop' and/or 'staging' environment setup
        - [x] Can use same cluster for now, but will have to configure the Kustomizations accordingly 
        - [x] Setup ImageUpdateAutomation based upon develop tags (or release candidate for staging) that update GitOps main branch automatically (not a branch/PR like prod)
        - [x] Use Now Playing movie-service 
        - [] Create a build development pipeline (Tekton and/or GitHub Action?) - initially can just create image tags manually to simulate build

    2. Ultimately need a PR opened automatically on image update - this could be done with GitHub Actions using this: https://fluxcd.io/docs/use-cases/gh-actions-auto-pr/
        or other similar approach using associated pipeline tech (i.e. Azure Pipeline on Azure Repo)

    3. Create a demo for a Trunk-based development workflow
        - [] Including feature flags