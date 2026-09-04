---
title: 'GCP VM에 Docker로 Nginx 띄우기'
date: 2026-09-04
tags:
  - GCP
  - Docker
  - 트러블슈팅
excerpt: "GCP Compute Engine 인스턴스에 Docker로 Nginx를 올리고 외부에 공개하기까지 겪은 권한 오류와 방화벽 문제를 정리했다."
---

GCP Compute Engine 인스턴스(`us-central1-a`, Ubuntu 22.04.5 LTS)에 브라우저 SSH로 접속해 Docker로 Nginx를 올리고, 외부에서 접속 가능하게 만드는 것까지 진행했다. 단순해 보이는 작업이었지만 실제로는 두 번 막혔다 — 한 번은 **docker 소켓 권한**에서, 한 번은 **방화벽 인그레스 규칙**에서. 아래는 그 과정을 그대로 정리한 기록이다.

## 1. 브라우저 SSH로 VM 접속

GCP Console의 VM 인스턴스 목록에서 **SSH** 버튼을 눌러 별도 키 설정 없이 바로 터미널 창을 띄웠다. 콘솔이 임시 SSH 키를 발급하고 브라우저 안에서 세션을 열어주는 방식이라 로컬에 아무것도 설치할 필요가 없었다.

![GCP Console 브라우저 SSH 접속 직후 화면](/images/uploads/2026-09-04-gcp-docker/01-ssh.webp)
*GCP Console 브라우저 SSH 창. `Welcome to Ubuntu 22.04.5 LTS` 배너와 함께 최초 접속.*

접속 직후 배너에는 시스템 로드, 메모리 사용률, 로그인된 사용자 수 같은 기본 상태가 함께 출력된다.

## 2. Docker 설치 및 서비스 상태 확인

VM에는 이미 Docker가 설치되어 있었다. 버전과 데몬 상태부터 확인했다.

```bash
$ docker --version
Docker version 29.1.3, build 29.1.3-0ubuntu3~22.04.2

$ sudo systemctl status docker
● docker.service - Docker Application Container Engine
   Active: active (running) since Fri 2026-09-04 05:24:19 UTC
   Main PID: 3326 (dockerd)
   ...
   Docker daemon has completed initialization
   API listen on /run/docker.sock
```

![docker --version과 systemctl status docker 출력](/images/uploads/2026-09-04-gcp-docker/02-docker-status.webp)
*`docker --version`과 `sudo systemctl status docker` 출력.*

`active (running)` 로 떠 있는 걸 확인했으니 바로 이미지를 받으러 갔다 — 여기서 첫 번째 벽에 부딪혔다.

## 3. Docker Hub 로그인

퍼블릭 이미지만 받을 거라면 필수는 아니지만, 이후 pull rate limit을 피하려고 `docker login` 을 먼저 해뒀다. 로그인은 데몬 소켓이 아니라 레지스트리 인증만 하기 때문에, 뒤에 나오는 권한 문제와는 무관하게 이 시점엔 문제없이 끝났다.

```bash
$ docker login
USING WEB-BASED LOGIN

Your one-time device confirmation code is: XXQW-RRCV
Press ENTER to open your browser or submit your device code here:
https://login.docker.com/activate

Waiting for authentication in the browser…

Login Succeeded
```

![docker login 웹 인증 흐름과 Login Succeeded 메시지](/images/uploads/2026-09-04-gcp-docker/03-docker-login.webp)
*일회성 디바이스 코드를 브라우저에 입력해 인증하는 docker login 웹 로그인 흐름.*

CLI에 아이디/비밀번호를 직접 치는 대신, 코드를 브라우저에 입력하고 웹에서 인증을 완료하는 디바이스 코드 방식이라 터미널에 자격 증명이 그대로 노출되지 않는다.

## 4. 첫 번째 삽질 — permission denied

로그인까지는 문제없었는데, `docker pull nginx` 를 실행하자마자 소켓 권한 에러가 났다.

```bash
$ docker pull nginx
Using default tag: latest
permission denied while trying to connect to the Docker API at unix:///var/run/docker.sock
```

![docker pull nginx 실행 시 permission denied 에러](/images/uploads/2026-09-04-gcp-docker/04-permission-denied.webp)
*`docker pull nginx` 실행 직후 소켓 접근이 거부되는 에러 메시지.*

> **원인**: `/var/run/docker.sock` 은 `root` 와 `docker` 그룹만 접근할 수 있다. 로그인 계정이 `docker` 그룹에 추가되어 있어도, 그 변경은 **새 로그인 셸의 그룹 목록에만 반영**되기 때문에 이미 열려 있던 SSH 세션에서는 여전히 권한이 없다.

## 5. newgrp으로 그룹 즉시 반영

세션을 새로 열지 않고도 `newgrp docker` 로 현재 셸에 그룹 변경을 즉시 적용할 수 있다.

```bash
$ newgrp docker
$ groups
docker adm dialout cdrom floppy audio dip video plugdev netdev lxd ubuntu google-sudoers <내계정>
```

![newgrp docker 실행 후 groups 출력에 docker가 포함됨](/images/uploads/2026-09-04-gcp-docker/05-newgrp-groups.webp)
*`newgrp docker` 실행 후 `groups` 출력 맨 앞에 `docker` 가 잡힌다.*

`groups` 출력 맨 앞에 `docker` 가 잡히는 걸 확인했다. 이 다음부터는 `sudo` 없이도 `docker` 명령이 정상적으로 통과했다.

## 6. nginx 이미지 Pull

```bash
$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
f340c1b7c1d6: Pull complete
956faab5efb3: Pull complete
a44b5c8be616: Pull complete
02fc02c4ab8d: Pull complete
c12f394dea35: Pull complete
6310eb16bf42: Pull complete
07db7bf2649b: Pull complete
76f27c02d218: Download complete
25202a7045eb: Download complete
Digest: sha256:05b8cb60c354a44ab824ea6e7dc69b46d50762cdbe728a347a5b656e6fb3d7c4
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
```

![docker pull nginx 레이어 다운로드 로그](/images/uploads/2026-09-04-gcp-docker/06-pull.webp)
*레이어별 Pull complete 로그가 순서대로 찍힌다.*

이어서 `docker images` 로 로컬에 받아진 이미지를 확인했다.

| IMAGE | IMAGE ID | DISK USAGE | CONTENT SIZE |
|---|---|---|---|
| `nginx:latest` | `05b8cb60c354` | 253MB | 69.2MB |

![docker images 출력, nginx:latest 이미지 확인](/images/uploads/2026-09-04-gcp-docker/06b-images.webp)
*`docker images` 출력. nginx:latest 이미지가 253MB로 잡혀 있다.*

## 7. 컨테이너 실행

호스트의 80번 포트를 컨테이너의 80번 포트로 그대로 매핑해서 백그라운드로 띄웠다.

```bash
$ docker run -d -p 80:80 --name mynginx nginx
91e43754edad81672cd2f63f27f0de579237a51884b550a70245ca10cc6fe75a
```

![docker run 명령 실행 후 컨테이너 ID 출력](/images/uploads/2026-09-04-gcp-docker/07a-run.webp)
*`docker run` 직후 출력되는 컨테이너 ID.*

```bash
$ docker ps
CONTAINER ID   IMAGE   COMMAND                  CREATED         STATUS          PORTS                                 NAMES
91e43754edad   nginx   "/docker-entrypoint...."  25 seconds ago  Up 24 seconds   0.0.0.0:80->80/tcp, [::]:80->80/tcp   mynginx
```

![docker ps로 확인한 Up 상태의 mynginx 컨테이너](/images/uploads/2026-09-04-gcp-docker/07b-ps.webp)
*`docker ps` 로 Up 상태와 포트 매핑을 확인.*

여기까지는 로컬 관점에서 끝났다. VM 안에서는 80번 포트가 열려 있지만, 외부 브라우저에서 이 IP로 접속하려면 별도로 GCP 방화벽을 통과시켜야 한다는 걸 이 다음 단계에서 알게 됐다.

## 8. 방화벽에서 80번 포트 열기

GCP의 VPC 방화벽은 기본적으로 **인그레스(수신) 트래픽을 전부 차단**한다. 기본으로 열려 있는 규칙은 SSH(22), RDP(3389), 내부망, ICMP뿐이라 HTTP(80)는 직접 열어줘야 한다.

인스턴스 편집 화면의 **네트워킹 → 방화벽** 섹션에서 `Allow HTTP traffic` 체크박스를 켜면, GCP가 `http-server` 네트워크 태그를 인스턴스에 자동으로 붙여준다.

![VM 편집 화면의 Allow HTTP traffic 체크박스와 http-server 네트워크 태그](/images/uploads/2026-09-04-gcp-docker/08a-firewall-checkbox.webp)
*VM 편집 화면의 Networking → Firewalls. Allow HTTP traffic 체크와 http-server 네트워크 태그.*

다만 체크박스만으로는 태그가 붙을 뿐, 그 태그를 대상으로 한 방화벽 규칙 자체가 없으면 여전히 막혀 있다. 실제로 **Network Security → Firewall policies** 목록을 보면 기본 규칙은 ICMP·내부망·RDP·SSH뿐, HTTP는 없다.

![Firewall policies 목록, HTTP 규칙 생성 전 기본 규칙만 존재](/images/uploads/2026-09-04-gcp-docker/08b-firewall-list.webp)
*규칙 생성 전 Firewall policies 목록. 기본 규칙에 HTTP(80)는 없다.*

그래서 `Create firewall rule` 로 규칙을 직접 만들었다.

![Create a firewall rule 폼, 이름을 default-allow-http로 설정](/images/uploads/2026-09-04-gcp-docker/08c-firewall-form-top.webp)
*규칙 이름을 default-allow-http로 지정하고 Ingress / Allow로 설정.*

| 항목 | 값 |
|---|---|
| Name | `default-allow-http` |
| Target tags | `http-server` |
| Source IPv4 ranges | `0.0.0.0/0` |
| Protocols / ports | `tcp:80` |
| Action | `Allow` |

![방화벽 규칙 폼에 Target tags http-server, Source 0.0.0.0/0, TCP 80이 채워진 화면](/images/uploads/2026-09-04-gcp-docker/08d-firewall-form-filled.webp)
*Target tags(http-server), Source IPv4 ranges(0.0.0.0/0), TCP 포트 80까지 채운 최종 폼.*

> **참고**: Source를 `0.0.0.0/0` 으로 두면 인터넷 전체에서 80번 포트로 접근이 가능해진다. 테스트용 데모라면 괜찮지만, 실제 서비스라면 필요한 대역만 좁혀서 허용하는 편이 안전하다.

## 9. 외부 접속 확인

VM 인스턴스 목록에서 External IP를 확인한 뒤, 그 주소로 바로 접속해봤다.

![VM instances 목록에서 External IP 컬럼 확인](/images/uploads/2026-09-04-gcp-docker/09a-external-ip.webp)
*VM instances 목록의 External IP 컬럼.*

![브라우저에서 외부 IP로 접속했을 때 뜨는 nginx 웰컴 페이지](/images/uploads/2026-09-04-gcp-docker/09b-nginx-welcome.webp)
*외부 IP로 접속하면 뜨는 Welcome to nginx! 기본 페이지.*

방화벽 규칙이 반영되기까지 잠깐의 지연이 있을 수 있지만, 대체로 몇 초 안에 `Welcome to nginx!` 페이지가 떴다. 이 시점부터는 이 IP:80으로 컨테이너 안의 nginx가 실제로 인터넷에 노출된 상태다.

## 10. 트러블슈팅 정리

이번 작업에서 막혔던 지점은 정확히 두 곳이었다. 다음에 같은 삽질을 반복하지 않으려고 원인과 해결을 표로 남긴다.

| 증상 | 원인 | 해결 |
|---|---|---|
| `docker pull` 시 `permission denied` | 계정을 `docker` 그룹에 추가했지만 현재 로그인 셸에는 반영 안 됨 | `newgrp docker` 실행 (또는 재로그인) |
| 컨테이너는 `Up` 상태인데 외부 브라우저에서 접속 안 됨 | GCP VPC 기본 방화벽에 HTTP(80) 인그레스 규칙이 없음 | 인스턴스에 `http-server` 태그 부여 + tcp:80 허용 규칙 생성 |

결과적으로 걸린 시간의 대부분은 *Docker 자체*가 아니라 *권한과 네트워크 경계*에서 소모됐다. 클라우드 VM에 뭔가를 처음 올릴 때는 애플리케이션 로직보다 이 두 가지 — 리눅스 그룹 권한, 클라우드 방화벽 — 를 먼저 의심해보는 게 맞는 것 같다.
