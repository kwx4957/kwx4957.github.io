---
title: Github Actions 카카오워크 알림 전송
categories: [cicd,githubactions]
tags: [cicd,githubactions,kakaowork]
date: 2025-08-11 01:01 +0900
---

카카오워크에서 봇에 대한 생성 과정은 다음으로 이루어져있습니다. 제가 원하는 것은 github에서 수행한 배포 과정에 대한 결과를 카카오워크에 채팅을 보내는 것이기에 알림형을 선택하게 되었습니다.

![](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdna%2F6smAi%2FbtsId3fuhU4%2FAAAAAAAAAAAAAAAAAAAAAIDPbEs40cshenb61XjXwQX5uLD8jU-rIZhOF7EA3h46%2Fimg.png%3Fcredential%3DyqXZFxpELC7KVnFOS48ylbz2pIh7yKj8%26expires%3D1756652399%26allow_ip%3D%26allow_referer%3D%26signature%3D%252FkRFAAWB03lJDcsiRro%252BSzn%252F0yI%253D)

관리자 사이트에서 개발자에 대한 권한을 부여합니다. 그리고나서 개발자는 봇을 생성하면 App Key가 발급이 됩니다.  
잘 작동하는지 테스트하기 위해 다음 요청을 보냈고 이에 대한 응답으로 user_id가 필요하다는 에러가 반환되었습니다.

{% raw %}

```sh
curl -X POST https://api.kakaowork.com/v1/conversations.open \
  -H "Content-type: application/json" \
  -H "Authorization: Bearer {APP_KEY}" 

100 98 100 98 0 0 2850 0 --:--:-- --:--:-- --:--:-- 2969{"error":{"code":"missing\_parameter","message":"user\_id or user\_ids must needed"},"success":false}
```

### 유저 조회

우선 초대하고자 하는 user_id에 대한 정보를 얻어야 합니다. users.list api를 호출하여 특정 워크스페이스에 속한 전체 멤버 목록과 상세 정보를 얻을수 있습니다.

전체 유저 목록 조회
```sh
curl -X GET https://api.kakaowork.com/v1/users.list \
     -H "Authorization: Bearer {APP_KEY}"
```

하지만 이러한 방식은 특정 유저 id를 얻기 위해서는 별도의 필터 처리를 필요로 했습니다. 만일 이메일을 알고있다면 아래 API를 호출하여 user_id를 얻을수 있습니다.  

이메일을 통한 특정 user_id 조회
```sh
curl -X GET https://api.kakaowork.com/v1/users.find_by_email?email={email} \
     -H "Authorization: Bearer {APP_KEY}"

100 348 100 348 0 0 10832 0 --:--:-- --:--:-- --:--:-- 11600{"success":true,"user":{"avatar\_url":null,"department":null,"display\_name":"홍길동","emails":\[\],"id":"11111","mobiles":\[\],"name":"홍길동","nickname":null,"position":null,"responsibility":null,"space\_id":"111111","status":"activated","tels":\[\],"vacation\_end\_time":null,"vacation\_start\_time":null,"work\_end\_time":null,"work\_start\_time":null}}
```

저에 대한 user_id를 얻은 이후에는 그룹 채팅방에 초대하고자 하는 인원 user\_id를 얻은 후에 봇 그룹 채팅방 생성하는 API를 호출하면 됩니다.

### 그룹 채팅방
봇 그룹 채팅방 생성 생성
```sh
curl -X POST https://api.kakaowork.com/v1/conversations.open \
     -H "Authorization: Bearer {app_key}" \
     -H "Content-Type: application/json" \
     -d '{ "user_ids": [111, 112, 113], "conversation_name": "github actions" }'
```

그룹이 생성한 이후에는 group_id에 대한 값이 출력이 됩니다. group_id는 그룹 메시지 전송에 필수로 필요합니다. 공식 문서에서 제공하는 api를 실행해본 결과 그룹 채팅방에 메시지가 가는 것을 확인할 수 있습니다.

