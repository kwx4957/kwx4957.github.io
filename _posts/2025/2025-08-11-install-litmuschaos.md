---
title: LitmusChaos 설치하기 
categories: [opensource,litmuschaos]
tags: [k8s,litmuschaos,chaosengineering,docs]
date: 2025-08-11 00:48 +0900
---

m2(arm)에서의 minikube 설치 및 litmusChaos 설치 과정입니다. Litmus의 버전은 3.91 버전입니다.

### minikube 설치

우선 homebrew를 사용하여 minikube를 설치했습니다. minikube를 설치하기 전에 앞서 해당 조건들이 충족되어야 합니다.

-   2개 이상의 CPU
-   20기가 이상의 디스크
-   2기가 이상의 메모리 용량
-   컨테이너 또는 VM

brew를 통해 minikube 설치 후 minikube를 시작해주었습니다. 만일 최신 k8s api-version이 아닌 다른 버전의 api를 사용하기를 원한다면 `minikube start --kubernetes-version=v1.29` 옵션을 통해 실행할 수 있습니다.

```sh
brew install minikube  
minikube start
```

minikube 동작 확인 
```sh
kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   86m   v1.30.0
```

### LitmusChaos 설치

LitmusChaos는 2가지 방식으로 설치하실 수 있습니다. helm과 kubectl입니다. helm이 좀 더 간편하다고 생각했고 관리하기 편하다고 생각하여 helm을 선택하게 되었습니다. mac arm의 경우에는 mongodb의 arm 기술적인 지원하지 않기에 다음 방식으로 설치해주었습니다.

```sh
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update 
helm repo list  
kubectl create ns litmus  

# ARM인 경우
helm install chaos litmuschaos/litmus \
  --namespace=litmus \
  --set portal.frontend.service.type=NodePort \
  --set mongodb.image.registry=ghcr.io/zcube \  
  --set mongodb.image.repository=bitnami-compat/mongodb \
  --set mongodb.image.tag=6.0.5

# amd64
helm install chaos litmuschaos/litmus --namespace=litmus \
  --set portal.frontend.service.type=NodePort
```

helm으로 설치하는 과정에서 auth-server와 server가 init상태에서 변하지 않을 경우 다음 명령어를 실행시켜주세요.

```sh 
helm uninstall -n litmus chaos  
helm repo remove litmuschaos  
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/

helm install chaos litmuschaos/litmus \
  --namespace=litmus \
  --set portal.frontend.service.type=NodePort \
  --set mongodb.image.registry=ghcr.io/zcube \
  --set mongodb.image.repository=bitnami-compat/mongodb \ 
  --set mongodb.image.tag=6.0.5

NAME: chaos
LAST DEPLOYED: Thu Jul 18 19:46:05 2024
NAMESPACE: litmus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing litmus 😀

Your release is named chaos and it's installed to namespace: litmus.
```

파드 실행 여부 조회
```sh
kubectl get pods -n litmus
NAME                                       READY   STATUS    RESTARTS        AGE
chaos-litmus-auth-server-8499fc4cf-6sf6h   1/1     Running   0               10m
chaos-litmus-frontend-6fd8ff8b48-z8pnw     1/1     Running   0               10m
chaos-litmus-server-5484b76bc-79clm        1/1     Running   0               10m
chaos-mongodb-0                            1/1     Running   0               10m
chaos-mongodb-1                            1/1     Running   1 (5m37s ago)   9m28s
chaos-mongodb-2                            1/1     Running   0               9m16s
chaos-mongodb-arbiter-0                    1/1     Running   0               10m

kubectl get svc -n litmus
NAME                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
chaos-litmus-auth-server-service   ClusterIP   10.111.108.221   <none>        9003/TCP,3030/TCP   11m
chaos-litmus-frontend-service      NodePort    10.103.249.236   <none>        9091:31495/TCP      11m
chaos-litmus-server-service        ClusterIP   10.104.133.180   <none>        9002/TCP,8000/TCP   11m
chaos-mongodb-arbiter-headless     ClusterIP   None             <none>        27017/TCP           11m
chaos-mongodb-headless             ClusterIP   None             <none>        27017/TCP           11m
```

