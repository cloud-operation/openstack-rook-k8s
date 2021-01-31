# Openstack on Kubernetes

#### Helm Install
```
# mkdir helm
# cd helm
$ curl -LO https://git.io/get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh
$ helm init

Follow kubernetes version helm 2 to patch.

# kubectl create serviceaccount --namespace kube-system tiller
# kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
# kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'


helm repo update
helm repo list
```
## NginX Ingress install by Helm
```
helm install --name nginx-ingress  --namespace kube-system stable/nginx-ingress --set controller.service.type=NodePort

kubectl -n kube-system edit deploy nginx-ingress-controller
...
spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=kube-system/nginx-ingress-default-backend
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --enable-ssl-passthrough # Add this flag
        - --configmap=kube-system/nginx-ingress-controller
```

## argocd installation part
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
watch kubectl get all -n argocd
kubectl get all -n argocd
kubectl get ing -n argocd
kubectl get ing -n argocd -o yaml
kubectl get pod -n argocd
kubectl get svc -n kube-system
```
## argocd ingress controller
```
cat << EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.brilliant.com.bd
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: https
        path: /
EOF

kubectl apply -f  argocd-ingress.yaml
```

## Haproxy
```
listen ingress_controller
  bind 118.67.212.7:80
  balance  roundrobin
  mode tcp
  server kube-con-1 192.168.13.3:30139 check inter 2000 rise 2 fall 5
  server kube-con-2 192.168.13.4:30139 check inter 2000 rise 2 fall 5
  server kube-con-3 192.168.13.5:30139 check inter 2000 rise 2 fall 5

listen ingress_controller_secure
  bind 118.67.212.7:443
  balance  roundrobin
  mode tcp
  server kube-con-1 192.168.13.3:30808 check inter 2000 rise 2 fall 5
  server kube-con-2 192.168.13.4:30808 check inter 2000 rise 2 fall 5
  server kube-con-3 192.168.13.5:30808 check inter 2000 rise 2 fall 5
root@haproxy:~#
```

# >>>>> Openstack Install on Kubernetes Start <<<<


### Clone  both openstack-helm and openstack-helm-infra
```
#    mkdir {git,openstack-helm}
#    cd openstack-helm
#    git clone https://github.com/openstack/openstack-helm.git
#    git clone https://github.com/openstack/openstack-helm-infra.git
```

### On Masternode clone  Compile and package openstack helm charts for argocd
```
#    helm serve & 
#    cd  openstack-helm-infra
#    make helm-toolkit
#    make ingress
#    make {openvswitch,mariadb,libvirt,rabbitmq,memcached}
#    cd ../openstack-helm
#    make {keystone,horizon,glance,cinder,nova,placement,neutron,heat} 
```

### Prepare and push helm charts to git
```
# git clone https://github.com/cloud-operation/openstack-rook-k8s.git
#    cp  ~/openstack-helm/openstack-helm/*.tgz ~/git/reponame/
#    cp  ~/openstack-helm/openstack-helm-infra/*.tgz ~/git/reponame/
#    cd ~/git/reponame/ 
#    tar -zxvf <tarfile>.tgz  # Repeat this for all packages

#    rm *.tgz

#    git add .
#    git commit -m "Adding openstack helm chart to repo"
#    git push
#    git status
```

### Preparing ArgoCD for openstack Deployment
1. Goto Setting & Click on Repositoring & Add Github repo using https
2. Goto Setting & Click on Project & Create a new project 

### Create namespace
```
#    kubectl create ns openstack
```

### Annotate kubernetes nodes for Ingress
```
# kubectl label nodes controller{1..3}.brilliant.com.bd openstack-helm-node-class=primary

# kubectl label nodes controller{1..3}.brilliant.com.bd openstack-control-plane=enabled

# kubectl describe node controller1.brilliant.com.bd
```
### Create Openstack-Ingress & Updates values.yaml as below by argoCD application
```
pod:
  replicas:
    ingress: 2
    error_page: 2
