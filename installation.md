# Install WSL + Ubuntu 
https://docs.microsoft.com/fr-fr/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package

# Install minikube on Fedora 
https://www.tutorialworks.com/kubernetes-fedora-dev-setup/

By default, minikube only exposes ports 30000-32767
`minikube start --extra-config=apiserver.service-node-port-range=1-65535`

# Install with kind on fedora 
## Install & start Docker 
`sudo dnf install docker-ce docker-ce-cli containerd.io`
`sudo systemctl start docker`

## Create the cluster 

Create a kind cluster with extraPortMappings and node-labels.

* extraPortMappings allow the local host to make requests to the Ingress controller over ports 80/443
* node-labels only allow the ingress controller to run on a specific node(s) matching the label selector

```
$ cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/

```

# Podinfo install 

## Create deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  labels:
    app: podinfo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: podinfo 
  template:
    metadata:
      labels:
        app: podinfo
    spec:
      containers:
      - name: podinfo
        image: stefanprodan/podinfo
        ports:
        - containerPort: 80
          protocol: TCP
```
### apply 
`$ k apply -f deployment.yaml`
`deployment.apps/podinfo created`
```
$ k describe svc podinfo-svc
Name:              podinfo-svc
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=podinfo
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.9.32
IPs:               10.96.9.32
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.0.5:80,10.244.0.6:80
Session Affinity:  None
Events:            <none>
```
## Create service.yaml 

### service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: podinfo-svc
spec:
  selector:
    app: podinfo
  ports:
    - port: 80
      protocol: TCP
```



## Setting up ingress
### ingress NGINX
```
$ VERSION=$(curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/stable.txt)
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    19  100    19    0     0     69      0 --:--:-- --:--:-- --:--:--    69
```
```
[lpirbay@localhost podinfo]$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/${VERSION}/deploy/static/provider/kind/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
```
### Check ingress is running 
``` 
$ kubectl wait --namespace ingress-nginx \
>   --for=condition=ready pod \
>   --selector=app.kubernetes.io/component=controller \
>   --timeout=90s
pod/ingress-nginx-controller-6cd89dbf45-kj2bz condition met

```
## Create ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo-ingress
  # annotations:
  #  nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: podinfo-svc
            port:
              number: 8080
```
### Apply 

```
[lpirbay@localhost podinfo]$ k apply -f ingress.yaml 
ingress.networking.k8s.io/podinfo-ingress created
[lpirbay@localhost podinfo]$ k describe ingress podinfo-ingress
Name:             podinfo-ingress
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /   podinfo-svc:8080 (10.244.0.5:80,10.244.0.6:80)
Annotations:  <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    17s   nginx-ingress-controller  Scheduled for sync
```
## Set resource limits
# Configure the secret

## Encode base64  
``` 
[lpirbay@localhost podinfo]$ YWRtaW4=
[lpirbay@localhost podinfo]$ echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
[lpirbay@localhost podinfo]$ echo -n 'podinfo' | base64
cG9kaW5mbw==
```
## Create sercret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: cG9kaW5mbw==
```

