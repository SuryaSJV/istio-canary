🚀 AKS + Istio + Canary Deployment Project
-------------------------------------------

This project demonstrates how to implement **Canary Deployments** on **Azure Kubernetes Service (AKS)** using **Istio Service Mesh**. It showcases a controlled rollout of a new version of a microservice by gradually shifting traffic between versions using Istio’s intelligent routing.

🎯 Project Goals
-----------------

1. Provision AKS cluster using Azure CLI.

  # Login to Azure
     ```az login```
  # Create Resource Group
     ```az group create --name aks-istio-rg --location eastus```
  # Create AKS Cluster (Standard)
```
az aks create --resource-group aks-istio-rg --name aks-istio-cluster --node-count 
2 --enable-addons monitoring --generate-ssh-keys
```
  # Get AKS Credentials
     ``` az aks get-credentials --resource-group aks-istio-rg --name aks-istio-cluster```


2. Install and configure Istio service mesh.
   # Install Istio (using Istioctl)
   ```
   1. curl -L https://github.com/istio/istio/releases/latest/download/istioctl-linux- 
amd64.tar.gz -o istioctl.tar.gz
   2.tar -xzf istioctl.tar.gz
   3.sudo mv istioctl /usr/local/bin/
   4.istioctl version
   5. Install Istio base + ingress gateway + default profile

```istioctl install --set profile=demo -y```

✅ Check pods:
```
kubectl get pods -n istio-system
```


# Label the Namespace for Istio Injection:
------------------------------------------

- Create a new namespace for your app
```
  kubectl create namespace canary-demo
```

- Enable automatic sidecar injection
```
kubectl label namespace canary-demo istio-injection=enabled
```

3. Deploy a sample HTTP app with two versions: `v1` and `v2`.
   
📄 deployment-v1.yaml

```
   apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-v1
  namespace: canary-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
      version: v1
  template:
    metadata:
      labels:
        app: hello
        version: v1
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo
        args:
        - "-text=Hello version 1"
        ports:
        - containerPort: 5678
```
📄 deployment-v2.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-v2
  namespace: canary-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
      version: v2
  template:
    metadata:
      labels:
        app: hello
        version: v2
    spec:
      containers:
      - name: hello
        image: hashicorp/http-echo
        args:
        - "-text=Hello version 2"
        ports:
        - containerPort: 5678

```

📄 service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: hello-service
  namespace: canary-demo
spec:
  selector:
    app: hello
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

✅ Apply deployments and service:
```
kubectl apply -f deployment-v1.yaml
kubectl apply -f deployment-v2.yaml
kubectl apply -f service.yaml
```
✅ Check pods:

```
kubectl get pods -n canary-demo
```

4. Gradually shift traffic using Istio `VirtualService` (canary rollout).

Create Istio Gateway and VirtualServices for Canary:

📄 gateway.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: hello-gateway
  namespace: canary-demo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

📄 destination-rule.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: hello-destination
  namespace: canary-demo
spec:
  host: hello-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

📄 virtual-service-canary.yaml

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-virtualservice
  namespace: canary-demo
spec:
  hosts:
  - "*"
  gateways:
  - hello-gateway
  http:
  - route:
    - destination:
        host: hello-service
        subset: v1
      weight: 90
    - destination:
        host: hello-service
        subset: v2
      weight: 10
```

✅ Apply all configs:

```
kubectl apply -f gateway.yaml
kubectl apply -f destination-rule.yaml
kubectl apply -f virtual-service-canary.yaml
```

Access Your Application:

-Get the external IP of the Istio Ingress Gateway:
```
kubectl get svc istio-ingressgateway -n istio-system
```

✅ Test App:

```
curl http://<External-IP>/
```

-You should get responses like:
  90%: "Hello version 1"
  10%: "Hello version 2"
Keep running curl multiple times to observe the distribution.


5. Observe rollout behavior with real requests.
-Shift Traffic Gradually:

📄 Update VirtualService to shift traffic 50-50:

```
http:
- route:
  - destination:
      host: hello-service
      subset: v1
    weight: 50
  - destination:
      host: hello-service
      subset: v2
    weight: 50
```
✅ Apply updated virtual service:

```
kubectl apply -f virtual-service-canary.yaml
```
✅ Expected:
Now, 50% requests go to v1 and 50% to v2.


6. Roll forward to full `v2` once validated.

-Full Rollout to v2:

📄 Update VirtualService to 100% to v2:

```
http:
- route:
  - destination:
      host: hello-service
      subset: v2
    weight: 100
```

✅ Apply change:

```
kubectl apply -f virtual-service-canary.yaml
```
✅ Expected:

Now, all requests serve "Hello version 2".

🎯 Final Validation
✅ Check Service
✅ Check Istio Gateway
✅ Confirm traffic split
✅ Confirm deployment behavior


## 🧱 Architecture

```plaintext
Client
  │
  ▼
[Istio Ingress Gateway]
  │
  ▼
[VirtualService (Canary Routing)]
  │
  ├──> [Deployment: v1] (e.g. 90%)
  └──> [Deployment: v2] (e.g. 10%)


⚙️ Tools Used
Tool	    -   Purpose
------------------------------------
Azure CLI	-   AKS provisioning
kubectl	    -   Kubernetes management
Istioctl	-   Istio installation & management
Istio	    -   Service Mesh & Traffic Routing
HashiCorp   -   HTTP-Echo	Sample app used for testing  