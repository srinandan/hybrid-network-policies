# Apigee hybrid Network Policies

This sample provides a set of instructions to secure an Apigee hybrid installation through network policies. 

## Background

This sample uses:

* [Apigee hybrid](https://docs.apigee.com/hybrid/beta2/reference-overview), currently in Beta
* [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/) to setup/install Apigee hybrid 
* [Project Calico](https://docs.projectcalico.org/v3.5/introduction/) as the CNI plugin

## Objective

Kubernetes network policies allow/control how pods communicate with each other (as well as how/if they can communicate externally/outside the cluster). A typical Apigee hybrid installation is made of multiple pods:

* apigee-deployment-admissionhook
* apigee-deployment-controller
* apigee-connect-agent - Apigee Connect Agent to communicate with the control plane
* apigee-runtime - Apigee API Gateway
* apigee-cassandra - API Gateway persistence layer
* apigee-udca - Apigee Analytics Agent (sends raw analytics to the control plane)
* apigee-metrics - Apigee Metrics Agent (sends application metrics to Stackdriver)
* apigee-logger - Apigee Logger Agent (sends applications logs to Stackdriver)
* apigee-mart - Apigee Administrative API Endpoint
* apigee-synchronizer - Apigee Agent to sync configuration between the control plane and runtime (data) plane

Each of these pods require specific access to ports and not every pod needs to communicate with every other pod.

## Installation Topology

This basic setup assumes all the Apigee hybrid components are installed on a single namespace called `apigee`

## Installing the Policy

Step 1: Label namespaces, these will come in handy for match selectors

```bash

kubectl label namespace apigee app=apigee
kubectl label namespace kube-system namespace=kube-system
```

Step 2: Apply Network policies

```bash

kubectl apply -f .
```

## Validation

The following network policies should be created

```bash

NAME                                   POD-SELECTOR                     AGE
netpol-allow-dns-cassandra             apigee.com/component=cassandra   37s
netpol-allow-dns-mart                  component=mart                   37s
netpol-allow-dns-metrics               component=apigee-metrics         37s
netpol-allow-dns-runtime               component=runtime                37s
netpol-allow-dns-sync                  component=synchronizer           36s
netpol-allow-dns-udca                  component=apigee-udca            37s
netpol-cassandra-mart                  app=apigee-cassandra             38s
netpol-cassandra-monitor               app=apigee-cassandra             37s
netpol-cassandra-runtime               app=apigee-cassandra             38s
netpol-cassandra-server                app=apigee-cassandra             37s
netpol-mart                            component=mart                   36s
netpol-mart-authz                      component=mart                   36s
netpol-monitoring-cassandra            app=apigee-metrics               36s
netpol-monitoring-mart                 component=mart                   35s
netpol-monitoring-prometheus-mart      app=apigee-metrics               36s
netpol-monitoring-prometheus-runtime   app=apigee-metrics               36s
netpol-monitoring-prometheus-sync      app=apigee-metrics               35s
netpol-monitoring-runtime              component=runtime                35s
netpol-runtime                         component=runtime                35s
netpol-sync                            component=synchronizer           35s
netpol-sync-runtime                    component=synchronizer           35s
netpol-udca                            component=apigee-udca            35s
netpol-udca-runtime                    component=apigee-udca            34s
```

### Testing the policies

Step 1: Add a test/sample container with cURL and other network utilities installed

#### Before Network policies
Step 2: Use cURL to access a pod (even if it is not an http endpoint)

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

Step 3: Use cURL again to access a pod

```bash

~ $ curl apigee-cassandra.apigee.svc.cluster.local:7000 -v
* Rebuilt URL to: apigee-cassandra.apigee.svc.cluster.local:7000/
*   Trying 10.56.1.3...
* TCP_NODELAY set
^C
```

The network connection does not complete. Use CTRL+C to terminate cURL.

## Explanation of policies

The following policies allow access to kube dns only:

* [netpol-allow-dns-cassandra](./netpol-dns.yaml)

The following policies allow access to kube DNS and Google DNS:

* [netpol-allow-dns-mart](./netpol-dns.yaml)
* [netpol-allow-dns-runtime](./netpol-dns.yaml)
* [netpol-allow-dns-sync](./netpol-dns.yaml)
* [netpol-allow-dns-udcs](./netpol-dns.yaml)
* [netpol-allow-dns-metrics](./netpol-dns.yaml)

The following policies secure Apigee Cassandra:

* [netpol-cassandra-mart](./netpol-cassandra-client.yaml): Allows access from MART to Cassandra
* [netpol-cassandra-runtime](./netpol-cassandra-client.yaml): Allows access from Runtime to Cassandra
* [netpol-cassandra-server](./netpol-cassandra-server.yaml): Allows Cassandra nodes to communicate with each other
* [netpol-cassandra-monitor](./netpol-cassandra-monitoring.yaml): Allows monitoring of Cassandra nodes

The following policies secure Apigee MART:

* [netpol-mart](./netpol-mart.yaml): Allows MART to be accessed by mart-istio-ingressgateway only
* [netpol-mart-authz](./netpol-mart.yaml): Allow MART authn-authz to access the internet (apigee.googleapis.com)

The following policies secure Apigee Runtime (Gateway):

* [netpol-runtim](./netpol-runtime.yaml): Allows the Apigee runtime to be accessed by istio-ingressgateway only

NOTE: The runtime itself does not have other policies. We cannot predict by default if the gateway accesses services inside the cluster and inside and outside the cluster.

The following policies secure Apigee synchronizer:

* [netpol-sync](./netpol-sync.yaml): Allow the sync to reach the control plane (apigee.googleapis.com)
* [netpol-sync-runtime](./netpol-sync.yaml): Allow only the runtime to access the synchronizer (for proxy bundles etc.)

The following policies secure Apigee UDCA (Analytics Agent):

* [netpol-udca](./netpol-udca.yaml): Allow UDCA access the control plane to send analytics
* [netpol-udca-runtime](./netpol-udca.yaml): Allow the local AX endpoint to be accessed to Runtime only
* [netpol-udca-token](./netpol-udca.yaml): Allow UDCA access to the token endpoint

The following policies secure Apigee Metrics/Monitoring:

* [netpol-monitoring-cassandra](./netpol-monitoring.yaml): Allow monitoring of Cassandra by the metrics pod
* [netpol-monitoring-prometheus-runtime](./netpol-monitoring.yaml): Allow monitoring of Runtime by the metrics pod
* [netpol-monitoring-prometheus-mart](./netpol-monitoring.yaml): Allow monitoring of MART by the metrics pod
* [netpol-monitoring-prometheus-sync](./netpol-monitoring.yaml): Allow monitoring of Sync by the metrics pod
