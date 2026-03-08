# Envoy Gateway on Oracle Kubernetes Engine (OKE)

This repo contains a minimal, working example of running Envoy Gateway with Gateway API
on Oracle Kubernetes Engine (OKE), fronted by an OCI Load Balancer.

## TLS support added using Kubernetes stored secrets and LB TLS termination

New available annotations allow us to offload Kubernetes stored secrets via ALB CCM integration, this can also be done via LetsEncrypt and Certificate managed certificates - will be added soon. 

## Structure

- `manifests/`
  - `envoyproxy-oci.yaml` - Configures Envoy Gateway to expose an OCI Load Balancer with flexible shape and TLS Termination.
  - `gatewayclass.yaml`   - GatewayClass that binds to the EnvoyProxy.
  - `gateway.yaml`        - Public HTTP Gateway with HTTP and HTTPS sections.
  - `demo-app.yaml`       - Simple http-echo demo application and Service.
  - `httproute.yaml`      - HTTPRoute that sends all traffic (`/`) to `my-app` bound to both HTTP and HTTPS.

## Pods as Backends on OKE

This deployment demonstrates a production‑grade configuration of Envoy Gateway on Oracle Kubernetes Engine (OKE) where the **OCI Load Balancer directly targets pod IPs**, rather than routing traffic through worker node IPs.

OKE supports this capability through its Enhanced Cluster architecture and the Native Pod Network (NPN) CNI, which allows OCI Load Balancers to register pods as backends. This improves traffic efficiency, reduces hop count, and enables more consistent load distribution.

Official Oracle documentation on pods-as-backends:
https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengconfiguringloadbalancersnetworkloadbalancers-subtopic.htm#contengcreatingloadbalancer_topic_Specifying_pods_as_backends

## Prerequisites

- An OKE cluster (1.30+ recommended).
- `kubectl` configured against the OKE cluster.
- `helm` installed locally.

## 1. Set the Pods NSG OCID

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

## 2. Install Envoy Gateway (current stable v1.7.0) - this Helm repo deploys Gateway API CRDs - if you preinstalled Gateway API CRDs this installation will fail!

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.7.0 -n envoy-gateway-system --create-namespace
```

Confirm the Envoy Gateway pods are running.

```bash
kubectl -n envoy-gateway-system get pods
```

## 3. Provide/Create TLS certificate as Kubernets secret for Loadbalancer TLS termination (if you have your own certificate use your required Ceritificate and key)

For the purpose of this demostration i will use a self-signed TLS certificate - 

Create the certificate and key - 
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=oke-envoy-gateway-bootstrap"
```
Create the Kubernetes secret and populate it with the previously created certificate and key - 
```bash
kubectl create secret tls envoy-gateway-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n envoy-gateway-system
```

Attention - this will create a localy stored tls.key and tls.crt files! removed these when done! this is not to be used in any scenario which is not a demo!

## 4. Apply manifests

```bash
kubectl apply -k manifests/
```

## 5. Get the external IP

Check the obejct of type Service:Loadbalancer has been successfully assigned a public IP, run the following command to populate an environment variable, if this fails your LB was not properly created, this can be caused by not assigning the correct NSG OCID as required in Step 1.

```bash
LB_IP=$(kubectl get svc -n envoy-gateway-system -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
echo $LB_IP
```

Now we should be able to curl our LB on HTTP - 

```bash
curl http://$LB_IP/
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
	request_uri=http://X.X.X.X:8080/

Request Headers:
	accept=*/*
	host=X.X.X.X
	user-agent=curl/8.7.1
	x-envoy-external-address=10.X.X.X
	x-forwarded-for=Y.Y.Y.Y,10.X.X.X
	x-forwarded-host=X.X.X.X:80
	x-forwarded-port=80
	x-forwarded-proto=http
	x-real-ip=Y.Y.Y.Y
	x-request-id=b225d939-6011-4535-ab64-cf5c97c48b63

Request Body:
	-no body in request-
```

Now Lets check our HTTPS termination, we will use the "-k" option to ensure curl will not check the validity of our certificate as it is self-signed, if you used your own valid certificate and potined your DNS record to the LB IP address you can remove the "-k" flag.

```bash
curl -k https://$LB_IP/
```

## 5. Cleanup

```bash
rm tls.crt tls.key
kubectl delete -k manifests/
helm uninstall eg -n envoy-gateway-system
kubectl delete namespace envoy-gateway-system
```
