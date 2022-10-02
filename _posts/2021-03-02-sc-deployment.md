---
layout: post
title: 常用(mariadb/MongoDB/Redis)StorageClass部署
subtitle: 
categories: K8S应用部署
tags: [Kubernetes-App-Deploy]
---



# 1.NFS部署


```
1.安装服务
# yum install nfs-utils rpcbind

2.创建数据目录 
# mkdir -p /data1/nfsdata 

3.在/etc/exports添加以下内容
/data1/nfsdata 10.168.0.0/24(rw,sync,no_root_squash)

4.启动服务
# service nfs start
 
```
# 2.nfs-provisioner部署

## 2.1 安装nfs-client-provisioner
```
# for file in class.yaml deployment.yaml rbac.yaml test-claim.yaml ; do wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/$file ; done
# kubectl apply -f rbac.yaml
# kubectl apply -f class.yaml 
# kubectl apply -f deployment.yaml 
```
## 2.2 查看运行client pod
```
# kubectl get pod
NAME                                    READY   STATUS      RESTARTS   AGE
nfs-client-provisioner-675667f4-fdtr8   1/1     Running     0          19h
```

## 2.3 查看storageclass
```
# kubectl get storageclasses.storage.k8s.io 
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  10m
```



# 3.部署MariaDB（单机）

## 3.1 创建pvc
```yaml
apiVersion: v1 
kind: PersistentVolumeClaim 
metadata: 
  name: mariadbpvc 
 # namespace: iam
spec: 
  storageClassName: managed-nfs-storage
  accessModes: 
    - ReadWriteMany 
  resources: 
    requests: 
      storage: 5Gi
```

## 3.2 创建deployement和service

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: mariadb-secret
type: Opaque
data:
  mariadb-root-password: MTIzNDU2 # echo -n '123456'|base64

---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
  namespace: default
  labels:
    app: mariadb
spec:
  ports:
    - name:
      protocol: TCP
      port: 3306
      targetPort: 3306
      nodePort: 32306
  selector:
    app: mariadb
  type: NodePort
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-sts
spec:
  serviceName: "mariadb-service"
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:latest
        ports:
        - containerPort: 3306
          name: mariadb-port
        env:
        - name: MARIADB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secret
              key: mariadb-root-password
        volumeMounts:
        - name: mariadb-data
          mountPath: /var/lib/mysql/
      volumes: 
        - name: mariadb-data 
          persistentVolumeClaim: 
            claimName: mariadbpvc     
```




# 4. 部署MongoDB副本集

## 4.1 生成repl的key
```
# openssl rand -base64 741 > ./key.txt
# kubectl create secret generic mongodb-replica-sets-key --from-file=internal-auth-mongodb-keyfile=./key.txt
```
## 4.2 部署三个MongoDB的statefulset

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-cluster
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-cluster
spec:
  serviceName: mongodb-cluster
  replicas: 3
  selector:
    matchLabels:
      role: mongo
      environment: produce
      replicaset: MainRepSet
  template:
    metadata:
      labels:
        role: mongo
        environment: produce
        replicaset: MainRepSet
    spec:
      containers:
      - name: mongodb-container
        image: mongo:4.2.22
        command:
        - "numactl"
        - "--interleave=all"
        - "mongod"
        - "--bind_ip"
        - "0.0.0.0"
        - "--replSet"
        - "MainRepSet"
        - "--auth"
        - "--clusterAuthMode"
        - "keyFile"
        - "--keyFile"
        - "/etc/secrets-volume/internal-auth-mongodb-keyfile"
        - "--setParameter"
        - "authenticationMechanisms=SCRAM-SHA-1"
        resources:
          requests:
            cpu: 0.5
            memory: 500Mi
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: secrets-volume
          readOnly: true
          mountPath: /etc/secrets-volume
        - name: mongodb-persistent-storage-claim
          mountPath: /data/db
      volumes:
      - name: secrets-volume
        secret:
          secretName: mongodb-replica-sets-key
          defaultMode: 256
  volumeClaimTemplates:
  - metadata:
      name: mongodb-persistent-storage-claim
      #annotations:
      #  volume.beta.kubernetes.io/storage-class: "standard"
    spec:
      storageClassName: "managed-nfs-storage"
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

## 4.3 查看自动生成的pv

```shell
# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                        STORAGECLASS          REASON   AGE
pvc-223059e6-08f3-4a57-a3cb-dacfd39ff6b7   10Gi       RWO            Delete           Bound    default/mongodb-persistent-storage-claim-mongodb-cluster-1   managed-nfs-storage            71s
pvc-35d350c2-79d0-4e34-afb0-38737f9eaac8   10Gi       RWO            Delete           Bound    default/mongodb-persistent-storage-claim-mongodb-cluster-0   managed-nfs-storage            7m33s
pvc-43cd097c-1d90-4977-9802-71277d696814   10Gi       RWO            Delete           Bound    default/mongodb-persistent-storage-claim-mongodb-cluster-2   managed-nfs-storage            26s
```

## 4.4 查看生成的pvc

```shell
# kubectl get pvc
NAME                                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
mongodb-persistent-storage-claim-mongodb-cluster-0   Bound    pvc-35d350c2-79d0-4e34-afb0-38737f9eaac8   10Gi       RWO            managed-nfs-storage   8m36s
mongodb-persistent-storage-claim-mongodb-cluster-1   Bound    pvc-223059e6-08f3-4a57-a3cb-dacfd39ff6b7   10Gi       RWO            managed-nfs-storage   2m14s
mongodb-persistent-storage-claim-mongodb-cluster-2   Bound    pvc-43cd097c-1d90-4977-9802-71277d696814   10Gi       RWO            managed-nfs-storage   89s
```
## 4.5 集群初始化

```shell

