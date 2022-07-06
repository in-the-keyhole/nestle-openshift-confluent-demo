# Nestle Openshift | Confluent Kafka configuration

## Authenticate with Openshift
```
oc login --server=$OPENSHIFT_SERVER --username=$OPENSHIFT_USER --password=$OPENSHIFT_PASSWORD
```

### Install Flux via OperatorHub
Browse to OperatorHub in the Openshift console - find Flux, and choose Install accepting defaults

#### Operators

> Operators allow for "extensions" of the core k8s capability
> Replacing a "human operator" they manage applications, automatically ensuring that the current state matches the desired state (control loop)
> They use "custom resource definitions" (CRDs) to manage application components

> For example, the Confluent Operator plays the role of the Managment Control Plane, ensuring that the desired configuration is achieved


### Bootstrap Flux
```
oc apply -f bootstrap/bootstrap.yaml
```

This will create a GitRepository pointed to the main branch of this repository, and a Kustomization that will automate deployment of clusters/production.  This will in turn, deploy resources defined by operators, followed by infrastructure, and finally by apps.
### Expose Control Center Dashboard

> Forward port for Control Center Dashboard

> Can also create a route to control center service on port 9021
```
oc port-forward controlcenter-0 9021:9021
```

### Review the producer-example application deployed in the nestle namespace - this produces messages on a topic named 'producer-9'