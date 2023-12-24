---
title: 'Kubernetes 入門'
disqus: hackmd
---

Kubernetes 入門
===

## Table of Contents

[TOC]

## 基礎
### 設定 Context(叢集)
>  cluster 是以 context 為單位，一個 context 有三個元件，分別是 User、Server、Certification  
>  通常會在 $HOME/.kube/config  
>  也可以用環境變數 KUBECONFIG 方式設定
```gherkin=
export KUBECONFIG=/tmp/admin.conf
```
### 查看 context 
```gherkin=
kubectl config get-contexts 
```
![截圖 2023-12-24 下午8.26.37](https://hackmd.io/_uploads/SkaY7iSwT.png)

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
### 取得 pod 的日誌
> 目前的日誌
```gherkin=
kubectl logs nginx
```
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
### ReplicaSet

### Deployment



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
### NodePort(少用)


## ConfigMap

### 有兩種產生方法
> 可用於應用程式所需的參數設定
> create 1. --from-file 
```gherkin=
#先把參數寫入一個檔 
echo -e "var1=val1\nvar2=val2" > configMapFile.txt

kubectl create cm mycmfile --from-file=configMapFile.xtx
```
> create config map 2 --from-literval
```gherkin=
kubectl create cm mycmliterval --from-literval=var1=val1 --from-literval=var2=val2
```

### 有三種使用方法
> env
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
    - name: mycmliterval # pod 裡面的變數名稱
      valueFrom:
        configMapKeyRef:
          name: mycmliterval # config map 名稱
          key: va1 # config map 裡的變數名稱
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>驗證
```gherkin=
kubectl exec -it nginx -- env | grep val1 # nginx 為 pod 名稱
```
>envFrom
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
        name: mycmliterval # the name of the config map
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
>驗證
```gherkin=
kubectl exec -it nginx -- env | grep val1 # nginx 為 pod 名稱
```
>volumes
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
      name: mycmliterval # config map 名稱
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
kubectl exec -it nginx -- cat /etc/lala/val1 # nginx 為 pod 名稱
```

## Secrets
> 僅用md5編碼，結果依然不安全
###  create --from-file 
```gherkin=
echo -n admin > username

kubectl create secret generic user-secret --from-file=username
```
> 查看內容
```gherkin=
kubectl edit secrte user-secret
```

```gherkin=
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
kubectl get secret mysecret2 -o yaml

echo -n YWRtaW4= | base64 -d # YWRtaW4= 為存在secret 裡面的值， decode 之後得到 admin
```
> 其他驗證方法
```gherkin=
# 用 --jsonpath
kubectl get secret mysecret2 -o jsonpath='{.data.username}' | base64 -d  # on MAC it is -D

# 用 --template
kubectl get secret mysecret2 --template '{{.data.username}}' | base64 -d  # on MAC it is -D

# 用 jq
kubectl get secret mysecret2 -o json | jq -r .data.username | base64 -d  # on MAC it is -D

```
### 以 volume 的方式取用 secret
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
## Storage
### PV

### PVC


###### tags: `Kubernetes` `Documentation`
