### Test MetalLB deployment and the IP address allocation
Create the namespace:
```bash
$ cat >nginx-test-deployments-namespace.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-test-deployments
EOF

$ kubectl apply -f nginx-test-deployments-namespace.yml
```

Create three nginx deployments and apply them.
```bash
$ cat >nginx-first-deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nginx-test-deployments
  name: nginx-first-deployment
  labels:
    app: nginx-first-deployment
spec:
  selector:
    matchLabels:
      app: nginx-first-deployment
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx-first-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "echo 'This is the FIRST nginx deployment' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]

---
apiVersion: v1
kind: Service
metadata:
  namespace: nginx-test-deployments
  name: nginx-first-deployment-service
  labels:
    app: nginx-first-deployment
  #annotations:
    #metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-first-deployment
  loadBalancerClass: "metallb"
  type: LoadBalancer
EOF
```

```bash
$ cat >nginx-second-deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nginx-test-deployments
  name: nginx-second-deployment
  labels:
    app: nginx-second-deployment
spec:
  selector:
    matchLabels:
      app: nginx-second-deployment
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx-second-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "echo 'This is the SECOND nginx deployment' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]

---
apiVersion: v1
kind: Service
metadata:
  namespace: nginx-test-deployments
  name: nginx-second-deployment-service
  labels:
    app: nginx-second-deployment
  #annotations:
    #metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-second-deployment
  loadBalancerClass: "metallb"
  type: LoadBalancer
EOF
```

```bash
$ cat >nginx-third-deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nginx-test-deployments
  name: nginx-third-deployment
  labels:
    app: nginx-third-deployment
spec:
  selector:
    matchLabels:
      app: nginx-third-deployment
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx-third-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "echo 'This is the THIRD nginx deployment' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]

---
apiVersion: v1
kind: Service
metadata:
  namespace: nginx-test-deployments
  name: nginx-third-deployment-service
  labels:
    app: nginx-third-deployment
  #annotations:
    #metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-third-deployment
  loadBalancerClass: "metallb"
  type: LoadBalancer
EOF
```

Apply the deployments:
```bash
$ kubectl apply -f nginx-first-deployment.yml
$ kubectl apply -f nginx-second-deployment.yml
$ kubectl apply -f nginx-third-deployment.yml
```
<br>

Each of the deployments should have a service of type `LoadBalancer` under the `nginx-test-deployments` namespace with a `Loadbalancer` Ingress field populated by an IP from the IP range configured previously. For example:
```bash
$ kubectl get services -n nginx-test-deployments
NAME                              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx-first-deployment-service    LoadBalancer   10.43.36.169    172.16.0.193    80:30484/TCP   49m
nginx-second-deployment-service   LoadBalancer   10.43.85.177    172.16.0.194    80:30553/TCP   47m
nginx-third-deployment-service    LoadBalancer   10.43.183.195   172.16.0.195    80:32657/TCP   39m
```

From the browser or with `curl` confirm that you can reach the IPs, for example: 
```bash
$ curl http://172.16.0.193
This is the FIRST nginx deployment
```
