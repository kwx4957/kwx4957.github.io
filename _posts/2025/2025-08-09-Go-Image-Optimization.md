---
title: Go Docker Image Optimization
categories: [Docker,go]
tags: [docker,go]
---

런타임 이미지로 distroless 또는 alpine을 사용하지 않는 이유는 golang은 컴파일 시에 의존성이 전부 바이너리에 포함된채로 컴파일되기 때문에, 별도의 구성 요소없이 scratch를 사용하는 것이 더 효율적이기 때문이다.

부가적인 설명을 하자면 `CGO_ENABLED=0`은 C 라이브러리의 연결을 완전히 비활성화하여 순수하게 GO 코드만으로 바이너리를 생성한다. `GooS`는 타켓 운영체제 지정. `GOARCH`는 서버의 아키텍처를 지정한다. 만일 설정이 없는 경우 실행되는 머신의 구성을 따라간다. `-a`는 의존되는 모든 패키지를 재빌드함으로써 일관성을 보장한다. `-ldflags '-s -w'`는 심볼 테이블 및 디버그 정보를 제거함으로써 바이너리 크기가 대폭 감소한다.

UPX란 실행 파일 압축 도구로. 바이너리를 압축하고, 실행 시 메모리에서 자동 압축해제 한다. `--best` 최고 압축률 사용(레벨 9), `--lzma` lzma` 압축 알고리즘 사용(높은 압축률, 느린 압축 속도)

### 결과

```bash
$ docker images | sort -r -k 4
REPOSITORY   TAG                    IMAGE ID       CREATED          SIZE
golang       default                e08d11698d09   33 minutes ago   1.41GB
golang       multi-stage            bddc52b98eba   2 hours ago      50.8MB
golang       multi-stage-rm-debug   84c641d8710f   13 minutes ago   45.2MB
golang       multi-stage-scratch    6dcccaf5e421   11 minutes ago   9.16MB
golang       multi-stage-upx        423c0f630c6a   10 minutes ago   4.06MB
```

### 기본 Dockerfile

```docker
FROM golang:1.24.2
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o swarm
ENTRYPOINT ["./swarm"]
CMD [ "run" ]
```

### 멀티 스테이지

```docker
FROM golang:1.24.2-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o swarm

FROM gcr.io/distroless/base-debian11
WORKDIR /
COPY --from=builder /app/swarm /swarm
ENTRYPOINT ["/swarm"]
CMD [ "run" ]
```

### 멀티 스테이지-rm-debug

```docker
FROM golang:1.24.2-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags='-w -s' -o swarm

FROM gcr.io/distroless/base-debian11
WORKDIR /
COPY --from=builder /app/swarm /swarm
ENTRYPOINT ["/swarm"]
CMD [ "run" ]
```

### 멀티 스테이지-scratch

```docker
FROM golang:1.24.2-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags='-w -s' -o swarm

FROM scratch
WORKDIR /
COPY --from=builder /app/swarm /swarm
ENTRYPOINT ["/swarm"]
CMD [ "run" ]
```

### UPX를 통한 최적화

```docker
FROM golang:1.24.2-alpine AS builder
RUN apk add --no-cache upx
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags='-w -s' -o swarm
RUN upx --best --lzma swarm

FROM scratch
COPY --from=builder /app/swarm /swarm
ENTRYPOINT ["/swarm"]
CMD [ "run" ]
```

[Ref]    
https://velog.io/@ilcm96/docker-multi-stage-build-upx    
[docker-guides](https://docs.docker.com/guides/golang/build-images/#multi-stage-builds)