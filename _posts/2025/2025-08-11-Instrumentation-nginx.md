---
title: Nginx에 Otel 자동 계측 적용하기
categories: [o11y,otel]
tags: [k8s,otel,o11y,nginx]
date: 2025-08-11 00:13 +0900
---

### Otel 설치
1. otel operator 배포
2. otel Collector(선택 사항으로 제외)
3. Instrumentation 생성
4. Application에 주석을 통한 자동 계측


otel operator helm 배포
```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install my-opentelemetry-operator open-telemetry/opentelemetry-operator \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=false \
  --set admissionWebhooks.autoGenerateCert.enabled=true \
  --set manager.extraArgs[0]="--enable-nginx-instrumentation=true"
```

Instrumentation 생성
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: nginx-instrumentation
spec:
  exporter:
    endpoint: http://tempo-distributor.default.svc.cluster.local:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "0.25"
  nginx:
    attrs:
    - name: NginxModuleOtelMaxQueueSize
      value: "4096"
```

Nginx 배포
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        instrumentation.opentelemetry.io/inject-nginx: "true" 
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
        ports:
        - containerPort: 80
```

만일 파드에서 모듈 가져오지 못한다는 로그가 발생한다면 nginx 버전의 문제일 가능성이 크다. nginx 버전을 최대한 낮춰보자.

[Ref]   
[Otel 공식문서](https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md)   
[Otel 예제](https://github.com/open-telemetry/opentelemetry-operator/tree/main/tests/e2e-instrumentation/instrumentation-nginx-multicontainer)   

[블로그]  
https://joeunvit.tistory.com/15  
https://wlsdn3004.tistory.com/49#c4  
https://luvstudy.tistory.com/237#minio-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0  
https://velog.io/@utcloud/OpenTelemetry-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EA%B5%AC%EC%84%B1#opentelemetry-operator-%EC%84%A4%EC%B9%98  