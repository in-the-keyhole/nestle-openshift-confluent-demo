# Nestle Openshift | Confluent Kafka configuration

## Authenticate with Openshift
```
oc login --server=$OPENSHIFT_SERVER --username=$OPENSHIFT_USER --password=$OPENSHIFT_PASSWORD
```

## Install Flux 

>Install with CLI - https://fluxcd.io/docs/use-cases/openshift/#flux-installation-with-cli

Set policy for flux to run as non-root
```
oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:kustomize-controller

oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:helm-controller

oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:source-controller

oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:notification-controller

oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:image-automation-controller

oc adm policy add-scc-to-user nonroot system:serviceaccount:$FLUX_NS:image-reflector-controller
```

Bootstrap Flux

```
flux bootstrap github \
  --owner=$GITHUB_ORGANIZATION \
  --repository=nestle-openshift-confluent-demo \
  --path=clusters/production
```
> This will create and seed the gitops repo on github

## Install Confluent Operator (Manually)

### Create an Openshift project (namespace)

```
oc new-project confluent --display-name=Confluent`
```

### Add the Confluent Helm repo

```
helm repo add confluentinc https://packages.confluent.io/helm

helm repo update
```

#### Operators

> Operators allow for "extensions" of the core k8s capability
> Replacing a "human operator" they manage applications, automatically ensuring that the current state matches the desired state (control loop)
> They use "custom resource definitions" (CRDs) to manage application components

> For example, the Confluent Operator plays the role of the Managment Control Plane, ensuring that the desired configuration is achieved

#### SCC

> Security context constraints (SCC) that control the actions that a pod can perform and what it has the ability to access.
> See https://docs.openshift.com/container-platform/3.11/architecture/additional_concepts/authorization.html#security-context-constraints

### Use Helm to install the Operator
>Disable the custom pod security context, use the default context

```
helm upgrade --install confluent-operator confluentinc/confluent-for-kubernetes \ 
  --set podSecurity.enabled=false --namespace confluent
```

> wait for the Operator pod to be running (via oc get pods -w)

### Stand up the desired Confluent services
> The Operator must be running
```
oc apply -f samples/confluent-platform-with-defaultSCC.yaml
```

> wait for all pods to be running (via oc get pods -w)

### Expose Control Center Dashboard

> Forward port for Control Center Dashboard

> Can also create a route to control center service on port 9021
```
oc port-forward controlcenter-0 9021:9021
```

### Test installation using demo producer app
```
oc apply -f samples/producer-app-data.yaml`
```