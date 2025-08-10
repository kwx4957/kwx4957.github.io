---
title: LitmusChaos 오픈소스 기여하기2
categories: [opensource,litmuschaos]
tags: [k8s,go,litmuschaos,chaosengineering]
---

### 상황

문제에 대해 기술하자면 LitmusChaos에서는 execution plane 영역과 control plane 영역 사이에서 중간자 역할을 수행하는 subscriber가 존재한다. 실험을 실행하는 과정에서 아직 argo workflow나 실험 파드가 존재하지 않는 경우, 파드의 로그를 가져오려고 하면 일관된 로그 메시지인 `“mainLogs":"Failed to get argo pod logs"` 를 사용자에게 전달한다. 이러한 로그 메시지는 사용자 입장에서 직관적이지 못하다. 사용자가 구성한 설정 값이나 시스템 환경에서 에러가 발생한 것으로 착각하게 만들수 있기 때문이다. 

### 문제 정의

사용자가 파드 로그를 조회할 때, argo workflow나 실험에 대한 파드가 존재하지 않아서 발생하는 문제이다. 실험을 실행하면 execution plane에서 argo workflow가 가장 먼저 실행된다. argo가 준비되지 않은 상태에서 발생하는 문제인지라 execution plane에서 문제를 처리하는 것은 어렵다고 생각했다. 좋은 방법은 실험을 생성하면 graphql에서 mongoDB로 가장 먼저 실험에 대한 document를 삽입한다. 이때 phase라 하는 실험 상태에 대한 필드 또한 가지고 있다. 따라서 graphql에서 phase 필드를 가져와 분기별로 처리한다면, 사용자에게 직관적이며 일관된 로그를 제공할수 있을 것이라 생각했다.

### 구성도

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/e455bf42-a30f-43bb-9521-45ff77e9bee7/147f9e54-e51a-492b-9e91-59ae4478ae4c/image.png)

### 어떻게 해결했는가?

처음에는 getExperimentRun이라는 실험에 대한 정보를 가져오는 쿼리가 존재했다. 그러나 새로운 api를 통해 가져오는 것이 맞다라는 생각을 가졌고 새로운 api를 생성했다. litmuschaos는 gqlgen을 통해 스키마와 리졸버를 자동으로 생성하며, 스키마는 파일의 최상단에 위치한다. 

```go
// 쿼리문
getExperimentRunPhase(request: ExperimentRunPhaseRequest!): ExperimentRun!

// input 스키마 
input ExperimentRunPhaseRequest {
  projectID: ID!
  
  """
  ID of the infra infra in which the experiment is running
  """
  infraID: InfraIdentity!
  
  """
  ID of the experiment run which is to be queried
  """
  experimentRunID: String!	
  
  """
  notifyID is required to give an ack for non cron experiment execution
  """
  notifyID: String
}
```

 스키마를 정의하고 나서는 gqlgen을 통해 정의된 스키마를 바탕으로 코드를 생성해준다. 

```go
go run github.com/99designs/gqlgen generate
```

하지만 새롭게 생성한 쿼리문을 지우고 기존 getExperimentRun를 활용해야 겠다는 생각이 들었다. 이유는 3가지이다. 

- 동일한 책임을 가진 쿼리문이 존재하며, 새로운 쿼리를 만들더라도 관리하는 것이 어려울 것이다.
- 서버가 클라이언트에 대해서 알고 있다고 느꼈다. 인증 부분에만 차이가 가지고 있기 때문이다.
- graphql의 목적 또는 철학과 부합하지 않는다.

새로 만들었던 스키마를 삭제하고 getExperimentRun에 인프라 인자 값과 인프라 인증에 대한 코드 부분을 추가해주었다.

```go
//Befroe
getExperimentRun(projectID: ID!, experimentRunID: ID,   notifyID: ID): ExperimentRun!

//After
getExperimentRun(projectID: ID!, experimentRunID: ID,   notifyID: ID, infraID: InfraIdentity): ExperimentRun
```

또 다른 문제가 포함되어 있으니 바로 subscriber의 로그 메시지 처리 부분이다. 간단한 코드 구조를 그려보자면 가장 밑단에서 로그 메세지를 처리해야 한다. 인프라 인증에 대한 값이 필요하므로 SendPodLogs에 대한 인자 값이 CreatePodLog까지 전달되어야 하는 문제가 있었다. 

```go
SendPodLogs -> GenerateLogPayload -> CreatePodLog -> GetLogs 순으로 파드 로그를 가져온다.
   ↓
SendRequest
```

코드를 보다 개선해서 다음과 같이 구조를 바꾸었다. 아쉬운 점은 GetPodLogs 이름이 GetLogs와 어떠한 차이를 가지고 있는지 구별이 안되는점. 더 나은 함수명을 고민해보았으나 마땅히 떠오르지가 않는다. 

```go
SendPodLogs
    | --------> GetPodLogs -> GetLogs
                           -> SendExperimentRunRuquest // graphql에 대한 query 요청
                           -> CreatePodLog
    | --------> GenerateLogPayload
SendRequest
```

### 결과

간단하지만 grqaph을 사용해본 바, rest api와는 색다른 맛을 느꼈다. 즐거운 점은 누군가 내 코드를 읽고 잘 사용하기 위해서 스스로 더 많은 생각과 고민을 하게 된다는 점. 왜라는 질문이 굉장히 중요함을 느낀다. 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/e455bf42-a30f-43bb-9521-45ff77e9bee7/e804c991-c41e-4ba3-b0b0-603ac169e1e6/image.png)

[Ref]   
[litmuschaos architecture](https://docs.litmuschaos.io/docs/architecture/chaos-execution-plane)   
[gqlgen getting-started](https://gqlgen.com/getting-started/)    
[litmuschaos pr](https://github.com/litmuschaos/litmus/pull/4926)    
[litmuschaos issue](https://github.com/litmuschaos/litmus/issues/4532)