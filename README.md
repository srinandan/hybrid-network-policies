# Apigee hybrid Network Policies

This sample provides a set of instructions to secure an Apigee hybrid installation through network policies. 

## Background

This sample uses:

* [Apigee hybrid](https://docs.apigee.com/hybrid/reference-overview)
* [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) to setup/install Apigee hybrid 
* [Project Calico](https://docs.projectcalico.org/v3.5/introduction/) as the CNI plugin

## Objective

Kubernetes network policies allow/control how pods communicate with each other (as well as how/if they can communicate externally/outside the cluster). A Kubernetes cluster can host one or more Apigee orgs and each org can have one or more Apigee environments. Pods installed by Apigee hybrid are cluster scoped, org scoped or environment scoped.

The following components are cluster scoped:

* apigee-deployment-admissionhook - Validates the ApigeeDeployment config before persisting it in Kubernetes
* apigee-deployment-controller - Custom controller that creates and updates Kubernetes and Istio resources
* apigee-metrics - Apigee Metrics Agent (sends application metrics to Stackdriver)
* apigee-logger - Apigee Logger Agent (sends applications logs to Stackdriver)
* istio-ingressgateway - Ingress proxy to all API gateways in the cluster
* apigee-mart-istio-ingressgateway - Ingress proxy to all MART applications in the cluster; NOTE: When using apigee-connect this component is not necessary.

Apigee Cassandra can host multiple orgs and there can be multiple instances per cluster:

* apigee-cassandra - API Gateway persistence layer

The following components are org scoped:

* apigee-mart - Apigee Administrative API Endpoint
* apigee-connect-agent - Apigee Connect Agent to communicate with the control plane

The following components are environment scoped:

* apigee-runtime - Apigee API Gateway
* apigee-udca - Apigee Analytics Agent (sends analytics to the control plane)
* apigee-synchronizer - Apigee Agent to sync configuration between the control plane and runtime (data) plane

The objective of applying network policies is to restrict which of these pods can communicate with other pods.

## Installation Topology

This installation assumes:

1. A single Apigee org
  a. Change the org name in this [kustomization.yaml](./overlays/sample1/org/kustomization.yaml#L7) file.
2. A single Apigee environment in that org
  a. Change the environment name in this [kustomization.yaml](./overlays/sample1/org/envs/test/kustomization.yaml#L7) file.
3. The product is installed on default namespaces:
  a. `apigee-system` - for the Apigee deployment controller and admissionhooks
  b. `apigee` - for all Apigee runtime components
  c. `istio-system` - for all Istio components

## Prequisites

This sample uses [Kustomize](https://kustomize.io/) and tested with v3.5.3. Kustomize can be used standalone or as part of kubectl (1.14+). Since Apigee hybrid is supported on 1.13.x, this sample will use Kubstomize standalone. 

### Apigee hybrid

This sample has been tested with Apigee hybrid 1.1.x. The sample will require changes to work with version 1.0.X.

## Explanation of policies
 
The following policies allow access to kube DNS and Google DNS:

* [netpol-allow-dns-cassandra](./base/common/netpol-dns.yaml)
* [netpol-allow-dns-mart](./base/common/netpol-dns.yaml)
* [netpol-allow-dns-runtime](./base/common/netpol-dns.yaml)
* [netpol-allow-dns-sync](./base/common/netpol-dns.yaml)
* [netpol-allow-dns-udca](./base/common/netpol-dns.yaml)
* [netpol-allow-dns-metrics](./base/common/netpol-dns.yaml)

NOTE: These DNS policies allow each pod access to the internet (Google DNS) with the following stanza:

```yaml

- ipBlock:
    cidr: 8.8.8.8/32
```

Remove this block if you don't want a pod to access the internet.

The following policies secure Apigee Cassandra:

* [netpol-cassandra-mart](./base/common/netpol-cassandra-client.yaml): Allows access from MART to Cassandra
* [netpol-cassandra-runtime](./base/common/netpol-cassandra-client.yaml): Allows access from Runtime to Cassandra
* [netpol-cassandra-server](./base/common/netpol-cassandra-server.yaml): Allows Cassandra nodes to communicate with each other
* [netpol-cassandra-monitor](./base/common/netpol-cassandra-monitoring.yaml): Allow only the metrics pod to scrape metrics

The following policies secure Apigee MART:

* [netpol-mart-ingress](./base/common/netpol-ingress.yaml): Allows MART to be accessed by mart-istio-ingressgateway only. 
* [netpol-mart-authz](./base/netpol-mart.yaml): Allow MART authn-authz to access the internet (apigee.googleapis.com)
* [netpol-mart-connect](./overlays/sample1/org/netpol-connect.yaml): Allow Apigee connect access to a MART instance
* [netpol-mart-metrics](./overlays/sample1/org/netpol-mart.yaml): Allow only the metrics pod to scrape metrics

The following policies secure Apigee Runtime (Gateway):

* [netpol-runtime-metrics](./base/common/netpol-runtime.yaml): Allow only the metrics pod to scrape metrics
* [netpol-runtime-ingress](./base/common/netpol-ingress.yaml): Allows the Apigee runtime to be accessed by istio-ingressgateway only

NOTE: The runtime itself does not have other policies. We cannot predict by default if the gateway accesses services inside the cluster and inside and outside the cluster.

The following policies secure Apigee synchronizer:

* [netpol-sync-authz](./overlays/sample1/org/envs/test/netpol-sync.yaml): Allow the sync to reach the control plane (apigee.googleapis.com)
* [netpol-sync-runtime](./overlays/sample1/org/envs/test/netpol-sync.yaml): Allow only the runtime to access the synchronizer (for proxy bundles etc.)
* [netpol-sync-metrics](./overlays/sample1/org/envs/test/netpol-sync.yaml): Allow only the metrics pod to scrape metrics

The following policies secure Apigee UDCA (Analytics Agent):

* [netpol-udca-authz](./overlays/sample1/org/envs/test/netpol-udca.yaml): Allow UDCA access the control plane to send analytics
* [netpol-udca-runtime](./overlays/sample1/org/envs/test/netpol-udca.yaml): Allow the local AX endpoint to be accessed to Runtime only
* [netpol-udca-token](./overlays/sample1/org/envs/test/netpol-udca.yaml): Allow UDCA access to the token endpoint

The following policies secure Apigee Metrics/Monitoring:

* [netpol-monitoring-cassandra](./base/common/netpol-monitoring.yaml): Allow monitoring of Cassandra by the metrics pod
* [netpol-monitoring-runtime](./base/common/netpol-monitoring.yaml): Allow monitoring of Runtime by the metrics pod
* [netpol-monitoring-mart](./base/common/netpol-monitoring.yaml): Allow monitoring of MART by the metrics pod
* [netpol-monitoring-sync](./base/common/netpol-monitoring.yaml): Allow monitoring of Sync by the metrics pod

## Installing the Policies

Step 1: Label namespaces, these will come in handy for match selectors

```bash

kubectl label namespace apigee app=apigee
kubectl label namespace apigee-system app=apigee-system
kubectl label namespace istio-system app=istio-system
kubectl label namespace kube-system namespace=kube-system
```

Step 2: Add org name and environment(s)

  a. Update your Apigee org name [here](./overlays/sample1/org/kustomization.yaml#L7)
  b. For each environment, create a new folder in `./overlays/org/envs` with all the files and change the environment name
  c. Change the environment name [here](./overlays/sample1/orgs/envs/test/kustomization.yaml#L7)

Step 3: Apply Network policies

```bash

kustomize build overlays/sample1 | kubectl apply -f -
```

## Validation

The following network policies should be created in the `apigee` namespace

```bash

NAME                                 POD-SELECTOR                                             AGE
netpol-connect-control-plane         app=apigee-connect-agent,org=srinandans-apigee           13s
netpol-connect-mart                  app=apigee-connect-agent,org=srinandans-apigee           13s
netpol-connect-metrics               app=apigee-connect-agent,org=srinandans-apigee           13s
netpol-mart-authz                    app=apigee-mart,org=srinandans-apigee                    12s
netpol-mart-connect                  app=apigee-mart,org=srinandans-apigee                    12s
netpol-mart-metrics                  app=apigee-mart,org=srinandans-apigee                    12s
netpol-netpol-allow-dns-cassandra    app=apigee-cassandra                                     12s
netpol-netpol-allow-dns-connect      app=apigee-connect-agent                                 12s
netpol-netpol-allow-dns-mart         app=apigee-mart                                          12s
netpol-netpol-allow-dns-metrics      app=apigee-metrics                                       12s
netpol-netpol-allow-dns-runtime      app=apigee-runtime                                       11s
netpol-netpol-allow-dns-sync         app=apigee-synchronizer                                  11s
netpol-netpol-allow-dns-udca         app=apigee-udca                                          11s
netpol-netpol-cassandra-mart         app=apigee-cassandra                                     11s
netpol-netpol-cassandra-monitor      app=apigee-cassandra                                     11s
netpol-netpol-cassandra-runtime      app=apigee-cassandra                                     11s
netpol-netpol-cassandra-server       app=apigee-cassandra                                     10s
netpol-netpol-mart-ingress           app=apigee-mart                                          10s
netpol-netpol-monitoring-cassandra   app=apigee-metrics                                       10s
netpol-netpol-monitoring-connect     app=apigee-metrics                                       10s
netpol-netpol-monitoring-mart        app=apigee-metrics                                       10s
netpol-netpol-monitoring-runtime     app=apigee-metrics                                       10s
netpol-netpol-monitoring-sync        app=apigee-metrics                                       10s
netpol-netpol-monitoring-udca        app=apigee-metrics                                       9s
netpol-netpol-runtime-ingress        app=apigee-runtime                                       9s
netpol-netpol-runtime-metrics        app=apigee-runtime                                       9s
netpol-sync-authz                    app=apigee-synchronizer,env=test,org=srinandans-apigee   9s
netpol-sync-metrics                  app=apigee-synchronizer,env=test,org=srinandans-apigee   9s
netpol-sync-runtime                  app=apigee-synchronizer,env=test,org=srinandans-apigee   9s
netpol-udca-authz                    app=apigee-udca,env=test,org=srinandans-apigee           9s
netpol-udca-metrics                  app=apigee-udca,env=test,org=srinandans-apigee           8s
netpol-udca-runtime                  app=apigee-udca,env=test,org=srinandans-apigee           8s
```

NOTE: The org and env names should match your org and env names

The following network policies should be created in the `apigee` namespace

```bash

NAME                  POD-SELECTOR                          AGE
netpol-ad             app=apigee-deployment-admissionhook
netpol-allow-dns-ah   app=apigee-deployment-admissionhook
```

### Testing the policies

Step 1: Add a test/sample container with cURL installed
Step 2: Exec into the pod and access shell `kubectl exec -it {podname} -n {namespace} sh

#### Before Network policies

Step 3: Use cURL to access a pod (even if it is not an http endpoint)

```bash

~ $ curl apigee-cassandra.apigee.svc.cluster.local:7000 -v
* Rebuilt URL to: apigee-cassandra.apigee.svc.cluster.local:7000/
*   Trying 10.56.1.3...
* TCP_NODELAY set
* connect to 10.56.1.3 port 7000 failed: Connection refused
*   Trying 10.56.2.2...
* TCP_NODELAY set
* connect to 10.56.2.2 port 7000 failed: Connection refused
*   Trying 10.56.3.2...
* TCP_NODELAY set
* connect to 10.56.3.2 port 7000 failed: Connection refused
* Failed to connect to apigee-cassandra.apigee.svc.cluster.local port 7000: Connection refused
* Closing connection 0
curl: (7) Failed to connect to apigee-cassandra.apigee.svc.cluster.local port 7000: Connection refused
```

You'll notice the connetion goes through (but rejected by the server since it is not a http endpoint)

#### After Network policies

Step 4: Use cURL again to access a pod

```bash

~ $ curl apigee-cassandra.apigee.svc.cluster.local:7000 -v
* Rebuilt URL to: apigee-cassandra.apigee.svc.cluster.local:7000/
*   Trying 10.56.1.3...
* TCP_NODELAY set
^C
```

The network connection does not complete. Use CTRL+C to terminate cURL.

## Alternate Methods

Some of the network policies can use `matchExpressions` instead of `matchLabels`. This will result in fewer policies. 

```yaml

- podSelector:
    matchExpressions:
      - {key: app, operator: In, values: [apigee-cassandra, apigee-udca, apigee-connect-agent]}
```

This will combine three network policies for monitoring to one.