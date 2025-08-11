---
layout: post
title: sudo-redirection-does-not-work
date: 2025-08-11 21:11 +0900
category: [linux,cmd]
tags: [linux]
---

sudo 명령어를 통해 redirection을 수행하고자 했으나 Permission Denied로 원하는 동작을 수행하지 못하였다. 예를 들어 다음 명령어를 수행하면 Permission denied이 뜨게된다.

```sh
sudo echo “hello, wold” > /etc/hosts
```

이러한 문제가 발생하는 원인은 해당 명령이 두 부분으로 나눠지기 때문이다. sudo echo "hello, world" 부분은 root 사용자로 명령을 수행한다. 하지만 redirection에는 sudo 명령어가 영향을 끼치지 않고 쉘에 의해 처리가 된다. 따라서 일반 사용자로 명령을 수행하고 Permission denied이 뜨게 된 것이다.

이러한 문제를 해결하기 위해서는 해당 명령어와 같이 sh를 수행하면 된다.
```sh
sudo sh -c "echo 'hello, worlld’ >> /etc/hosts"

echo “hello” | sudo tee -a /etc/hosts >> /dev/null
```


[https://stackoverflow.com/questions/84882/sudo-echo-something-etc-privilegedfile-doesnt-work](https://stackoverflow.com/questions/84882/sudo-echo-something-etc-privilegedfile-doesnt-work) 
[https://www.lesstif.com/lpt/linux-tee-89556049.html](https://www.lesstif.com/lpt/linux-tee-89556049.html)