ChaosCenter에 접속하기 위해서는 chaos-litmus-frontend-service로 접속할 수 있습니다. 클러스터로 구성된 경우 NodeIP:Port로 접속할 수 있습니다. `http://127.0.0.1:31495`

minikube의 경우 다음 명령어를 실행하여 접근할 수 있습니다.

```sh
minikube service -n litmus chaos-litmus-frontend-service
```

처음 접속할 시에는 아이디와 비밀번호는 다음과 같습니다.
```
Username: admin  
Password: litmus
```

### 발생한 문제
1. MongoDB image(arm)  

`kubectl get pods -A` 명령어를 수행한 결과 다음과 같이 mongodb가 CrashBackoff가 난 것을 확인할 수 있다. 정확한 원인을 찾기 위해 다음 명령을 수행했다.

```sh
kubectl describe pods chaos-mongodb-0 -n litmus
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  11m                  default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
  Normal   Scheduled         11m                  default-scheduler  Successfully assigned litmus/chaos-mongodb-0 to minikube
  Normal   Pulling           11m                  kubelet            Pulling image "docker.io/bitnami/bitnami-shell:10-debian-10-r431"
  Normal   Pulled            11m                  kubelet            Successfully pulled image "docker.io/bitnami/bitnami-shell:10-debian-10-r431" in 3.387s (44.771s including waiting). Image size: 79507540 bytes.
  Normal   Created           11m                  kubelet            Created container volume-permissions
  Normal   Started           11m                  kubelet            Started container volume-permissions
  Normal   Created           10m (x4 over 11m)    kubelet            Created container mongodb
  Normal   Started           10m (x4 over 11m)    kubelet            Started container mongodb
  Normal   Pulled            9m26s (x5 over 11m)  kubelet            Container image "docker.io/bitnami/mongodb:5.0.8-debian-10-r24" already present on machine
  Warning  BackOff           96s (x46 over 10m)   kubelet            Back-off restarting failed container mongodb in pod chaos-mongodb-0_litmus(1c61f1eb-5046-4960-9abb-2f79e418ff4f)
```

