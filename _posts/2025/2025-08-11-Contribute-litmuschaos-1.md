---
title: LitmusChaos 오픈소스 기여하기1
categories: [opensource,litmuschaos]
tags: [k8s,litmuschaos,chaosengineering,docs]
date: 2025-08-11 00:36 +0900
---

> 오픈소스 컨트리뷰션 2024 에 참가하여 litmusChaos 프로젝트를 다루는 도중에 발생한 일을 기록한 글입니다.

litmusChaos를 다루던 중에 web 부분에서 간단한 오류를 발견하게 되었다. 해당 부분을 수정하기 위해서 코드를 다운받고 [공식 가이드](https://github.com/litmuschaos/litmus/wiki/ChaosCenter-Development-Guide\)에 따라서 코드를 실행시켰다.

공식 가이드를 처음부터 발견한 것은 아니었다. web 부분을 실행하기 위해서 이것저것 하다가 백엔드가 필요한 것을 깨달았고, 뒤끝에 문서를 뒤지다가 `contributing.md` 에서 가이드라인 문서를 발견하게 되었다.

공식문서에 따라 설정을 하던 중에 오류에 맞닥뜨리는데 web, auth, server 그리고 mongodb를 실행시키는 것에는 문제가 없었다. 그러나 chaos infrastructure를 설정하기 위해서 litmusctl를 설치하고 나서 `litmusctl config set-account` 실행한 후 https://localhost:8185를 입력하면 다음 에러가 발생한다.

> Post https://localhost:8185/auth/login: tls: failed to verify certificate: x509: certificate signed by unknown authority

대충 자체 서명한 인증이라 인증을 못 해주겠다는 뜻. 그래서 자체 서명을 받기 위해서 상단의 명령을 실행시키니까 크롬에서 localhost가 변조되어서 위험하다고 접근조차 못 하게 한다. 외 않되??

자체 서명을 인증받기 위해서 다음 과정을 수행했다.

```sh
openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout certificates/localhost-key.pem -out certificates/localhost.pem -config config/pem.cfg -extensions 'v3_req'

Generating a RSA private key
.+++++
writing new private key to 'certificates/localhost-key.pem'
-----
unable to get 'req_distinguished_name' section
problems making Certificate Request
```

에러 발생해서 하단의 과정으로 수행
```
openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout certificates/localhost-key.pem -out certificates/localhost.pem -config ./pem.cfg -extensions 'v3_req'
```

Lets'encrpt 방식
```sh
openssl req -x509 -out localhost.crt -keyout localhost.key \
-newkey rsa:2048 -nodes -sha256 \
-subj '//CN=localhost' -extensions EXT -config <( \
printf "[dn]\nCN=localhost\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:localhost\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

하지만 윈도우에서는 기본 리눅스 명령어를 실행하면 에러가 발생한다. 이 경우에는 /부분을 하나 더 추가하여 escape 시켜줌으로써 명령이 잘 작동한다.

```sh
name is expected to be in the format /type0=value0/type1=value1/type2=... where characters may be
escaped by \

기존의 명령
openssl req -x509 -out certificates/localhost.pem -keyout certificates/localhost-key.pem \
  -newkey rsa:2048 -nodes -sha256 \
  -subj "/CN=localhost" -extensions EXT -config pem.cfg

수정 명령 
openssl req -x509 -out certificates/localhost.pem -keyout certificates/localhost-key.pem \
  -newkey rsa:2048 -nodes -sha256 \
  -subj "//CN=localhost" -extensions EXT -config pem.cfg
```

Mac
```sh
openssl req -x509 -nodes -days 730 -newkey rsa:2048 -keyout certificates/localhost.pem -out certificates/localhost.crt -config config/pem.cfg -extensions 'req'

❯ sudo security add-trusted-cert -d -r trustRoot -k "/Library/Keychains/System.keychain" localhost.crt
```

Windows에서 CA를 신뢰할수 있도록 설정한다.
```sh
certutil -addstore "Root" "localhost.crt"

Root "신뢰할 수 있는 루트 인증 기관"
서명이 공개 키와 일치합니다.
"XXX" 인증서가 저장소에 추가되었습니다.
CertUtil: -addstore 명령이 성공적으로 완료되었습니다.
```

```sh
litmusctl config set-account

Post "https://localhost:8185/auth/login": tls: failed to verify certificate: x509: certificate signed by unknown authority
-----
litmusctl config set-account -e https://localhost:8185 -u admin -p litmus --skipSSL     

Unmatched status code:Error occurred while trying to proxy: localhost:8185/login
-----
 litmusctl config set-account -e https://localhost:8185 -u admin -p litmus --cacert ./certificates/localhost-key.pem -n

Post "https://localhost:8185/auth/login": tls: failed to verify certificate: x509: certificate relies on legacy Common Name field, use SANs instead
```

또 삽질을 진행하다가 혹시나 싶어서 시크릿창으로 들어가니까 잘 된다,,, 돼. 히히히히히. 몰랐는데 지금 다시 해당 실행했던 명령어를 보니 proxy 부분이 발생할 때는 자체 서명한 인증이 신뢰할 수 있는 기관에 잘 등록된 상태임을 알 수가 있다.

![](/assets/img/litmuschaos/init-litmuschaos.png)

이제 인증도 받았겠다. 되겠지?? 다시 `litmusctl config set-account` 실행하니 또 에러가 발생한다.

> Unmatched status code:Error occurred while trying to proxy: localhost:8185/login

프론트에서 발생한 로그를 확인하니 다음 에러가 출력됐다.

```sh
[webpack-dev-server] [HPM] Error occurred while proxying request localhost:8185/capabilities to http://localhost:3000/ [ECONNREFUSED] (https://nodejs.org/api/errors.html#errors_common_system_errors)
```

모든 서버 프로세스가 실행했는데 왜 안될까라는 의구심이 피어났다. 하단의 명령어를 실행하니 Auth 서버를 실행 중이지만 3000번 포트는 존재하지 않는다. 그렇다면 Auth 서버는 어디로 갔는가?

```sh
netstat -ano | find "3000"
```

Auth 서버의 실행하는 로그를 보던 중에 Server 의 경우 실행되는 포트가 출력되는 것을 발견. 인증 서버도 동일하게 해당 부분을 출력한다. 그러나 포트에 대한 부분이 없음을 찾게 되었다. uitls 폴더의 config.go에 해당 변수를 주입받는 것을 찾게 되었다. 코드를 살펴보던 중에 다음 코드를 발견했다.

```go
//config.go 
RestPort = os.Getenv("REST_PORT") 
GrpcPort = os.Getenv("GRPC_PORT")
```

이러한 문제가 발생한 원인을 공식문서에서는 환경변수를 주입하지 않았다. 따라서 Auth 서버로 프록시하는 과정에서 당연히 에러가 발생하게 된 것이다.

```go
//main.go의 runRestServer 
log.Infof("Listening and serving HTTP on :%s", utils.RestPort)
```

이제는 진짜진짜로 실행될 거라는 마음을 품고 다시 litmusctl을 실행하니 또 다른 문제가 발생하는데

```sh
litmusctl config set-account

Host endpoint where litmus is installed: https://localhost:8185
Username [Default: admin]: admin
Password: *********

account.username/admin configured
🚫 ChaosCenter version: ci is not compatible with the installed LitmusCTL version: 1.8.0
Compatible ChaosCenter versions are: 
[ '3.0.0' '3.1.0' '3.2.0' '3.3.0' '3.4.0' '3.5.0' '3.6.0' '3.6.1' '3.7.0' '3.8.0' '3.9.0' '3.9.1' ]
```

![](/assets/img/litmuschaos/init-litmuschaos2.png)

Pending 상태에 걸려 아무것도 수행할 수가 없었다. 카오스센터가 설정되지 않아서 발생하지 않는 문제인가 싶어서 공식 문서를 재진행했으나 해당 부분이 오류일 것이 아니라는 직감이 들었다. 오류를 보면 알 수 있듯이 컨트롤플레인의 api-server에 접근하고 있기 때문이다.

```sh
> litmusctl connect chaos-infra

Project list:
1.  admin-project

Select a project [Range: 1-1]: 1

Installation Modes:
1. Cluster
2. Namespace

Select Mode [Default: cluster] [Range: 1-2]: 

🏃 Running prerequisites check....Post "https://127.0.0.1:53525/apis/authorization.k8s.io/v1/selfsubjectaccessreviews": dial tcp 127.0.0.1:53525: connect: connection refused
Post "https://127.0.0.1:53525/apis/authorization.k8s.io/v1/selfsubjectaccessreviews": dial tcp 127.0.0.1:53525: connect: connection refused

🚫 You don't have sufficient permissions.
🙄 Please use a service account with sufficient permissions.
```

즉 현재 진행 중인 작업은 web 개발을 위한 작업이 아닌듯했다.

[Chaos center](https://docs.litmuschaos.io/docs/architecture/chaos-control-plane)가 동작하는 방식은 다음과 같다. 다음 로직에서 인증 서버와 DB의 문제라면 로그인 과정에서 에러가 발생할 것이라는 생각이 들었다. dex라는 환경변수를 주입받는데 해당하는 부분에 대한 문제인가 싶어서 알아보던 결과 dex oauth의 역할을 담당. 따라서 dex의 환경변수는 아니라는 생각이 들었다.

graphql 서버 또는 db의 문제라는 생각이 들었다. 문제를 찾기 위해 graphql의 server.go를 뒤지던 중 main 함수에서 utils.Config.RestPort라는 부분을 발견하게 되었고 의식의 흐름대로 util 패키지 전부를 살피던 중 variable.go라는 파일을 발견 다음과 같이 인증서버의 grpc와 통신하고 있는 것을 알게 되었다. 그 결과 인증서버를 실행하기 전에 환경변수 3030을 주입한 결과 로컬 환경에서도 잘 작동한다.

```go
LitmusAuthGrpcEndpoint      string   `split_words:"true" default:"localhost"`
LitmusAuthGrpcPort          string   `split_words:"true" default:"3030"`
KubeConfigFilePath          string   `split_words:"true"`
RemoteHubMaxSize            string   `split_words:"true"`
SkipSslVerify               string   `split_words:"true"`
RestPort                    string   `split_words:"true" default:"8080"`
GrpcPort                    string   `split_words:"true" default:"8000"`
```

이후 Auth 서버를 실행하기 전에 다음 환경변수를 주입하고 실행하니 Chaos Center가 잘 작동한다.

```go
export REST_PORT=3000 
export GRPC_PORT=3030
```

해당 문제를 해결하여 로컬 환경에서 개발할 수 있는 환경을 구성하니 멘토님이 말씀 하시길 공식문서에 개발 가이드라인을 작성하면 좋을 것이라고, 정리하자면 환경변수가 주입되지 않아서 삽질에 삽질한 경험이라고 볼 수 있다.


[https://gitlab.com/gitlab-org/gitlab-runner/-/issues/28841](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/28841)  
[https://gist.github.com/cecilemuller/9492b848eb8fe46d462abeb26656c4f8](https://gist.github.com/cecilemuller/9492b848eb8fe46d462abeb26656c4f8)  
[https://letsencrypt.org/ko/docs/certificates-for-localhost/](https://letsencrypt.org/ko/docs/certificates-for-localhost/)  
[https://kdev.ing/windows-certlm/](https://kdev.ing/windows-certlm/)