```

### Create Storage Class for volume.
```
# cat storageclass.yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: osp-pv
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
    # Disallow setting pool with replica 1, this could lead to data loss without recovery.
    # Make sure you're *ABSOLUTELY CERTAIN* that is what you want
    requireSafeReplicaSize: true
    # gives a hint (%) to Ceph in terms of expected consumption of the total cluster capacity of a given pool
    # for more info: https://docs.ceph.com/docs/master/rados/operations/placement-groups/#specifying-expected-pool-size
    #targetSizeRatio: .5
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: general
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
    # clusterID is the namespace where the rook cluster is running
    # If you change this namespace, also change the namespace below where the secret namespaces are defined
    clusterID: rook-ceph # namespace:cluster

    # If you want to use erasure coded pool with RBD, you need to create
    # two pools. one erasure coded and one replicated.
    # You need to specify the replicated pool here in the `pool` parameter, it is
    # used for the metadata of the images.
    # The erasure coded pool must be set as the `dataPool` parameter below.
    #dataPool: ec-data-pool
    pool: osp-pv

    # (optional) mapOptions is a comma-separated list of map options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # mapOptions: lock_on_read,queue_depth=1024

    # (optional) unmapOptions is a comma-separated list of unmap options.
    # For krbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd/#kernel-rbd-krbd-options
    # For nbd options refer
    # https://docs.ceph.com/docs/master/man/8/rbd-nbd/#options
    # unmapOptions: force

    # RBD image format. Defaults to "2".
    imageFormat: "2"

    # RBD image features. Available for imageFormat: "2". CSI RBD currently supports only `layering` feature.
    imageFeatures: layering

    # The secrets contain Ceph admin credentials. These are generated automatically by the operator
    # in the same namespace as the cluster.
    csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
    csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
    csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
    csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
    csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster
    # Specify the filesystem type of the volume. If not specified, csi-provisioner
    # will set default as `ext4`. Note that `xfs` is not recommended due to potential deadlock
    # in hyperconverged settings where the volume is mounted on the same node as the osds.
    csi.storage.k8s.io/fstype: ext4
# uncomment the following to use rbd-nbd as mounter on supported nodes
# **IMPORTANT**: If you are using rbd-nbd as the mounter, during upgrade you will be hit a ceph-csi
# issue that causes the mount to be disconnected. You will need to follow special upgrade steps
# to restart your application pods. Therefore, this option is not recommended.
#mounter: rbd-nbd
allowVolumeExpansion: true
reclaimPolicy: Delete
```

### Deploy MariaDB
``` 
pod:
  replicas:
    server: 3
    ingress: 3
```
### Deploy RabbitMQ
``` 
Deploy RabbitMQ with default settings. No changes need in values.yaml file
```
### Deploy Memcached
```
Deploy RabbitMQ with default settings. No changes need in values.yaml file
```
### Deploy Keystone
#### >>>keystone deployment-api.yaml "subPath" spelling mistake correction file: "subpath: access_rules.json" to subPath: access_rules.json<<<

``` 
pod:
  replicas:
    api: 3
```
### Deploy Horizon
``` 
Deploy Horizon with default settings. No changes need in values.yaml file
```
### Horizon Ingress
##### create ingress rule to access horizon. create a file name: horizon-ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: horizon-ingress
  namespace: openstack
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: horizon.brilliant.com.bd
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: horizon-int
            port:
              number: 80
```
### Deploy Glance
in values.yaml file update storage as below:
```
storage: pvc
volume.size: 5Gi
```
``` 
pod:
  replicas:
    api: 1
    registry: 2
```
### Deploy Cinder and connect with external CEPH
#### collect key ring from ceph and make base64
```
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# cat /etc/ceph/keyring
[client.admin]
key = AQC4Ww1gMUHSNxAAVHykuyyDLEOkBnLMsePuWQ==
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]#
```
#### Base64 encode
```
# echo "AQC4Ww1gMUHSNxAAVHykuyyDLEOkBnLMsePuWQ==" | base64
QVFDNFd3MWdNVUhTTnhBQVZIeWt1eXlETEVPa0JuTE1zZVB1V1E9PQo=
```

#### Now create secret:
```
# cat ceph-admin-keyring-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: "pvc-ceph-client-key"
  namespace: openstack
type: kubernetes.io/rbd
data:
  key: QVFDNFd3MWdNVUhTTnhBQVZIeWt1eXlETEVPa0JuTE1zZVB1V1E9PQo=
```
#### configMap
##### Collect ceph host network mon IP
```
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# cat /etc/ceph/ceph.conf
[global]
mon_host = 192.168.13.4:6789,192.168.13.5:6789,192.168.13.3:6789

[client.admin]
keyring = /etc/ceph/keyring
```
#### Now create configMap
```
# cat cindar-configmp.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-etc
  namespace: openstack
data:
  ceph.conf: |
    [global]
    mon_host = 192.168.13.3:6789,192.168.13.4:6789,192.168.13.5:6789
```
#### 
```
# kubectl create -f ceph-admin-keyring-secret.yaml
# kubectl create -f cindar-configmp.yaml

```

#### Now deploy cinder from argocd
```
pod:
  replicas:
    api: 2
    volume: 1
    scheduler: 1
    backup: 1
```
#### Now login to horizon dashboard and create a volume. Check that volume create in ceph by below ceph command:
```
[root@rook-ceph-tools-77bf5b9b7d-skl4t /]# rbd ls cinder.volumes
6e14f4ae-79a5-4562-8737-760d9b62bb44

```

### Reference docs:
