---
title: docker가 ufw를 무시해요!
comments: true
date: 2020-09-23 18:23:26
tags:
    - docker
    - ufw
    - 보안
category: 일상
---

개인 서버에 매번 scp를 통해서 파일을 올리기가 너무 귀찮아서 [filestash](https://www.filestash.app/)를 설정한 참이었다.
기본 설정으로 docker가 8334 포트를 열어두는데, nginx 리버스 프록시를 사용하기로 했기 때문에 ufw를 통해 포트를 닫았다.

초기 설정을 위해 8334 포트가 들어간 URL을 통해 접근했었는데, 이를 브라우저에서 기억하는 바람에 오늘 우연히 서버에 8334 포트로 접근하게 되었다.
**그런데 ufw를 통해 접근이 통제되어야 할 filestash가 멀쩡히 접근이 되는 게 아닌가!**

급한 마음에 서버에 접속해 ufw 설정을 확인해보니 멀쩡히 8334 포트를 막고 있는 것이 보였다.
다른 포트로 ufw 문제인지 확인해 보았으나 ufw 동작에는 이상이 없었다.
나는 docker 문제라고 확신하고 검색을 하던 중 [docker 저장소의 열려 있는 이슈를 찾아냈다.](https://github.com/docker/for-linux/issues/690)
이슈에서 언급한 [techrepublic 기사](https://www.techrepublic.com/article/how-to-fix-the-docker-and-ufw-security-flaw/)에 따르면, docker가 iptables를 직접 수정하기 때문에 일종의 프론트엔드인 ufw의 영향에서 벗어난다는 것이 주된 이유로 추정된다.

이슈 스레드에서 문제를 해결해 준다는 [ufw-docker](https://github.com/chaifeng/ufw-docker)라는 친구를 발견했다.
`/etc/ufw/after.rules`에 아래 내용을 추가하고 서버를 재시작하면 해결된다.
```bash
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

트래픽을 제대로 막았다고 생각했는데 줄줄 새고 있었다.
만능으로 생각되던 docker와 만능으로 생각되던 ufw의 조합이 이런 상상도 못한 함정을 만들어낼 줄은 몰랐다.
정말 "하나를 잘 하는" 이상적인 유닉스 철학을 이상한 곳에서 보게 되어서 황당했던 한편 실제 동작 확인을 하지 않은 안일한 내 모습을 반성했다.

여담으로, docker가 iptables를 직접 사용하는 동작은 ufw라는 제3자 소프트웨어에 의존하지 않는다는 점에서 가치가 있다고 생각한다.
다만 쉽고 편한 배포를 모토로 삼는 docker 사용자가 쉽고 편한 방화벽인 ufw를 사용하지 않을 리가 만무하기 때문에 이 문제는 해결되어야 할(혹은 공지되어야 할) 필요가 있는데, 위 이슈가 여전히 열린 상태라는 점은 아리송하다.
지금도 어딘가에서 비밀스런 docker 컨테이너가 ufw 허점 위에서 돌아가고 있는 게 아닐까.