##### ----初始化副本集-------
# kubectl exec -it pod/mongodb-cluster-0 -- bash
# mongo
> use admin
> rs.initiate({_id: "MainRepSet", version: 1, members: [
      { _id: 0, host : "mongodb-cluster-0.mongodb-cluster.default.svc.cluster.local:27017" },
      { _id: 1, host : "mongodb-cluster-1.mongodb-cluster.default.svc.cluster.local:27017" },
      { _id: 2, host : "mongodb-cluster-2.mongodb-cluster.default.svc.cluster.local:27017" }
]});

##### ----创建管理员用户-------

db.getSiblingDB("admin").createUser({
      user : "root",
      pwd  : "123456",
      roles: [ { role: "root", db: "admin" } ]
});

##### ----验证创建管理员后的权限-------
MainRepSet:PRIMARY> rs.status()
{
        "operationTime" : Timestamp(1664188420, 4),
        "ok" : 0,
        "errmsg" : "command replSetGetStatus requires authentication",
        "code" : 13,
        "codeName" : "Unauthorized",
        "$clusterTime" : {
                "clusterTime" : Timestamp(1664188444, 1),
                "signature" : {
                        "hash" : BinData(0,"KW4xheFxYXAcPRgkHJwAYUehvMU="),
                        "keyId" : NumberLong("7147634425965051908")
                }
        }
}
MainRepSet:PRIMARY>

# mongo --host mongodb-cluster-0.mongodb-cluster.default.svc.cluster.local  --port 27017 -uroot -p123456

```

# 问题解决

## 错误1

```
# kubectl logs nfs-client-provisioner-675667f4-fdtr8  -n default

controller.go:1004] provision "default/mariadbpvc" class "managed-nfs-storage": unexpected error getting claim reference: selfLink was empty, can't make reference
```
## 修复方法

```
修改三个master阶段的api-server配置
vim /etc/kubernetes/manifests/kube-apiserver.yaml 
 - --feature-gates=RemoveSelfLink=false
```

## 错误2

```
unable to create directory to provision new pv mkdir /persistentvolumes/ permission denied
```
## 修复方法
```
在nfs服务器端把数据目录的权限修改为777
```

## 错误2
```
# kubectl logs mariadb-sts-0 
2022-09-26 09:27:35+00:00 [Note] [Entrypoint]: Entrypoint script for MariaDB Server 1:10.9.3+maria~ubu2204 started.
chown: changing ownership of '/var/lib/mysql/': Operation not permitted
```

## 修复方法
```
nfs服务器增加no_root_squash配置
cat /etc/exports
/data1/nfsdata 10.168.0.0/24(rw,sync,no_root_squash)
```
