---
title: 'Kubernetes 入門'
disqus: hackmd
---

Kubernetes 入門
===

## Table of Contents

[TOC]

## 準備
### Kind
> 
```gherkin=
kind create cluster --config multi-nodes.yaml
```
> kind/multi-nodes.yaml
```gherkin
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```
![截圖 2024-01-06 上午12.16.05](https://hackmd.io/_uploads/HkP73oBdp.png)
> 登入 control-plane
```gherkin=
docker exec -it kind-control-plane  /bin/bash
```

## 基礎
### 設定 Context(叢集)
>  cluster 是以 context 為單位，一個 context 有三個元件，分別是 User、Server、Certification  
>  通常會在 $HOME/.kube/config  
![截圖 2023-12-25 上午12.05.01](https://hackmd.io/_uploads/H1U6LCrPa.png)

>  也可以用環境變數 KUBECONFIG 方式設定
```gherkin=
export KUBECONFIG=/tmp/admin.conf
```
### 查看 context 
```gherkin=
kubectl config get-contexts 
```
![截圖 2023-12-25 上午12.03.09](https://hackmd.io/_uploads/SyfDLRHPT.png)

### 切換 context 
```gherkin=
kubectl config use-context mycontext
```
### 建立 namespace
```gherkin=
kubectl create namespace mynamespace
#或是
kubectl create ns mynamespace
```
### 刪除 namespace
```gherkin=
kubectl delete namespace mynamespace
#或是
kubectl delete ns mynamespace
```
### 產生 yaml 但不部署
```gherkin=
# -o yaml 輸出成 yaml 格式，--dry-run=client 不部署
kubectl run nginx --image=nginx --restart=Never --port=80 --dry-run=client -o yaml > nginx-pod.yaml
```
```gherkin=
# 執行 下列指令即可部署
kubectl apply -f nginx-pod.yaml
```
### 替換鏡像版本，方法 1
```gherkin=
kubectl set image pod/nginx nginx=nginx:1.7.1
```
> 觀察
```gherkin=
kubectl describe po nginx # you will see an event 'Container will be killed and recreated'
kubectl get po nginx -w # watch it
```
### 替換鏡像版本，方法 2
```gherkin=
# 編輯 pod/nginx 內容，將 - image: nginx 修改成 nginx:1.7.1 即可
kubectl edit pod/nginx
```
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-01-27T02:03:42Z"
  labels:
    run: nginx
  name: nginx
  namespace: default
  resourceVersion: "71316"
  uid: 89713655-2f7a-4a49-8ada-8627a5eefd63
spec:
  containers:
  - image: nginx # 修改成nginx:1.7.1 即可
    imagePullPolicy: Always
    name: nginx
    ports:
    - containerPort: 80
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-78shp
      readOnly: true
  dnsPolicy: ClusterFirst
```
## Pod (K8s 最小操作單元)
### 取得 pod 的細節
```gherkin=
kubectl describe po nginx
```
![截圖 2023-12-25 上午12.08.36](https://hackmd.io/_uploads/HJHqw0SPp.png)

### 取得 pod 的日誌
> 目前的日誌
```gherkin=
kubectl logs nginx
```
![截圖 2023-12-25 上午12.06.59](https://hackmd.io/_uploads/H1n4w0HPT.png)

> 損毀前或重啟前的日誌
```gherkin=
kubectl logs nginx -p
# or
kubectl logs nginx --previous
```
### 登入 pod
```gherkin=
kubectl exec -it nginx -- /bin/sh
```
### 部署並執行完指令即刻刪除
```gherkin=
# 此例為執行env，顯示 image busybox 的環境變數 
kubectl run busybox --image=busybox -it --rm --restart=Never -- env
```
### 一個 pod 裡面多容器
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx # container 1
    name: nginx
    resources: {}
  - args:
    - /bin/sh
    - -c
    - echo hello;sleep 3600 # busybox 啟動就結束，所以需要 spleep，才有時間進行驗證
    image: busybox # container 2
    name: busybox
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
> 驗證
```gherkin=
# -c 指定 container  
kubectl exec -it busybox -c busybox -- /bin/sh
ls
exit

# or you can do the above with just an one-liner
kubectl exec -it busybox -c busybox -- ls

```
## Workload Resource
### Deployment
> 透過指領產生 yaml 檔，不直接部署
```gherkin=
kubectl create deploy nginx --image=nginx --dry-run=client -o yaml
```
> yaml 檔內容如下
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

## Service
> 在此僅列出常用型態

### ClusterIP
```gherkin=
kubectl run nginx --image=nginx

kubectl expose pod/nginx --type=ClusterIP --port 80 --target-port 80 
```
#### 驗證方法 1. 執行下列指令
```gherkin=
kubectl run dnsutil --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it -- nslookup
```
#### 驗證方法 2.
> 部署 jessie-dnsutils 下列內容 (參考：https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
    command:
      - sleep
      - "infinity"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```
> 或執行
```gherkin=
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
```
> 驗證
```gherkin=
kubectl exec -i -t dnsutils -- nslookup nginx
```
```
# 取得內容如下，顯示格式 [service name].[namespace name].svc.cluster.local 
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	nginx.default.svc.cluster.local
Address: 10.100.2.163
```

### LoadBalancer
#### Minikube
```gherkin=
minikube tunnel
```
#### Kind上安裝 MetalLB (參考：https://kind.sigs.k8s.io/docs/user/loadbalancer/)
```gherkin=
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```
> 取得 docker 上的網路資訊，確定 load balancer 可分派 ip 範圍
```gherkin= 
docker network inspect -f '{{.IPAM.Config}}' kind
```
> 輸出範例
```
172.19.0.0/16
```
> 設定 IP Address Pool
```gherkin= 
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```
> 官方範例(參考用)
```
kubectl apply -f https://kind.sigs.k8s.io/examples/loadbalancer/metallb-config.yaml
```

### External Name (用來取得其他 namespace 的 ClusterIP Service)
> 新建 namespace ns1，並部署 nginx
```gherkin=
kubectl create ns ns1

kubectl run ns1-nginx --image=nginx -n ns1

kubectl expose pod/ns1-nginx --type=ClusterIP --port 80 --target-port 80 -n ns1
```
> 新建 namespace ns2，並部署 nginx
```gherkin=
kubectl create ns ns2

kubectl run ns2-nginx --image=nginx -n ns2

kubectl expose pod/ns2-nginx --type=ClusterIP --port 80 --target-port 80 -n ns2
```
#### 驗證方法 1.
> 執行下列指令
```gherkin=
kubectl run dnsutil --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it -- nslookup
```
#### 驗證方法 2.
> 或是，在 namespace ns1 中 部署 jessie-dnsutils 下列內容
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: ns1 # 部署在 n1
spec:
  containers:
  - name: dnsutils
    image: registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3
    command:
      - sleep
      - "infinity"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

> 驗證
```gherkin=
kubectl exec -i -t dnsutils -n ns1 -- nslookup ns1-nginx
```
```
# 取得內容如下，顯示格式 [service name].[namespace Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	ns1-nginx.ns1.svc.cluster.local
Address: 10.111.2.27
```
> 但無法取得 ns2 的 service ns2-nginx，因此需要在ns1部署此內容
> 
```gherkin=
apiVersion: v1
kind: Service
metadata:
  name: ns2-nginx
  namespace: ns1
spec:
  type: ExternalName
  externalName: ns2-nginx.ns2.svc.cluster.local
  ports:
    - name: http 
      port: 80
      targetPort: 80
```

```
# 取得內容如下，顯示格式 [service name].[namespace 
Server:		10.96.0.10
Address:	10.96.0.10#53

ns2-nginx.ns1.svc.cluster.local	canonical name = ns2-nginx.ns2.svc.cluster.local.
Name:	ns2-nginx.ns2.svc.cluster.local
Address: 10.110.109.186
```

## Volume
### ConfigMap

#### 兩種產生方法(可用於應用程式所需的參數設定)
> 從檔案讀取後寫入 --from-file 
```gherkin=
#先把參數寫入一個檔 
echo -e "var1=val1\nvar2=val2" > configMapFile.txt

kubectl create cm mycmfile --from-file=configMapFile.txt
```
```gherkin=
kubectl get configmap mycmfile -o yaml
```
```
apiVersion: v1
data:
  configMapFile.txt: |
    var1=val1
    var2=val2
kind: ConfigMap
metadata:
  creationTimestamp: "2024-01-04T01:50:49Z"
  name: mycmfile
  namespace: default
  resourceVersion: "74218"
  uid: 33f56c3c-8ec0-4d84-b3b1-f275ac082a49
```
> 從參數寫入 --from-literval
```gherkin=
kubectl create cm mycmliteral --from-literal=var1=val1 --from-literal=var2=val2
```
```gherkin=
kubectl get configMap mycmliteral -o yaml
```
```
apiVersion: v1
data:
  var1: val1
  var2: val2
kind: ConfigMap
metadata:
  creationTimestamp: "2024-01-04T15:31:14Z"
  name: mycmliteral
  namespace: default
  resourceVersion: "98321"
  uid: 68614916-f194-49ad-9c58-f4b0ede4f6d7
```

#### 有三種使用方法
> env(volume/configMap/pod-env.yaml)
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    env:
    - name: value # pod 裡面的變數名稱
      valueFrom:
        configMapKeyRef:
          name: mycmliteral # config map 名稱
          key: var1 # config map 裡的變數名稱
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>驗證
```gherkin=
kubectl exec -it nginx -- env # nginx 為 pod 名稱
```
```
root@nginx:/# env
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=nginx
PWD=/
PKG_RELEASE=1~bookworm
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
NJS_VERSION=0.8.2
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_VERSION=1.25.3
value=val1 <----------------------
_=/usr/bin/env
```
>envFrom(volume/configMap/pod-envFrom.yaml)
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    envFrom: # different than previous one, that was 'env'
    - configMapRef: # different from the previous one, was 'configMapKeyRef'
        name: mycmliteral # the name of the config map
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>驗證
```gherkin=
kubectl exec -it nginx -- env # nginx 為 pod 名稱
```
```
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
var1=val1 <----------------------
var2=val2 <----------------------
HOSTNAME=nginx
PWD=/
PKG_RELEASE=1~bookworm
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
NJS_VERSION=0.8.2
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_VERSION=1.25.3
_=/usr/bin/env
```
>volumes(volume/configMap/pod-volumes.yaml)
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes: # add a volumes list
  - name: myvolume # 後續 vloume mount 會用到
    configMap:
      name: mycmliteral # config map 名稱
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    volumeMounts: # your volume mounts are listed here
    - name: myvolume # 前面 pod.spec.volumes.name 指定的名稱，此處為 myvolume
      mountPath: /etc/lala # mount 進來的路徑
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>驗證
```gherkin=
kubectl exec -it nginx -- cat /etc/lala/ 
```
```
total 16
drwxrwxrwx 3 root root 4096 Jan  4 15:41 .
drwxr-xr-x 1 root root 4096 Jan  4 15:41 ..
drwxr-xr-x 2 root root 4096 Jan  4 15:41 ..2024_01_04_15_41_29.233495579
lrwxrwxrwx 1 root root   31 Jan  4 15:41 ..data -> ..2024_01_04_15_41_29.233495579
lrwxrwxrwx 1 root root   11 Jan  4 15:41 var1 -> ..data/var1
lrwxrwxrwx 1 root root   11 Jan  4 15:41 var2 -> ..data/var2
```
```gherkin=
kubectl exec -it nginx -- cat /etc/lala/var1 # nginx 為 pod 名稱
### 取得 val1
```

### Secrets
> 僅用 base64 編碼，不算是真的加密，結果依然不安全
#### 兩種產生方式 
> 從參數寫入 --from-literval
```gherkin=
kubectl create secret generic secret-from-literal --from-literal=username=admin
```
> 查看內容
```gherkin=
kubectl get secrte secret-from-literal -o yaml
```
```
apiVersion: v1
data:
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2024-01-05T01:47:47Z"
  name: secret-from-literal
  namespace: default
  resourceVersion: "154332"
  uid: 57476e5b-23f9-4d33-afa2-49b24e69bb9c
type: Opaque
```

> 從檔案寫入 --from-file 

```gherkin=
echo -n admin > username

kubectl create secret generic user-secret --from-file=username
```
> 查看內容
```gherkin=
kubectl get secrte user-secret -o yaml
```
```
apiVersion: v1
data:
  username: YWRtaW4= # 此為 admin 被 base64 encode 後的結果
kind: Secret
metadata:
  creationTimestamp: "2023-01-26T15:54:38Z"
  name: user-secret
  namespace: default
  resourceVersion: "67002"
  uid: ba0e0a7e-cf92-4200-98cb-14cdbbf6b2b8
type: Opaque
```
> 驗證
```gherkin=
kubectl get secret user-secret -o yaml

echo -n YWRtaW4= | base64 -d # YWRtaW4= 為存在secret 裡面的值， decode 之後得到 admin
```
> 其他驗證方法
```gherkin=
# 用 --jsonpath
kubectl get secret user-secret -o jsonpath='{.data.username}' | base64 -d  # on MAC it is -D

# 用 --template
kubectl get secret user-secret --template '{{.data.username}}' | base64 -d  # on MAC it is -D

# 用 jq
kubectl get secret user-secret -o json | jq -r .data.username | base64 -d  # on MAC it is -D

```
#### 以 volume 的方式取用 secret
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes: # specify the volumes
  - name: foo # 後續會用到的名稱
    secret: # we want a secret
      secretName: user-secret # name of the secret - this must already exist on pod creation
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    volumeMounts: # our volume mounts
    - name: foo # 上面 volume 指定的名稱
      mountPath: /etc/foo #volume mount 的路徑
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
> 驗證
```gherkin=
k exec pod/nginx -it -- cat /etc/foo/username
```
```gherkin=
#得到原內容 
admin
```
### emptyDir 
> 可用於容器的資料暫存
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: registry.k8s.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi
```
> 或同一Pod裡面的不同容器資料交換
```gherkin=
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  volumes:
    - name: html
      emptyDir: {}
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
      - name: html
        mountPath: /usr/share/nginx/html
  - name: alpine
    image: alpine
    volumeMounts:
      - name: html
        mountPath: /html
    command: [ "/bin/sh", "-c" ]
    args:
      - while true; do
        echo $(hostname) $(date) >> /html/index.html;
        sleep 10;
        done
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```





###### tags: `Kubernetes` `Documentation`
