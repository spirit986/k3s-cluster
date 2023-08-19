Once the cluster is fully deployed do the following.

Create a self signed certificate ClusterIssuer

More info on the difference between ClusterIssuer and Issuer you can find on the [cert-manager official documentation](https://cert-manager.io/docs/concepts/issuer/).

If this hasn't been done already during the installation, first create the self-signed ClusterIssuer:
```bash
$ cat >cert-manager-ClusterIssuer.yml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
EOF

$ kubectl apply -f cert-manager-ClusterIssuer.yml
```
<br>

Create a test namespace. I named it `nginx-test-deployments`
```bash
$ cat >nginx-https-test-deployments-namespace.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-test-deployments
EOF

$ kubectl apply -f nginx-https-test-deployments-namespace.yml
```
<br>

Create the certificate (self-signed) which will be used by our CA (Certificate Authority)
```bash
$ cat >nginx-test-deployments-ca-cert.yml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-test-deployments-ca-cert
  namespace: nginx-test-deployments
spec:
  isCA: true
  commonName: project.dev
  secretName: nginx-test-deployments-ca-secret

  privateKey:
    algorithm: ECDSA
    size: 256

  subject:
    organizations:
      - startup.dev
    organizationalUnits:
      - Project Development

  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
    group: cert-manager.io
EOF

$ kubectl apply -f nginx-test-deployments-ca-cert.yml
```
<br>

Create the CA issuer. This will issue certificates for the `nginx-test-deployments` namespace only.
```bash
$ cat >nginx-test-deployments-ca-issuer.yml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: nginx-test-deployments-ca-issuer
  namespace: nginx-test-deployments
spec:
  ca:
    secretName: nginx-test-deployments-ca-secret
EOF

$ kubectl apply -f nginx-test-deployments-ca-issuer.yml
```
<br>

Create the certificate which will be used by our `ingress-nginx` deployment.
```bash
$ cat >nginx-first-https-deployment-cert.yml <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-first-https-deployment-certificate
  namespace: nginx-test-deployments
spec:
  secretName: nginx-https-test-deployments-tls
  
  duration: 87600h #10y
  renewBefore: 3600h

  subject:
    organizations:
      - startup.dev
  
  isCA: False

  privateKey:
    ## Pick either ECDSA or RSA (default)

    #algorithm: ECDSA
    #size: 256

    algorithm: RSA
    encoding: PKCS1
    size: 2048

  usages:
    - server auth
    - client auth

  dnsNames:
    - myproject.startup.dev

  issuerRef:
    name: nginx-test-deployments-ca-issuer
    kind: Issuer
    #group: cert-manager.io
EOF

$ kubectl apply -f nginx-first-https-deployment-cert.yml
```
<br>

Create an example deployment with `ingress-nginx`:
```bash
$ cat >nginx-first-https-deployment.yml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: nginx-test-deployments
  name: nginx-first-https-deployment
  labels:
    app: nginx-first-https-deployment
spec:
  selector:
    matchLabels:
      app: nginx-first-https-deployment
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx-first-https-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        command: ["/bin/sh"]
        args: ["-c", "echo 'This is the FIRST nginx HTTPS deployment' > /usr/share/nginx/html/index.html && nginx -g 'daemon off;'"]

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-first-https-deployment
  namespace: nginx-test-deployments
  labels:
    app: nginx-first-https-deployment
  annotations: {}
spec:
  selector:
      app: "nginx-first-https-deployment"
  ports:
  - name: http-web
    port: 80
    targetPort: 80
    protocol: TCP


---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-first-https-deployment-ingress
  namespace: nginx-test-deployments
  annotations: 
    cert-manager.io/issuer: nginx-test-deployments-ca-issuer

spec:
  ingressClassName: nginx
  tls:
  - hosts: 
    - "myproject.startup.dev"

    secretName: nginx-https-test-deployments-tls

  rules:
  - host: "myproject.startup.dev"
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
           name: nginx-first-https-deployment
           port:
             number: 80
EOF

$ kubectl apply -f nginx-first-https-deployment.yml
```