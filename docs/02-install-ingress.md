
# Configure Ingress to use MetalLB

If we simply deploy LoadBalancer services for all our workloads, then we're soon going to use up the IP address pool! Let's now configure the venerable nginx ingress to work with MetalLB, then we can take advantage of all the ingress goodness such as host and path based routing.

## Prerequisites

If you are actually intending to run workloads in your cluster, and you're not building this purely as an exercise, then:

* Your network has an internal DNS server and the clients on your network are configured to use it (normally via DHCP options).
* If you do not have a DNS server, then you can add the hosts you want to use with ingress to your local hosts file, although this becomes a lot more difficult if you expect devices such as phones and tablets to be able to connect.

## Install Ingress Controller

1. Create namespace for the deployment
    ```
    kubectl create namespace ingress-nginx
    kubectl config set-context --current --namespace ingress-nginx
    ```
1. Deploy the chart. The default installation options for nginx ingress is for its service to request a LoadBalancer. Since we've already installed MetalLB, then MetalLB will pick up on the service creation.
    ```bash
    helm upgrade --install ingress-nginx ingress-nginx \
       --repo https://kubernetes.github.io/ingress-nginx
    ```

    Wait 30 seconds or so for it to all come up.

    ```bash
    kubectl get service
    ```

    > Output

    ```
    NAME                          TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                      AGE
    ingress-nginx-nginx-ingress   LoadBalancer   10.96.137.43   192.168.230.128   80:32010/TCP,443:31416/TCP   45s
    ```

    We can see the ingress controller now has an external IP allocated from MetalLB's address pool.

## Test

For the test, we will use the [same nginx deployment](../manifests/test-deployment.yaml) we used to test the MetalLB installation, but this time we will give it a [ClusterIP service](../manifests/test-clusterip-service.yaml) and an [Ingress resource](../manifests/test-ingress.yaml) to attach it to the ingress controller.

Deploy service and ingress resources:

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
  type: ClusterIP
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-test
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: nginx-test
            port:
              number: 80
```

And test...

```bash
# Get IP of the ingress service
IP=$(kubectl get services -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

curl -H "Host: foo.bar.com" http://$IP/testpath
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


