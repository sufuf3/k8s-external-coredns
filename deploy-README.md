# Deploy Document

## 部署流程
### 1. 部署 Ingress Controller
KaaS install 原本就可以設定要不要安裝 [ingress controller](https://gitlab.com/itri-opteam/KaaS-install/blob/master/KaaS-ansible/group_vars/all.yml#L79)  
```sh
$ kubectl get all -n kube-system -l k8s-app=nginx-ingress
NAME                                            READY     STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-6579c78869-44wf9   1/1       Running   2          20d

NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.111.167.65   100.67.151.8   80:32583/TCP,443:30004/TCP   20d

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller   1         1         1            1           20d

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/nginx-ingress-controller-6579c78869   1         1         1         20d

$ kubectl -n ingress-nginx get all
NAME                                            READY     STATUS             RESTARTS   AGE
pod/default-http-backend-5c6d95c48-9q4tv        1/1       Running            2          20d
pod/nginx-ingress-controller-6c9fcdf8d9-26rqk   0/1       CrashLoopBackOff   1744       20d

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/default-http-backend   ClusterIP   10.106.95.79   <none>        80/TCP    20d

NAME                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/default-http-backend       1         1         1            1           20d
deployment.apps/nginx-ingress-controller   1         1         1            0           20d

NAME                                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/default-http-backend-5c6d95c48        1         1         1         20d
replicaset.apps/nginx-ingress-controller-6c9fcdf8d9   1         1         0         20d
```

在這邊請記住 ingress-nginx 的 external IP: 100.67.151.8 。
```sh
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.111.167.65   100.67.151.8   80:32583/TCP,443:30004/TCP   20d
```
### 2. Create CoreDNS
1. get repo

```sh
git clone https://github.com/sufuf3/k8s-external-coredns.git
cd k8s-external-coredns/coredns
```

2. deploy yml files

```sh
kubectl create -f rbac-sa.yml
kubectl create -f rbac.yml
kubectl create -f cm.yml
kubectl create -f rbac-deploy.yml
kubectl create -f svc-lb-udp.yml
kubectl create -f svc-lb-tcp.yml
kubectl create -f etcd-deploy.yml
kubectl create -f etcd-svc.yml
```

3. check
```sh
$ kubectl get all -n kube-system -l k8s-app=kube-dns
NAME                            READY     STATUS    RESTARTS   AGE
pod/coredns-78666c95bd-2w2hd    1/1       Running   0          25m
pod/coredns-78666c95bd-6cnvk    1/1       Running   0          25m

NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)         AGE
service/kube-coredns       LoadBalancer   10.105.132.174   100.67.151.8   53:31663/UDP    9m
service/kube-coredns-tcp   LoadBalancer   10.98.151.162    100.67.151.8   53:31569/TCP    9m

NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns    2         2         2            2           25m

NAME                                  DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-78666c95bd    2         2         2         25m

$ kubectl get all -n kube-system -l k8s-app=kube-dns-etcd
NAME                               READY     STATUS    RESTARTS   AGE
pod/coredns-etcd-9c44584cd-c2zw2   1/1       Running   0          7m

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/coredns-etcd   ClusterIP   10.100.39.87   <none>        2379/TCP,2380/TCP   6m

NAME                           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns-etcd   1         1         1            1           7m

NAME                                     DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-etcd-9c44584cd   1         1         1         7m
```

如果有 kube-dns ，把他移掉
```sh
kubectl delete --namespace=kube-system deployment kube-dns
```

4. dig 測試

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
cluster.local.      300 IN  SOA ns.dns.cluster.local. hostmaster.cluster.local. 1533283375 7200 1800 86400 30

```

### 3. Create externalDNS

