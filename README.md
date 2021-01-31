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

### Deploy MariaDB
``` 
pod:
  replicas:
    server: 3
    ingress: 3
```# openstack-rook-k8s
