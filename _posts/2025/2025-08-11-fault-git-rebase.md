---
title: Git rebase를 잘못 수행한 경우 
categories: [git]
tags: [git]
date: 2025-08-11 00:26 +0900
---

원격 저장소의 커밋 내역을 수정하기 위해 git rebase를 수행했다. 문제는 rebase 이후 원격저장소에 커밋을 반영한 결과, rebase 이전 다른 사람들의 커밋에도 영향을 끼친것이다.

![](/assets/img/git/rebase1.png)


내가 작업한 커밋이 아니지만 같이 수행한 것으로 표기된다. 예상되는 원인으로는 다른 브랜치에서 작업한 후 master 브랜치에 이를 병합하고나서 rebaes를 수행한 것이 원인으로 보인다. 병합한 것은 최근이지만 커밋들은 직전의 커밋이 아닌 중간중간에 속해있기 때문이다.


로컬 저장소 git 로그 확인
1. git reflog

![](/assets/img/git/rebase2.png)

특정 시점으로 되돌아가기
2. git reset --hard HEAD@{61}

다른 브랜치에서 작업한 후 다시 병합을 해주었다.

https://velog.io/@whoyoung90/TIL-51-git-rebase를-잘못한-경우-되돌아가기