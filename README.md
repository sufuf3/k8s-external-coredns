# Deploy Kubernetes ExternalDNS for CoreDNS
Collect the deployment file to help me setup CoreDNS and ExternalDNS on top of Kubernetes.

![](https://i.imgur.com/rZ8PoFC.png)

## Steps
### 環境
```sh
$ kubectl get node
NAME      STATUS    ROLES     AGE       VERSION
coco2     Ready     master    36d       v1.10.2
coco3     Ready     master    36d       v1.10.2
coco4     Ready     master    36d       v1.10.2
coco5     Ready     <none>    36d       v1.10.2
coco6     Ready     <none>    36d       v1.10.2
```

### 0. 安裝 ingress
```sh
$ kubectl get all -n kube-system -l k8s-app=nginx-ingress
NAME                                            READY     STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-6579c78869-44wf9   1/1       Running   2          35d

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.111.167.65   100.67.151.8   80:32583/TCP,443:30004/TCP   35d

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   1         1         1            1           35d

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/nginx-ingress-controller-6579c78869   1         1         1         35d
```
```sh
$ kubectl -n ingress-nginx get all
NAME                                            READY     STATUS             RESTARTS   AGE
pod/default-http-backend-5c6d95c48-9q4tv        1/1       Running            2          36d
pod/nginx-ingress-controller-6c9fcdf8d9-26rqk   0/1       CrashLoopBackOff   5995       36d

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/default-http-backend   ClusterIP   10.106.95.79   <none>        80/TCP    36d

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-http-backend       1         1         1            1           36d
deployment.apps/nginx-ingress-controller   1         1         1            0           36d

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/default-http-backend-5c6d95c48        1         1         1         36d
replicaset.apps/nginx-ingress-controller-6c9fcdf8d9   1         1         0         36d
```

在這邊請記住 ingress-nginx 的 external IP: 100.67.151.8 。  
```sh
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.111.167.65   100.67.151.8   80:32583/TCP,443:30004/TCP   35d
```

### 1. Create CoreDNS

- get repo
```sh
git clone https://github.com/sufuf3/k8s-external-coredns.git
cd k8s-external-coredns
```

- deploy yml files
```sh
kubectl create -f coredns/rbac/rbac-sa.yml
kubectl create -f coredns/rbac/rbac.yml
kubectl create -f coredns/cm.yml
kubectl create -f coredns/rbac/rbac-deploy.yml
kubectl create -f coredns/svc-lb-tcp.yml
kubectl create -f coredns/svc-lb-udp.yml
```
- check
```sh
$ kubectl get all -n kube-system -l k8s-app=kube-dns
NAME                           READY     STATUS    RESTARTS   AGE
pod/coredns-75fcbf7b7f-8bgcx   1/1       Running   0          2m
pod/coredns-75fcbf7b7f-ghsr9   1/1       Running   0          2m

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
service/kube-coredns-tcp   LoadBalancer   10.108.48.108   100.67.151.8   53:30450/TCP   1m
service/kube-coredns-udp   LoadBalancer   10.97.153.126   100.67.151.8   53:31933/UDP   1m

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2         2         2            2           2m

NAME                                 DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-75fcbf7b7f   2         2         2         2m
```

> 如果有 kube-dns ，把他移掉 `kubectl delete --namespace=kube-system deployment kube-dns`  



- dig 測試

dig 是 DNS lookup 的工具。語法是  
```sh
SYNOPSIS
       dig [@server] [-b address] [-c class] [-f filename] [-k filename] [-m] [-p port#] [-q name] [-t type] [-v] [-x addr] [-y [hmac:]name:key] [-4] [-6] [name]
           [type] [class] [queryopt...]
```

所以 `dig @100.67.151.8 SOA cluster.local +noall +answer +time=2 +tries=1` 是從 DNS server 100.67.151.8 這台查詢。查詢的 type 是 SOA ，要查詢的 resource record 名稱是 cluster.local 。   
`+noall` (Clear all display flags), `+answer`(Display the answer section of a reply.), `+time=T`(Sets the timeout for a query to T seconds.), `+tries=T`(Sets the number of times to try UDP queries to server to T instead of the default)  
TTL 是 300 秒，所以如果有任何更改，需要等 300 秒(快取時間)  
  
```sh
$ dig @100.67.151.8 SOA cluster.local +noall +answer +time=2 +tries=1

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> @100.67.151.8 SOA cluster.local +noall +answer +time=2 +tries=1
; (1 server found)
;; global options: +cmd
cluster.local.          300     IN      SOA     ns.dns.cluster.local. hostmaster.cluster.local. 1534569563 7200 1800 86400 30
```

## 2. Create externalDNS
