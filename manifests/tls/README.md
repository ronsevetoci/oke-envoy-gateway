# TLS for Envoy Gateway on OKE
This folder shows how to add HTTPS (TLS termination with cert-manager + Let’s Encrypt) to an existing Envoy Gateway deployment on OKE, using nip.io for quick, disposable DNS.

---

## 📌 Prerequisites

* A working **Envoy Gateway** installation on OKE
* An `EnvoyProxy` object that creates a **LoadBalancer Service**
* A valid **public IP address** for the Envoy LB
* A functional **Gateway** and **HTTPRoute**
* A deployed backend app (e.g. `demo-app`)

---

## 🚀 Install cert-manager (Simple Helm Method)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --set "extraArgs={--enable-gateway-api}"
```

```bash
kubectl get pods -n cert-manager
```

---

## 🌐 1. Choose a Domain

We use [nip.io](https://nip.io), a wildcard DNS service that maps `<IP>.nip.io` directly to that IP address.  
This lets you get a valid hostname without buying or configuring a real domain.

1. Get the external IP of the Envoy LoadBalancer:

   ```bash
   LB_IP=$(kubectl get svc -n envoy-gateway-system -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
   echo "$LB_IP"
   ```

2. Build a hostname for TLS:

   ```bash
   DOMAIN="echo.${LB_IP}.nip.io"
   echo "$DOMAIN"
   ```

3. Use this hostname (`$DOMAIN`) as the DNS name in the TLS manifests. The sample YAMLs in this folder already use `echo.<something>.nip.io` and can be updated with a simple search/replace.

---

## 🔐 2. Create a ClusterIssuer (Let’s Encrypt)

Apply:

```bash
kubectl apply -f manifests/tls/clusterissuer.yaml
```

---

## 📄 3. Request a Certificate

Apply:

```bash
kubectl apply -f manifests/tls/certificate.yaml
```

Check status:

```bash
kubectl describe certificate echo-tls -n default
kubectl get challenges -A
```

---

## 🌉 4. Add HTTPS Listener to the Gateway

Apply:

```bash
kubectl apply -f gateway-https.yaml
```

---

## 🌉 5. Create HTTP Route bound to the TLS hostname

Apply:

```bash
kubectl apply -f manifests/tls/route_tls.yaml
```

---

## 🧪 6. Test HTTPS

```bash
curl -v https://echo.79.76.98.111.nip.io/
```