部署规划
10.0.0.99 master
10.0.0.44 node1
10.0.0.74 node2
10.0.0.121 node3 
10.0.0.72 node4 #data set
10.0.0.124 node5 #NFS
10.0.0.118 node6 #front&back
#k8s集群为master+node1-3
#k8s版本为1.16.2
#docker版本为19.03.12
#calico版本为v3.9.0
#k8s pod网段为192.168.0.0/16
#k8s service网段为172.16.0.0/16

#nfsserver端（node5）
#安装nfs
apt -y install nfs-kernel-server 
#配置nfs
mkdir /nfs
echo '/nfs *(rw,sync,no_root_squash)' >>/etc/exports
#启动nfs
systemctl enable rpcbind.service;systemctl restart  rpcbind.service
systemctl enable nfs-server.service;systemctl restart  nfs-server.service

#### 单节点不用执行这一步##############################
#除nfsserver端以外的其他节点
#安装nfsclient
apt -y install nfs-common
#测试nfs
showmount -e 10.0.0.124
mkdir /nfs && mount -t nfs 10.0.0.124:/nfs /nfs
 ###  ############################################# #####

#nfs对接k8s（此步骤仅在k8s master执行）
#创建nfs_plugin_deployment
cat > nfs_plugin_deployment.yaml << EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/duhao1027/nfs-client-provisioner:1.5
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: mynfs                 # 根据自己的名称来修改
            - name: NFS_SERVER
              value: 10.0.0.124        # NFS服务器所在的 ip
            - name: NFS_PATH
              value: /nfs/                  # 共享存储目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.0.0.124         # NFS服务器所在的 ip
            path: /nfs/                     # 共享存储目录
EOF
kubectl apply -f nfs_plugin_deployment.yaml



#创建rbac
cat > nfs_rbac.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kube-system
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: kube-system
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f nfs_rbac.yaml


#创建sc
cat > nfs_sc.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs
# 和 deployment 自定义 value 一致
provisioner: mynfs  
EOF

kubectl apply -f nfs_sc.yaml

#设置默认sc
kubectl patch storageclass nfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'


#测试k8s对接nfs是否成功
#pvc测试
cat > nfs_pvc_test.yaml << EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  accessModes:
    - ReadWriteMany
  # 指定刚才创建的 storage-class 的 metadata.name
  storageClassName: "nfs"
  resources:
    requests:
      storage: 2Gi
---
kind: Pod
apiVersion: v1
metadata:
  name: pvc-test
spec:
  containers:
  - name: test-busybox
    image: busybox:1.28
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
      # 和 pvc 的 metadata.name 一致
        claimName: pvc-test
EOF
kubectl apply -f nfs_pvc_test.yaml
#pvc变为Bound状态后，pvc-testpod也变为Completed状态 即为sc创建成功。

kubectl get pvc
kubectl get pv