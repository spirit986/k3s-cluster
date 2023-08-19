## Test Longhorn deployment functionality
The example bellow tests volume provisioning and functionality with two modes:
* **ReadWriteOnce** - A.K.A. RWO Mode | This is the default for most storage provisioners. Only one pod can write on the volume. Additionaly multiple pods can also access the volume BUT ONLY if all of them are deployed on the same node where the volume is attached.
* **ReadWriteMany** - A.K.A. RWX Mode | Multiple pods can access the volume even if they are on different nodes. Since the RWX mode is more complex for implementation, additional reference and how it is implemented can be found on the Official [Longhorn Documentation](https://longhorn.io/docs/1.4.2/advanced-resources/rwx-workloads/).

#### Additional notes
Since both of the deployments during this trial make use of `metallb` to expose the services, the exibit tests the `metallb` functionality as well.
<br>
<br>

## Create the namespace separately
If later one want's to delete everything from the test, he/she can only delete the namespace.

Create a test namespace. I named it `longhorn-test-deployments`
```bash
$ cat >longhorn-test-deployments-namespace.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: longhorn-test-deployments
EOF

$ kubectl apply -f longhorn-test-deployments-namespace.yml
```
<br>

## Deploy MySQL with Longhorn persistent volume in **RWO** Mode (ReadWriteOnce)
Create a the PVC, the Service and the Deployment. The service type is of a LoadBalancer and will utilize the MetalLB deployment. You might want to deploy and configure MetalLB, prior executing this test.

By using the `annotations.metallb.universe.tf/loadBalancerIPs` on the `Service` you can also set a static IP address assignment.

```bash
$ cat >mysql-with-pv-deployment.yml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-test-deployment-pvc
  namespace: longhorn-test-deployments
  labels:
    app: mysql-test-deployment
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn-best-effort-reclaim-delete
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-test-deployment-service
  namespace: longhorn-test-deployments
  labels:
    app: mysql-test-deployment
  #annotations:
    #metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
    - port: 3306
      targetPort: 3306
      protocol: TCP
  selector:
    app: mysql-test-deployment
  type: LoadBalancer
  loadBalancerClass: "metallb"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-test-deployment
  namespace: longhorn-test-deployments
  labels:
    app: mysql-test-deployment
spec:
  selector:
    matchLabels:
      app: mysql-test-deployment
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql-test-deployment
    spec:
      restartPolicy: Always
      containers:
      - image: mysql:5.6
        name: mysql
        livenessProbe:
          exec:
            command:
              - ls
              - /var/lib/mysql/lost+found
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: changeme-to-whatever
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-test-deployment-volume
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-test-deployment-volume
        persistentVolumeClaim:
          claimName: mysql-test-deployment-pvc
EOF

$ kubectl apply -f mysql-with-pv-deployment.yml
```

### Verification steps
1. Open the Longhorn UI
2. **Dashboard section** - If this is the first deployment, one volume will be visible on the left gauge.
3. **Volume section** - the volume will be visible showing as **Healthy**, with the name like `pvc-aeab180a-204e-4b9c-a1f2-5dfc63e27556`, and the **Attached To** the name of the deployment `mysql-test-deployment-799fdb9765` along with the node name where the volume is actually attached on the node.
4. If you click on the volume name it will take you to a separate page where you can see additional details about the volume on the left side pannel. Right pannel will show the replicas on each node and the botton pannel will offer a preview ofthe Snapshots and backups.
5. The snapshot function should work out of the box. The backup function requires a backup target to be configured.
<br><br>


## Deploy NGINX with Longhorn persistent volume in **RWX** Mode (ReadWriteMany)
The snipet bellow will test the ReadWriteMany mode. In this setup multiple pods running on different nodes will be able to read and write from the same volume.

This is very usefull for distributed web applictaions where the web app will be exposed on more than one node for load balancing. An example will be a kubernetes wordpress deployment. Multiple wordpress apps can be deployed (one on each node) and will be able to share the load but each will store the images on the same volume.

```bash
$ cat >nginx-with-pv-rwx-deployment.yml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-test-deployment-rwx-pvc
  namespace: longhorn-test-deployments
  labels:
    app: nginx-test-deployment-rwx
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn-best-effort-reclaim-delete
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test-deployment-rwx-service
  namespace: longhorn-test-deployments
  labels:
    app: nginx-test-deployment-rwx
  #annotations:
    #metallb.universe.tf/loadBalancerIPs: 192.168.1.100
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: nginx-test-deployment-rwx
  type: LoadBalancer
  loadBalancerClass: "metallb"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test-deployment-rwx
  namespace: longhorn-test-deployments
  labels:
    app: nginx-test-deployment-rwx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-test-deployment-rwx
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nginx-test-deployment-rwx
    spec:
      restartPolicy: Always

      containers:
      - image: nginx:1.19.0
        name: nginx
        livenessProbe:
          exec:
            command:
              - ls
              - /usr/share/nginx/html
          initialDelaySeconds: 5
          periodSeconds: 5
        ports:
        - containerPort: 80
          name: http
        volumeMounts:
        - name: nginx-test-deployment-rwx-volume
          mountPath: /usr/share/nginx/html

      - image: ubuntu:20.04
        name: ubuntu-sidecar
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - name: nginx-test-deployment-rwx-volume
          mountPath: /shared-data

      volumes:
      - name: nginx-test-deployment-rwx-volume
        persistentVolumeClaim:
          claimName: nginx-test-deployment-rwx-pvc
EOF

$ kubectl apply -f nginx-with-pv-rwx-deployment.yml
```

### Verification steps
Verification can be done from the LensUI or with the `kubectl` cli.
1. From the Lens UI click on **Deployments** on the left sidebar. From the **Namespace** selector dropdown (top right) select the `longhorn-test-deployments` namespace.
2. Find and observe the `nginx-test-deployment-rwx`, make sure that 3/3 pods are running.
3. On the left sidebar again on **Pods** and on the main pannel locate the `nginx-test-deployment-rwx-ID-NUMBERS-HERE` pods which are the pods from the same deployment. Each of the pods should have two containers, the `nginx` container and the `ubuntu-sidecar` container. The `ubuntu-sidecar` container is for ease of access convinience.
4. Execute a shell to the three sidecar containers. 
  - Click on one of the `nginx-test-deployment-rwx` pods.
  - From the right pannel that shows up on the top righ there is a CLI like icon. Hower the mouse cursor on top of it and from the pop-up menu select the `ubuntu-sidecar` container. A linux shell will open on the bottom of the Lens UI.
  - Do this for the all three pods.

5. On the shell that opens navigate to `/shared-data` and execute the snippet bellow:
```bash
$ cd /shared-data
$ cat >file1.txt <<EOF
One
Two
Three
EOF
```
6. Verify that the file is present also on the other two pods from their own shell window. Additionaly alter the file from one of the two pods and make sure that the changes have reflected accordingly.
