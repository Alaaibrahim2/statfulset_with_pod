# Kubernetes StatefulSet with Static AWS EBS Storage

A hands-on Kubernetes project demonstrating how to deploy a **MySQL StatefulSet** on a self-managed **kubeadm cluster running on AWS EC2**, using **statically provisioned Amazon EBS volumes** through the **AWS EBS CSI Driver**.

The project focuses on understanding how StatefulSets provide stable Pod identity and dedicated persistent storage for stateful workloads.

## Architecture

```text
                         AWS
                          │
              ┌───────────┴───────────┐
              │   kubeadm Kubernetes  │
              │       Cluster         │
              └───────────┬───────────┘
                          │
                 ┌────────┴────────┐
                 │                 │
              Worker 1          Worker 2
                 │                 │
              MySQL Pods       MySQL Pods
                 │                 │
        ┌────────┼─────────┐       │
        ▼        ▼         ▼       │
      PVC-0    PVC-1     PVC-2     │
        │        │         │       │
        ▼        ▼         ▼       │
       PV-0     PV-1      PV-2      │
        │        │         │       │
        ▼        ▼         ▼       │
      EBS-0    EBS-1     EBS-2
```

## Technologies

* Kubernetes `v1.30.14`
* kubeadm
* AWS EC2
* Amazon EBS
* AWS EBS CSI Driver
* Helm `v3.19.4`
* MySQL `8.0`
* StatefulSet
* PersistentVolume (PV)
* PersistentVolumeClaim (PVC)
* Headless Service

## Project Design

The cluster contains:

```text
1 Master
2 Worker Nodes
```

All EC2 instances are located in:

```text
us-east-1d
```

Three EBS volumes are manually created in AWS:

```text
EBS-0
EBS-1
EBS-2
```

Each EBS volume is represented by a static Kubernetes PersistentVolume:

```text
EBS-0 → mysql-pv-0
EBS-1 → mysql-pv-1
EBS-2 → mysql-pv-2
```

The PVs use the AWS EBS CSI driver:

```yaml
csi:
  driver: ebs.csi.aws.com
  volumeHandle: <EBS-VOLUME-ID>
```

`volumeHandle` identifies the existing EBS volume that the PV represents.

## Storage Flow

The complete storage relationship is:

```text
StatefulSet
     │
     ▼
volumeClaimTemplates
     │
     ▼
PVC
     │
     ▼
Static PV
     │
     ▼
AWS EBS
```

The StatefulSet automatically creates one PVC for every Pod:

```text
mysql-0 → mysql-data-mysql-0
mysql-1 → mysql-data-mysql-1
mysql-2 → mysql-data-mysql-2
```

The static PVs are pre-associated with the expected PVCs using `claimRef`:

```text
mysql-pv-0 → mysql-data-mysql-0
mysql-pv-1 → mysql-data-mysql-1
mysql-pv-2 → mysql-data-mysql-2
```

Therefore:

```text
mysql-0
  ↓
mysql-data-mysql-0
  ↓
mysql-pv-0
  ↓
EBS-0

mysql-1
  ↓
mysql-data-mysql-1
  ↓
mysql-pv-1
  ↓
EBS-1

mysql-2
  ↓
mysql-data-mysql-2
  ↓
mysql-pv-2
  ↓
EBS-2
```

## AWS IAM

An IAM Role is attached to the EC2 instances and provides the permissions required by the AWS EBS CSI Driver to interact with Amazon EBS.

The EBS CSI Driver is installed using Helm:

```bash
helm repo add aws-ebs-csi-driver \
  https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update

helm upgrade --install aws-ebs-csi-driver \
  aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system
```

Verify the driver:

```bash
kubectl get pods -n kube-system | grep ebs
```

Expected result:

```text
ebs-csi-controller-xxxxx   Running
ebs-csi-controller-xxxxx   Running
ebs-csi-node-xxxxx         Running
ebs-csi-node-xxxxx         Running
ebs-csi-node-xxxxx         Running
```

## Static PersistentVolumes

Example PV:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv-0
spec:
  capacity:
    storage: 5Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: mysql-static

  claimRef:
    namespace: default
    name: mysql-data-mysql-0

  csi:
    driver: ebs.csi.aws.com
    volumeHandle: <EBS-VOLUME-ID>
```

Three PVs are created, one for each EBS volume.

## StatefulSet

The MySQL StatefulSet runs three replicas:

```yaml
replicas: 3
```

This creates stable Pod identities:

```text
mysql-0
mysql-1
mysql-2
```

The StatefulSet uses:

```yaml
volumeClaimTemplates:
  - metadata:
      name: mysql-data
```

which automatically creates:

```text
mysql-data-mysql-0
mysql-data-mysql-1
mysql-data-mysql-2
```

The MySQL data directory is mounted at:

```text
/var/lib/mysql
```

This directory is backed by the dedicated EBS volume assigned to each Pod.

## Headless Service

A Headless Service is used for stable network identities:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

The StatefulSet uses:

```yaml
serviceName: mysql
```

This provides stable DNS identities such as:

```text
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

Pod IP addresses may change, but the StatefulSet Pod identity and DNS name remain stable.

## MySQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8.0

          ports:
            - containerPort: 3306

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "rootpassword"

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        storageClassName: mysql-static
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
```

## Verification

Check StatefulSet Pods:

```bash
kubectl get pods -o wide
```

Check PVCs:

```bash
kubectl get pvc
```

Expected:

```text
NAME                   STATUS   VOLUME
mysql-data-mysql-0     Bound    mysql-pv-0
mysql-data-mysql-1     Bound    mysql-pv-1
mysql-data-mysql-2     Bound    mysql-pv-2
```

Check PVs:

```bash
kubectl get pv
```

Check EBS CSI attachments:

```bash
kubectl get volumeattachments
```

## Persistence Test

Connect to `mysql-0`:

```bash
kubectl exec -it mysql-0 -- mysql -u root -p
```

Create a database and table:

```sql
CREATE DATABASE testdb;

USE testdb;

CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

INSERT INTO users VALUES (1, 'Alaa');

SELECT * FROM users;
```

Delete the Pod:

```bash
kubectl delete pod mysql-0
```

The StatefulSet automatically recreates:

```text
mysql-0
```

The new Pod receives the same:

```text
Pod identity
     ↓
PVC
     ↓
PV
     ↓
EBS volume
```

Verify the data again:

```bash
kubectl exec -it mysql-0 -- mysql -u root -p
```

```sql
USE testdb;

SELECT * FROM users;
```

The previously created data should still exist.

## Key Concepts Demonstrated

* StatefulSet stable Pod identity
* StatefulSet `volumeClaimTemplates`
* Static PersistentVolume provisioning
* PersistentVolumeClaim binding
* PV `claimRef`
* AWS EBS CSI Driver
* EBS-backed persistent storage
* `ReadWriteOnce` storage
* Headless Services
* Stable Pod DNS identities
* Persistent data after Pod recreation
* Separation between Pod lifecycle and storage lifecycle
* AWS Availability Zone considerations

## Important Note

This project demonstrates **persistent storage and StatefulSet behavior**, but creating three MySQL Pods does **not automatically create a MySQL replication cluster**.

Each Pod has its own independent MySQL data directory and its own EBS volume.

For production-grade MySQL replication, additional MySQL replication configuration or a dedicated MySQL operator would be required.
