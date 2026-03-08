# Envoy Gateway on Oracle Kubernetes Engine (OKE)

This repo contains a minimal, working example of running Envoy Gateway with Gateway API
on Oracle Kubernetes Engine (OKE), fronted by an OCI Load Balancer.

## Structure

- `manifests/`
  - `envoyproxy-oci.yaml` - Configures Envoy Gateway to expose an OCI Load Balancer with flexible shape.
  - `gatewayclass.yaml`   - GatewayClass that binds to the EnvoyProxy.
  - `gateway.yaml`        - Public HTTP Gateway in the `default` namespace.
  - `demo-app.yaml`       - Simple http-echo demo application and Service.
  - `httproute.yaml`      - HTTPRoute that sends all traffic (`/`) to `my-app`.

## Pods as Backends on OKE

This deployment demonstrates a production‑grade configuration of Envoy Gateway on Oracle Kubernetes Engine (OKE) where the **OCI Load Balancer directly targets pod IPs**, rather than routing traffic through worker node IPs.

OKE supports this capability through its Enhanced Cluster architecture and the Native Pod Network (NPN) CNI, which allows OCI Load Balancers to register pods as backends. This improves traffic efficiency, reduces hop count, and enables more consistent load distribution.

Official Oracle documentation on pods-as-backends:
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengconfiguringloadbalancersnetworkloadbalancers-subtopic.htm#contengcreatingloadbalancer_topic_Specifying_pods_as_backends

## Prerequisites

- An OKE cluster (1.30+ recommended).
- `kubectl` configured against the OKE cluster.
- `helm` installed locally.

### Required Update: Set the Pods NSG OCID

Before applying the manifests, you **must update** the file:

```
`manifests/envoyproxy-oci.yaml`
```

and replace the placeholder:

```
<PODS_NSG_OCID>
```

with the actual OCID of the Network Security Group (NSG) attached to your **pods subnet**.

This OCID is required so the OCI Load Balancer can register pod IPs as backends when operating in **NSG rule‑management mode**.

## 1. Install Envoy Gateway (current stable v1.7.0)

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.7.0 -n envoy-gateway-system --create-namespace
```

```bash
kubectl -n envoy-gateway-system get pods
```

## 2. Apply manifests

```bash
kubectl apply -k manifests/
```

## 3. Get the external IP

```bash
kubectl get svc -A -l gateway.envoyproxy.io/owning-gateway-name=public-gateway
```

Take the `EXTERNAL-IP` of the Envoy Gateway LoadBalancer service and test:

```bash
curl http://<EXTERNAL-IP>/
```

You should get a response with your request information and the details of the pod answering:

```text
Hostname: demo-app-6d449f9fd9-9wxg6

Pod Information:
	-no pod information available-

Server values:
	server_version=nginx: 1.13.3 - lua: 10008

Request Information:
	client_address=10.0.164.247
	method=GET
	real path=/
	query=
	request_version=1.1
	request_uri=http://138.2.156.172:8080/

Request Headers:
	accept=*/*
	host=138.2.156.172
	user-agent=curl/8.7.1
	x-envoy-external-address=10.0.2.55
	x-forwarded-for=62.56.234.66,10.0.2.55
	x-forwarded-host=138.2.156.172:80
	x-forwarded-port=80
	x-forwarded-proto=http
	x-real-ip=62.56.234.66
	x-request-id=b225d939-6011-4535-ab64-cf5c97c48b63

Request Body:
	-no body in request-
```

## 4. Cleanup

```bash
kubectl delete -k manifests/
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace envoy-gateway-system
```

## 5. TLS setup
For TLS configuration, including Let's Encrypt and HTTPS setup,
see the dedicated guide in [`manifests/tls/README.md`](manifests/tls/README.md).
