---
title: Using externalDNS with bind
categories: [k8s]
tags: [k8s,linux]
date: 2025-10-22 01:12:09 +0900
---

## BIND 구성

Generate certificates
```sh
tsig-keygen -a hmac-sha256 externaldns-key
key "externaldns-key" {
        algorithm hmac-sha256;
        secret "+FwSsuTOWuyr/h9pEUDUjAS6kk+OHIQaxbrmCmOBbx8=";
};
```

dns 설정
```sh
# 하단의 설정에 추가해준다. 
sudo vi /etc/named.conf
key "externaldns-key" {
        algorithm hmac-sha256;
        secret "+FwSsuTOWuyr/h9pEUDUjAS6kk+OHIQaxbrmCmOBbx8=";
};

zone "gateway.com" {
    type master;
    file "/etc/bind/pri/k8s/k8s.zone";
    allow-transfer {
        key "externaldns-key";
    };
    update-policy {
        grant externaldns-key zonesub ANY;
    };
};

# 저장 후 검증
named-checkconf /etc/named.conf
```

k8s zone 설정
```sh
# 존 파일 생성
mkdir -p /etc/bind/pri/k8s/
vi /etc/bind/pri/k8s/k8s.zone
$TTL 60
@         IN SOA  ns.gateway.com. admin.google.com. (
                                16         ; serial
                                60         ; refresh (1 minute)
                                60         ; retry (1 minute)
                                60         ; expire (1 minute)
                                60         ; minimum (1 minute)
                                )
                        NS      ns.gateway.com.
ns                      A       192.168.0.1

named-checkzone gateway.com /etc/bind/pri/k8s/k8s.zone

# 권한 설정 및 named 재시작
sudo chown root:named /etc/bind/pri/k8s/k8s.zone
sudo chmod 640 /etc/bind/pri/k8s/k8s.zone
sudo chcon -R -t named_zone_t /etc/bind/pri/k8s
systemctl restart named
```

## k8s external-dns 설정
해당 방식은 gateway-api와 bind를 활용한 방식이다. rfc2136-host와 앞서 생성한 tsig-secret 값을 맞춰준다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: external-dns
  labels:
    name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
  namespace: external-dns
rules:
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - watch
  - list
- apiGroups: 
  - gateway.networking.k8s.io
  resources:
  - gateways
  - httproutes
  - grpcroutes
  - tlsroutes
  verbs: 
  - get
  - list
  - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
  namespace: external-dns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.19.0
        args:
        - --registry=txt
        - --txt-prefix=external-dns-
        - --txt-owner-id=k8s
        - --provider=rfc2136
        - --rfc2136-host=192.168.0.1
        - --rfc2136-port=53
        - --rfc2136-zone=gateway.com
        - --rfc2136-tsig-secret=+FwSsuTOWuyr/h9pEUDUjAS6kk+OHIQaxbrmCmOBbx8=
        - --rfc2136-tsig-secret-alg=hmac-sha256
        - --rfc2136-tsig-keyname=externaldns-key
        - --rfc2136-tsig-axfr
        - --source=gateway-httproute
        - --source=gateway-grpcroute
        - --source=gateway-tlsroute
        - --domain-filter=gateway.com
```

HTTPRoute 예시
```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: external-dns-httproute
  namespace: default
  annotations:
    external-dns.alpha.kubernetes.io/hostname: test.gateway.com
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  parentRefs: 
  - name: default-gateway
    namespace: default
  hostnames:
  - test.gateway.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: jenkins
      namespace: jenkins
      port: 8080
```

## 트러블슈팅
1. external-dns crashbackoff

gatway API 클라이언트 연동 실패로 인한 문제가 발생한다. 비단 나뿐만이 아니라 다른 사람들 또한 해당 문제를 겪고 있따. clusterrolebing에 대해 gateway 리소스에 대한 권한을 부여해주었다. 공식 문서와는 다르게 helm으로 템플릿 비교하며 해결 완료. 

```sh
time="2025-10-22T01:23:35Z" level=fatal msg="failed to sync *v1beta1.Gateway: context deadline exceeded with timeout 1m0s"
   2025-10-22 10:22:35.528   
time="2025-10-22T01:22:35Z" level=info msg="Created Kubernetes client https://10.233.0.1:443"
   2025-10-22 10:22:35.528   
time="2025-10-22T01:22:35Z" level=info msg="Using inCluster-config based on serviceaccount-token"
   2025-10-22 10:22:35.528   
time="2025-10-22T01:22:35Z" level=info msg="Instantiating new Kubernetes client"
   2025-10-22 10:22:35.528   
time="2025-10-22T01:22:35Z" level=info msg="Created GatewayAPI client https://10.233.0.1:443"
```
[troubleshooting]
https://github.com/kubernetes-sigs/external-dns/issues/4768

[공식]  
https://kubernetes-sigs.github.io/external-dns/latest/docs/sources/gateway-api/  
https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/rfc2136/#using-with-bind

[dns]
https://docs.rockylinux.org/10/guides/dns/private_dns_server_using_bind/  
https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/8/html/deploying_different_types_of_servers/assembly_configuring-zones-on-a-bind-dns-server_assembly_setting-up-and-configuring-a-bind-dns-server

[blog]  
https://weng-albert.medium.com/build-an-externaldns-with-external-bind-dns-en-fd8375c49919