그룹 메시지 전송
```sh
   curl -v -g -X POST https://api.kakaowork.com/v1/messages.send \
   -H "Authorization: Bearer {APP_KEY}" \
   -H "Content-Type: application/json" \
   -d '{"conversation_id":"{group_id}", "text": "github actions 수행",
      "blocks": [
                    {"type": "header", "text": "Github Actions 배포 결과", "style": "white"},
                    {"type": "text", "text": "브랜치",
                      "inlines":

                        [{"type": "styled", "text": "브랜치 : ", "bold": true },

                        {"type": "styled", "text": "${{ github.ref_name }}" }]},

                    {"type": "text", "text": "작업자",

                      "inlines":

                        [{"type": "styled", "text": "작업수행자 : ", "bold": true},

                        {"type": "styled", "text": "${{ github.actor }}" }]},

                    {"type": "text", "text": "결과",

                      "inlines":

                        [{"type": "styled", "text": "수행결과 : ","bold": true},

                        {"type": "styled", "text": "깃 수행 결과 "}]},

                    {"type": "text", "text": "github link",

                      "inlines":

                        [{"type": "styled", "text": "Githublink "},

                        {"type": "link", "text": "Repository", "url": "${{ github.repositoryUrl }}" }]},

                    {"type": "description", "term": "날짜",

                      "content": {"type": "text","text": "2020년 9월 16일 7시"},

                      "accent": true}

      ]}'
```

윈도우에서 curl를 수행하면 다음 에러가 발생할 수 있습니다. 동일 요청에 대해서 POSTNAM 또는 리눅스에서 요청을 수행했을 때 정상적으로 요청이 이뤄지게 됩니다. 이는 윈도우에서 쌍따옴표에 대한 처리가 되지 않기 때문입니다.

> { "error": { "code": "bad\_request", "message": "bad\_request from ingress" } }

만일 윈도우에서 해당 curl를 사용하기 위해서는 다음으로 변경해줘야 한다.
```sh
curl -v -g -X POST https://api.kakaowork.com/v1/messages.send \
           -H "Authorization: Bearer  {APP_KEY}" \
           -H "Content-Type: application/json" \
           -d "{'conversation_id':'group_id','text':'카카오워크에 오신걸 환영합니다.'}"
```

이후 기존 github action에서 해당 코드를 추가해주었습니다. 해당 작업에서 시크릿을 가져올 때 "${{ secrets.APP\_KEY}}" 형태로 값을 불러와야지 curl 요청을 보낼때 bad request for ingress 요청이 발생하지 않습니다.

```yaml
steps:
    - name: get currrent date
      id: date
      run: echo "::set-output name=date::$(TZ='Asia/Seoul' date '+%Y-%m-%d %H:%M')"

    - name: send notifictaion to kakao work
      run: |
        curl -v -g -X POST https://api.kakaowork.com/v1/messages.send \
             -H "Authorization: Bearer ${{ secrets.APP_KEY }}" \
             -H "Content-Type: application/json" \
             -d '{"conversation_id": ${{ secrets.CONVENTION_ID }},"text": "github actions 수행",
                  "blocks": [
                    {"type": "header", "text": "Github Actions 배포 결과", "style": "white"},
                    {"type": "text", "text": "브랜치",
                      "inlines":
                        [{"type": "styled", "text": "브랜치 : ", "bold": true },
                        {"type": "styled", "text": "${{ github.ref_name }}" }]},
                    {"type": "text", "text": "작업자",
                      "inlines":
                        [{"type": "styled", "text": "작업수행자 : ", "bold": true},
                        {"type": "styled", "text": "${{ github.actor }}" }]},
                    {"type": "text", "text": "결과",
                      "inlines":
                        [{"type": "styled", "text": "수행결과 : ","bold": true},
                        {"type": "styled", "text": "${{ needs.build.result }}"}]},
                    {"type": "text", "text": "github link",
                      "inlines":
                        [{"type": "styled", "text": "Githublink "},
                        {"type": "link", "text": "Repository", "url": " ${{ github.server_url }}/${{ github.repository }}" }]},
                    {"type": "description", "term": "날짜",
                      "content": {"type": "text","text": "${{ steps.date.outputs.date }}"},
                      "accent": true}
                  ]}' > /dev/null
```
{% endraw %}

Ref  
[https://docs.kakaoi.ai/kakao\_work/botdevguide/#%EC%95%8C%EB%A6%BC%ED%98%95](https://docs.kakaoi.ai/kakao_work/botdevguide/#%EC%95%8C%EB%A6%BC%ED%98%95)  
[https://docs.kakaoi.ai/kakao\_work/webapireference/commonguide/](https://docs.kakaoi.ai/kakao_work/webapireference/commonguide/)  
[https://blog.naver.com/techshare/221178764058](https://blog.naver.com/techshare/221178764058)  
[https://docs.kakaoi.ai/kakao\_work/botdevguide/#%EC%95%8C%EB%A6%BC%ED%98%95](https://docs.kakaoi.ai/kakao_work/botdevguide/#%EC%95%8C%EB%A6%BC%ED%98%95)