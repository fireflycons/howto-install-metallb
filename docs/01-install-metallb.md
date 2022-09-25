# Install MetalLB

The MetalLB documentation may be found [here](https://metallb.universe.tf/).

## Deploy MetalLB

1. Add helm repo
    ```bash
    helm repo add metallb https://metallb.github.io/metallb
    helm repo update
    ```
1. Create namespace for the deployment
    ```
    kubectl create namespace metallb-system
    kubectl config set-context --current --namespace metallb-system
    ```
1. Deploy the chart
    ```
    helm install metallb metallb/metallb
    ```

At this point there will be one Deployment and one DaemonSet created. The pods of the DaemonSet will not be running until we complete the configuration.

## Configure

First, we have to choose some unused IP addresses in the network for MetalLB to own. The way I have my home network set up is as a block of 256 addresses at 192.168.230.0/24.

* The first block of 64 addresses is reserved for static IPs such as the router and any servers (e.g. the cluster nodes)
* The next block of 64 is managed by a DCHP scope for laptops, phones etc.
* The remaining addresses are free - and it is from this pool that I'll assign addresses to MetalLB

To configure MetalLB, we now need to create and apply two configuration resources

1. An [address pool](https://metallb.universe.tf/apis/#metallb.io/v1beta1.IPAddressPool). Addresses many be defined using either a CIDR block, or if you cannot express the addresses you need as a CIDR you can use a range like `192.168.86.10-192.168.86.20`. From the points above, it can be seen that I have free addresses from `192.168.230.128` to `192.168.230.255`, so I'm going to use the range `192.168.230.128/27` which will dedicate addresses  `192.168.230.128` thru `192.168.230.159`.
    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: default-pool
      namespace: metallb-system
    spec:
       addresses:
       - 192.168.230.128/27
    ```
1. A [Layer2 Advertisement](https://metallb.universe.tf/apis/#metallb.io/v1beta1.L2Advertisement) which allows MetalLB  to advertise the LoadBalancer IPs provided by the selected pools via L2.
    ```yaml
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: default-pool-advertisement
      namespace: metallb-system
    spec:
      ipAddressPools:
      - default-pool
    ```
Once you apply these manifests, the DaemonSet pods should start.

## Test

Now we shall create a deployment with a service of type `LoadBalancer` to expose it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deployment
  name: nginx-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - image: nginx
        name: nginx
```

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-deployment
  type: LoadBalancer
```

Give it 30 seconds or so to all come up, then -

```bash
kubectl get services -n default
```

> Output

```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>            443/TCP        41d
nginx        LoadBalancer   10.96.223.238   192.168.230.129   80:31538/TCP   52s
```

Now we see that MetalLB has intercepted the `LoadBalancer` service type and assigned the service an external IP from the address pool! Let's check it out:

```bash
curl http://192.168.230.129
```

> Output

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Next: [Install and configure Ingress](./02-install-ingress.md)