해당 문제가 발생한 원인은 litmus helm chart는 bitnami/mongodb를 사용합니다. 그러나 bitnami/mongodb는 arm에 대한 기술적인 지원을 하지 않기에 mongodb 이미지가 작동을 하지 않아 발생하는 문제입니다. 해당 문제의 경우 [https://github.com/bitnami/containers/issues/40947#issuecomment-1968364385](https://github.com/bitnami/containers/issues/40947#issuecomment-1968364385) 에 따라 문제를 해결해주었습니다.

2. auth server 및 server init 문제

공식 문서에 따라 mongodb 파드들이 crashBackoff 문제가 발생하던 것을 해결했으나 다른 문제가 발생했습니다. 이처럼 mongodb는 실행이 잘되었습니다. 그러나 auth server와 server가 init 상태에셔 변하지 않는 것을 확인했습니다

```sh
chaos-litmus-auth-server-8499fc4cf-xxlsd   0/1     Init:0/1   0          9m
chaos-litmus-frontend-6fd8ff8b48-gtg5m     1/1     Running    0          9m
chaos-litmus-server-5484b76bc-zhrd4        0/1     Init:0/1   0          9m
chaos-mongodb-0                            1/1     Running    0          9m
chaos-mongodb-1                            1/1     Running    0          8m39s
chaos-mongodb-2                            1/1     Running    0          8m2s
chaos-mongodb-arbiter-0                    1/1     Running    0          8m40s
```

`kubectl describe pods chaos-litmus-server-5484b76bc-gfgft --namespace litmus` 명령을 통해 로그를 확인했습니다. mongon-db가 실행되었음에도 불과하고 계속해서 실행하기를 기다리는 중이었습니다.

왜 실행이 안되는지 확인하기 위해서 살펴보던 중에 다음 스크립트를 확인할수 있었습니다. 초기화 컨테이너 부분에서 5초마다 mongodb가 연결이 되었는지 지속적으로 확인하는 스크립트를 발견했습니다. 처음에는 configMap에서 DB\_SERVER를 가져오지 못하는 줄 알았습니다.

```sh
Init Containers:
  wait-for-mongodb:
    Container ID:  docker://329981d4f9c6e818f73858cbaa90c9fac0bb7fb3c263f964816d0c31630df63a
    Image:         litmuschaos.docker.scarf.sh/litmuschaos/mongo:6
    Image ID:      docker-pullable://litmuschaos.docker.scarf.sh/litmuschaos/mongo@sha256:8f6a98f913be5227347e2ac66a707635652e8bc6ddbc682ab2b4db568acb9c0c
    Port:          <none>
    Host Port:     <none>
    Command:
      /bin/bash
      -c
    Args:
      until [[ $(mongosh -u ${DB_USER} -p ${DB_PASSWORD} ${DB_SERVER} --eval 'rs.status()' | grep 'ok' | wc -l) -eq 1 ]]; do sleep 5; echo 'Waiting for the MongoDB to be ready...'; done; echo 'Connection with MongoDB established'

Environment:
      DB_PASSWORD:  1234
      DB_USER:      root
      DB_SERVER:    <set to the key 'DB_SERVER' of config map 'chaos-litmus-admin-config'>  Optional: false

Environment Variables from:
  chaos-litmus-admin-secret  Secret     Optional: false
  chaos-litmus-admin-config  ConfigMap  Optional: false
```

`kubectl describe -n litmus cm chaos-litmus-admin-config` 으로 DB\_SERVER 정보를 확인했습니다. DB\_SERVER에 대한 정의는 잘되어있기에 어떤 것이 문제인지 알수 없었습니다.

```yaml
Data
====
DB_SERVER:
----
mongodb://chaos-mongodb-0.chaos-mongodb-headless:27017,chaos-mongodb-1.chaos-mongodb-headless:27017,chaos-mongodb-2.chaos-mongodb-headless:27017/admin
SKIP_SSL_VERIFY:
----
```

`kubectl logs chaos-litmus-auth-server-8499fc4cf-xtmzj -c wait-for-mongodb -n litmus` 으로 로그를 확인했으나 별다른 정보를 얻지 못하였습니다.

```sh
MongoServerSelectionError: Server selection timed out after 30000 ms
Waiting for the MongoDB to be ready...
```

다른 방법으로는 minikube를 다른 버전으로 실행시키는 것이었습니다. 그러나 동일한 문제가 발생되었습니다.

그러던 중 동일한 mongodb init 이슈가 발생하는 글을 발견했습니다. [https://github.com/Orange-OpenSource/towards5gs-helm/issues/4](https://github.com/Orange-OpenSource/towards5gs-helm/issues/4). 해당 issue에서는 k8s core-dns의 문제일 가능성에 대해서 이야기를 했고 [https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) 공식문서에 따라 문제 해결을 시도해보았습니다.

```sh
kubectl logs --namespace=kube-system -l k8s-app=kube-dns
kubectl -n kube-system rollout restart deployment coredns
```

`kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml` 명령어 실행 후 `kubectl exec -i -t dnsutils -- nslookup kubernetes.default` 실행했으나 정상적으로 잘 작동이 됨을 확인했습니다.

```
Server:    10.0.0.10
Address 1: 10.0.0.10

Name:      kubernetes.default
Address 1: 10.0.0.1
```

```sh
kubectl exec -i -t dnsutils -- nslookup chaos-mongodb-headless.litmus
kubectl exec -i -t dnsutils -- ping chaos-mongodb-headless.litmus.svc.cluster.local
kubectl logs --namespace=kube-system -l k8s-app=kube-dns

[INFO] 10.244.0.21:51231 - 48491 "PTR IN 20.0.244.10.in-addr.arpa. udp 42 false 512" NOERROR qr,aa,rd 143 0.000074291s
[INFO] 10.244.0.21:54463 - 20356 "PTR IN 20.0.244.10.in-addr.arpa. udp 42 false 512" NOERROR qr,aa,rd 143 0.00004675s
[INFO] 10.244.0.21:59173 - 60513 "PTR IN 20.0.244.10.in-addr.arpa. udp 42 false 512" NOERROR qr,aa,rd 143 0.000035917s
[INFO] 10.244.0.21:60011 - 63967 "PTR IN 20.0.244.10.in-addr.arpa. udp 42 false 512" NOERROR qr,aa,rd 143 0.000031166s
[INFO] 10.244.0.26:44737 - 37671 "AAAA IN chaos-mongodb-0.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 174 0.000169166s
[INFO] 10.244.0.26:44737 - 41018 "A IN chaos-mongodb-0.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 160 0.000250666s
[INFO] 10.244.0.26:41007 - 28748 "AAAA IN chaos-mongodb-1.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 174 0.000253917s
[INFO] 10.244.0.26:41007 - 37559 "A IN chaos-mongodb-1.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 160 0.000305917s
[INFO] 10.244.0.26:52689 - 47038 "AAAA IN chaos-mongodb-2.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 174 0.000234958s
[INFO] 10.244.0.26:52689 - 56252 "A IN chaos-mongodb-2.chaos-mongodb-headless.litmus.svc.cluster.local. udp 81 false 512" NOERROR qr,aa,rd 160 0.000259958s
```

별다른 정보를 얻지 못하던 중 하나의 글을 발견했습니다. [https://github.com/suyeon-jung-dev/suyeon-jung-dev/blob/main/learning-records/devops/litmus/installing\_chaos.md](https://github.com/suyeon-jung-dev/suyeon-jung-dev/blob/main/learning-records/devops/litmus/installing_chaos.md). 글을 따라 ns 삭제 및 helm repo 를 삭제하고 다시 등록한 결과 문제를 해결하게 되었습니다.

```sh
kubectl delete ns litmus  
helm repo remove litmuschaos
```

그리고나서 helm repo를 다시 추가한 결과 litmuschaos가 잘 작동되는 것을 확인했습니다. 처음 helm repo를 추가할 때 arm 이미지가 아닌 방식으로 추가해서 발생했던 문제인것 같습니다. 

```sh
kubectl get pods -n litmus
NAME                                       READY   STATUS    RESTARTS      AGE
chaos-litmus-auth-server-8499fc4cf-6sf6h   1/1     Running   0             41m
chaos-litmus-frontend-6fd8ff8b48-z8pnw     1/1     Running   0             41m
chaos-litmus-server-5484b76bc-79clm        1/1     Running   0             41m
chaos-mongodb-0                            1/1     Running   0             41m
chaos-mongodb-1                            1/1     Running   1 (36m ago)   40m
chaos-mongodb-2                            1/1     Running   0             40m
chaos-mongodb-arbiter-0                    1/1     Running   0             41m
```

Ref  
[minukube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fhomebrew)  
[litmusChoas](https://docs.litmuschaos.io/docs/getting-started/installation/)  
[litmus-helm](https://github.com/litmuschaos/litmus-helm/blob/master/charts/litmus/values.yaml#L348)  
[동작 중인 컨테이너 접속](https://kubernetes.io/ko/docs/tasks/debug/debug-application/get-shell-running-container/)  
[mongodb](https://www.mongodb.com/docs/v4.4/mongo/)  
[참조한 github](https://github.com/suyeon-jung-dev/suyeon-jung-dev/blob/main/learning-records/devops/litmus/installing_chaos.md)