# Sangfor EDS CSI 1.0.0
[Container Storage Interface (CSI)](https://github.com/container-storage-interface/), driver and provisioner for the Sangfor EDS nasplugin.

# Overview

Sangfor EDS CSI plugins implement an interface between CSI and Container Orchestrator (CO) - EDS cluster.

Current Sangfor EDS CSI plugins was tested in Kubernetes environment (requires Kubernetes 1.14+), and

the code can be run with any CSI enabled CO.

For details about configuration and deployment of the EDS CSI nasplugin, please refer the documentation.

For example usage of the CSI nasplugin, see examples below.

Before to go, you should have installed Sangfor EDS cluster.

You can get latest version of Sangfor EDS CSI nasplugin at docker hub by running docker CLI

download from `http://nas.sangfor.org:5000/sharing/cwq1mDxL2`

### if you download eds-csi plugin, you should load images manually

`$ tar -zxf eds-csi-v1.14.6-09718fd-202004301500.tar.gz && cd eds-csi-v1.14.6-09718fd-202004301500`

`$ docker load -i csi-provisioner.tar && docker load -i csi-node-driver-registrar.tar`

`$ docker load -i csi-nasplugin.tar.gz`

# Configuration Requirements

- Service Accounts with required RBAC permissions.

# Deployment

In this section, you will learn how to deploy the EDS CSI nasplugin and some necessary sidecar containers.

## Prepare EDS cluster & kubernetes cluster
| cluster     | version      |
| ------------| ------------ |
| Kubernetes  | 1.14+        |
| Sangfor EDS | 3.0.4+       |

## Deploy CSI plugins

### Get yaml file

Get yaml file from below links:

[csi-eds-rbac.yaml](https://github.com/evan37717/sangfor-eds-csi/blob/master/deploy/csi-eds-rbac.yaml)

[csi-eds-nasplugin-provisioner.yaml](https://github.com/evan37717/sangfor-eds-csi/blob/master/deploy/csi-eds-nasplugin-provisioner.yaml)

[csi-eds-nasplugin.yaml](https://github.com/evan37717/sangfor-eds-csi/blob/master/deploy/csi-eds-nasplugin.yaml)

### Create RBACs & sidecar containers

Deploy RBACs for sidecar containers and nasplugin:

   `$ kubectl create -f csi-eds-rbac.yaml`

Deploy CSI sidecar containers:

   `$ kubectl create -f csi-eds-nasplugin-provisioner.yaml`

### Create EDS CSI nasplugin

   `$ kubectl create -f csi-eds-nasplugin.yaml`

### Verify Pod
   `$ kubectl -nkube-system get pod -o wide |grep csi`
   
```
csi-nasplugin-4gzh7                     2/2     Running       0          5d10h   10.212.22.226   10.212.22.226   <none>           <none>
csi-nasplugin-4tmxw                     2/2     Running       0          5d10h   10.212.22.224   10.212.22.224   <none>           <none>
csi-nasplugin-s58cv                     2/2     Running       0          5d10h   10.212.22.225   10.212.22.225   <none>           <none>
csi-provisioner-0                       2/2     Running       0          5d10h   10.212.22.224   10.212.22.224   <none>           <none>
```
   
Congratulation! you have finished the deployment. Now you can use them to provide Sangfor EDS NAS service.

# Usage

In this section，you will learn how to provision NAS with Sangfor EDS CSI driver. 

Here will Assumes that you have installed Sangfor EDS Cluster.

## Preparation

To continue，make sure you have finish the Deployment part.

Login to you EDS dashboard, your dashboard address should be https://your_domain_ip/index
### Dashbord
![dashboard](https://github.com/evan37717/sangfor-eds-csi/blob/master/images/dashbord.png)
### Create file Storage
![create-nfs](https://github.com/evan37717/sangfor-eds-csi/blob/master/images/create-nfs.png)
### Configure NFS
![configure-nfs-gateway](https://github.com/evan37717/sangfor-eds-csi/blob/master/images/configure-nfs-gateway.png)

## Edit yaml for StorageClass
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: eds-nas-sp
mountOptions:
- nolock,tcp
- vers=3
parameters:
  vip: "10.212.22.213"
  volume: "csi-test"
  omuser: "admin"
  ompasswd: "123456"
  archiveOnDelete: "false"
provisioner: eds.csi.file.sangfor.com
reclaimPolicy: Delete
```
## Create storageclass

`$ kubectl create -f storageclass.yaml`

## Edit yaml for PersistentVolumeClaim
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: nas-csi-pvc-sp
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: eds-nas-sp
  resources:
    requests:
      storage: 20Gi
```
## Create pvc
`$ kubectl create -f pvc.yaml`

## Verify pv & pvc

Run kubectl check command

`$ kubectl get pv`
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
nas-7e419d5e-1c8b-11ea-a080-fefcfeaed50a   20Gi       RWO            Delete           Bound    default/nas-csi-pvc-sp   eds-nas-sp              5d9h
```
`$  kubectl get pvc`

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nas-csi-pvc-sp   Bound    nas-7e419d5e-1c8b-11ea-a080-fefcfeaed50a   20Gi       RWO            eds-nas-sp     5d9h
```

## Edite yaml for deploy
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nas
  labels:
    app: centos7
spec:
  selector:
    matchLabels:
      app: centos7
  template:
    metadata:
      labels:
        app: centos7
    spec:
      containers:
      - name: centos7
        image: centos:7.4.1708
        imagePullPolicy: IfNotPresent
        command: [ "/bin/bash", "-ce", "tail -f /dev/null" ]
        volumeMounts:
          - name: nas-csi-pvc-sp
            mountPath: "/data"
      volumes:
        - name: nas-csi-pvc-sp
          persistentVolumeClaim:
            claimName: nas-csi-pvc-sp
```
## Create deploy
`$ kubectl create -f deploy.yaml`

## Verify deploy
  `$ kubectl get pods`
```
  NAME                              READY   STATUS    RESTARTS   AGE
deployment-nas-57947c6cd9-vwt9f   1/1     Running   0          5d9h
```
